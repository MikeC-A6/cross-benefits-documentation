# Disability Benefit Decisions Kafka Architecture

## Overview

This document explains the integration and data flow between three key repositories responsible for handling disability benefit decisions through Kafka events, specifically for Veterans Pension, Supplemental Claims, and Higher Level Review decisions.

## Repository Relationships

The three repositories work together in a sequential pipeline to process claim lifecycle events and notify veterans about decision letter availability:

1. **bip-bie-claim-lifecycle-sp** → Produces initial claim lifecycle events
2. **event-bus-decision-letter-filter-producer** → Filters and transforms events
3. **eventbus-gateway** → Consumes filtered events and triggers notifications

## Repository Details

### 1. bip-bie-claim-lifecycle-sp
**Repository**: https://github.com/department-of-veterans-affairs/bip-bie-claim-lifecycle-sp

**Purpose**: BIP Benefits Integration Events (BIE) Claim Lifecycle Stream Processor

**Technology**: Java/Maven Spring Boot application with Kafka Streams

**Role**: This is the **event producer** in the pipeline. It:
- Evaluates claim lifecycle data from VA systems
- Generates `ClaimLifeCycleUpdatedEvent` events
- Publishes events to Kafka topics as part of the BIE catalog of events
- Acts as the source of truth for claim status changes

**Key Events Generated**:
- `ClaimLifeCycleUpdatedEvent` - documented at https://community.max.gov/display/VAExternal/ClaimLifecycleStatusUpdatedEvent

### 2. event-bus-decision-letter-filter-producer
**Repository**: https://github.com/department-of-veterans-affairs/event-bus-decision-letter-filter-producer

**Purpose**: Decision Letter Filter and Producer

**Technology**: Java/Maven Spring Boot application with Kafka Streams

**Role**: This is the **event filter/transformer** in the pipeline. It:
- Consumes deduplicated claim lifecycle status events from the BIP system
- Applies filtering logic to determine which events represent decision letter availability
- Produces filtered events to the `decision_letter_availability` topic

**Kafka Topics**:
- **Input**: `deduplicated_claim_lifecycle_status_events`
- **Output**: `decision_letter_availability`

**Filtering Criteria** (all must be met):
1. **Claim Status**: `ClaimLifecycleStatus` must be either "Authorized" or "Continued at Authorization"
2. **Claim Type**: `ClaimTypeCode` must start with specific two-digit prefixes (010, 110, 020, etc.)
3. **Participant Match**: `ClaimantParticipantId` and `VeteranParticipantId` must match (ensuring the veteran filed their own claim)

### 3. eventbus-gateway
**Repository**: https://github.com/department-of-veterans-affairs/eventbus-gateway

**Purpose**: Event Bus Gateway for Downstream Integrations

**Technology**: Ruby on Rails application with Karafka (Ruby Kafka library)

**Role**: This is the **event consumer** in the pipeline. It:
- Consumes events from the `decision_letter_availability` topic
- Processes decision letter availability notifications
- Triggers email notifications to veterans through the VA API
- Integrates with Sign-In Service for authentication

**Kafka Topics**:
- **Input**: `decision_letter_availability`

**Consumer**: `DecisionLetteryAvailabilityConsumer`
- Deserializes Avro messages using Schema Registry
- Filters for relevant claim codes (010, 110, 020)
- Sends email notifications via vets-api email endpoint
- Uses OAuth token authentication

## Data Flow Architecture

```
[VA Systems] 
    ↓ (claim lifecycle changes)
[bip-bie-claim-lifecycle-sp]
    ↓ (ClaimLifeCycleUpdatedEvent)
[Kafka: deduplicated_claim_lifecycle_status_events]
    ↓
[event-bus-decision-letter-filter-producer]
    ↓ (applies filtering logic)
[Kafka: decision_letter_availability]
    ↓
[eventbus-gateway]
    ↓ (DecisionLetterAvailabilityConsumer)
[Email Notification to Veteran]
```

## Event Schema and Format

### ClaimLifeCycleUpdatedEvent Fields
Based on the test data and filtering logic, key fields include:
- `ClaimLifecycleStatus`: Status of the claim (e.g., "Authorized")
- `ClaimTypeCode`: Type of claim (e.g., "010", "020", "110")
- `ClaimId`: Unique identifier for the claim
- `VeteranParticipantId`: Veteran's participant ID
- `ClaimantParticipantId`: Claimant's participant ID
- `DateClosed`: When the claim was closed
- `DateUpdated`: When the claim was last updated

### Decision Letter Availability Event
The filtered events represent that a decision letter is now available for download by the veteran.

## Infrastructure Components

### Kafka Infrastructure
- **Event Bus Cluster**: Managed Kafka cluster for event streaming
- **Schema Registry**: Manages Avro schemas for event serialization/deserialization
- **Topics**: 
  - `deduplicated_claim_lifecycle_status_events`
  - `decision_letter_availability`

### Authentication & Security
- **OAuth/SASL**: Used for Kafka authentication
- **Sign-In Service**: Provides access tokens for API calls
- **Avro Schema Registry**: Ensures data contract compliance

## Configuration and Deployment

### Environment Configuration
- **Development**: Uses local Docker setup with docker-compose
- **Staging/Production**: Deployed via GitHub Actions to AWS EKS
- **Configuration**: Managed through [bip-enterprise-integration-events-config](https://github.com/department-of-veterans-affairs/bip-enterprise-integration-events-config)

### Key Environment Variables
- `BOOTSTRAP_SERVERS`: Kafka broker endpoints
- `EMAIL_TEMPLATE_ID`: Template for notification emails
- `API_URL`: VA API endpoint for email sending

## Monitoring and Observability

### Logging
- **Kafka Events**: Logged to `karafka.log` in eventbus-gateway
- **DataDog Integration**: Available for metrics and tracing
- **Error Tracking**: Configured for both consumer and producer errors

### Health Checks
- Application health endpoints
- Kafka consumer lag monitoring
- Schema Registry connectivity

## Testing

### Integration Testing
- **Docker Compose**: Local development environment
- **Testcontainers**: Used for integration tests
- **Manual Testing**: Via Kafka console producers/consumers

### Test Data Examples
The README includes sample events for positive and negative test cases demonstrating the filtering logic.

## Adding New Claim Types

To extend support for additional disability benefit decisions:

1. **Update Filter Logic**: Modify the claim type code filters in `event-bus-decision-letter-filter-producer`
2. **Update Consumer Logic**: Modify `RELEVANT_CLAIM_CODES` in `DecisionLetterAvailabilityConsumer`
3. **Schema Updates**: Ensure Avro schemas support any new event fields
4. **Testing**: Add test cases for new claim types

## Conclusion

This architecture provides a scalable, event-driven approach to notifying veterans about decision letter availability. The pipeline ensures that only relevant, authorized decisions trigger notifications while maintaining data integrity through schema validation and filtering logic.

The separation of concerns allows each component to be independently developed, tested, and deployed while maintaining a cohesive end-to-end workflow for veteran notifications. 
