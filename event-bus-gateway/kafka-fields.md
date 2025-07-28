# Event Bus Decision Letter Filter Producer - Kafka Fields Documentation

## Overview

This document outlines the fields that are posted to Kafka by the [Event Bus Decision Letter Filter Producer](https://github.com/department-of-veterans-affairs/event-bus-decision-letter-filter-producer) application. This application filters deduplicated claim lifecycle status events from the Benefits Integration Platform (BIP) and publishes decision letter availability notifications.

## Application Architecture

### Data Flow
```
Input Topic: deduplicated_claim_lifecycle_status_events
     ↓
   [Filters Applied]
     ↓
Output Topic: decision_letter_availability
```

### Key Components
- **Entry Point**: `src/main/java/com/eventbus/decisionletter/App.java`
- **Configuration**: `src/main/java/com/eventbus/decisionletter/config/DependencyProvider.java`
- **Filtering Logic**: `src/main/java/com/eventbus/decisionletter/filter/` package

## Kafka Topics

| Topic | Type | Description |
|-------|------|-------------|
| `deduplicated_claim_lifecycle_status_events` | Input | Raw claim lifecycle events from BIP |
| `decision_letter_availability` | Output | Filtered events representing decision letter availability |

## Data Structure

The application processes `DecisionLetterAvailabilityRecord` objects with the following fields:

### Core Claim Information
| Field | Type | Description |
|-------|------|-------------|
| `ClaimId` | Long | Unique claim identifier |
| `ClaimTypeCode` | String | Code indicating the type of claim (filtered by regex) |
| `ClaimLifecycleStatus` | String | Current status of the claim lifecycle |
| `ClaimLifecycleStatusId` | Integer | Numeric ID for the lifecycle status |

### Participant Information
| Field | Type | Description |
|-------|------|-------------|
| `ClaimantParticipantId` | Long | ID of the person filing the claim |
| `VeteranParticipantId` | Long | ID of the veteran (must match claimant for filtering) |
| `StatusChangedByParticipantId` | Long | ID of participant who changed the status |

### Case and Assignment Details
| Field | Type | Description |
|-------|------|-------------|
| `CaseId` | Long | Associated case identifier |
| `CaseAssignmentLocationId` | Long | Location where case is assigned |
| `CaseAssignmentStatusNumber` | Integer | Status number for case assignment |

### Jurisdictional Information
| Field | Type | Description |
|-------|------|-------------|
| `ClaimJurisdictionLocationId` | Long | Jurisdiction location for the claim |
| `ProgramTypeCode` | String | Program type associated with the claim |

### Status and Reason Information
| Field | Type | Description |
|-------|------|-------------|
| `LifecycleStatusReasonTypeCode` | String | Code for the reason behind status change |
| `ReasonDetailTypeCode` | String | Detailed reason code |
| `ReasonText` | String | Human-readable reason text |

### Actor Information
| Field | Type | Description |
|-------|------|-------------|
| `ActorApplicationId` | String | Application that triggered the event |
| `ActorStation` | String | Station identifier |
| `ActorUserId` | String | User who performed the action |

### Timestamps
| Field | Type | Description |
|-------|------|-------------|
| `DateClosed` | Long | Timestamp when claim was closed |
| `DateUpdated` | Long | Timestamp of last update |
| `bie_ts` | Long | BIE (Benefits Integration Environment) timestamp |
| `source_ts` | Long | Source system timestamp |
| `connector_ts` | Long | Connector timestamp |

## Filtering Logic

The application applies three main filters that control which events are posted to Kafka:

### 1. Claim Status Filter
**File**: `src/main/java/com/eventbus/decisionletter/filter/ClaimStatusRecordFilter.java`

**Logic**: Only processes events with specific claim lifecycle statuses:
- `"Authorized"`
- `"Continued at Authorization"`

**Code Reference**:
```java
private static final Set<String> ALLOWED_CLAIM_STATUSES = 
    Set.of("Authorized", "Continued at Authorization");
```

### 2. Claim Type Filter
**File**: `src/main/java/com/eventbus/decisionletter/filter/ClaimTypeRecordFilter.java`

**Logic**: Only processes events where `ClaimTypeCode` matches specific patterns:

**Allowed Prefixes**:
- `01*` - Initial comp with more than 8 issues
- `02*` - Reopened comp or claim for increase  
- `03*` - Higher Level Review (HLR)
- `04*` - Supplemental claim
- `07*` - Board of Veterans Appeals (BVA) claims
- `11*` - Initial comp with less than 8 issues
- `12*` - Reopened pension claim
- `17*` - Appeal Action
- `18*` - Initial pension claim
- `40*` - Non-claim correspondence that results in a notification letter

**Regex Pattern**:
```java
public static final String ALLOWED_CLAIM_TYPES_REGEX = 
    "(01|02|03|04|07|11|12|17|18|40)[0-9][A-Za-z0-9]*";
```

### 3. Participant ID Filter
**File**: `src/main/java/com/eventbus/decisionletter/filter/ParticipantIdRecordFilter.java`

**Logic**: Only processes events where:
- Both `ClaimantParticipantId` and `VeteranParticipantId` are non-null
- `ClaimantParticipantId` equals `VeteranParticipantId` (veteran filed their own claim)

**Code Reference**:
```java
return Objects.nonNull(claimant) && Objects.nonNull(veteran) && claimant.equals(veteran);
```

### Composite Filter
**File**: `src/main/java/com/eventbus/decisionletter/filter/CompositeRecordFilter.java`

All three filters must pass for an event to be published to the output topic.

## Configuration Files

### Environment Variables
**File**: `src/main/java/com/eventbus/decisionletter/config/DependencyProvider.java`

| Variable | Default | Description |
|----------|---------|-------------|
| `APPLICATION_ID_CONFIG` | `"dla-filter"` | Kafka Streams application ID |
| `EB_SOURCE_TOPIC` | `"deduplicated_claim_lifecycle_status_events"` | Input topic |
| `EB_TARGET_TOPIC` | `"decision_letter_availability"` | Output topic |
| `EB_BOOTSTRAP_SERVERS` | `"localhost:9092"` | Kafka broker addresses |
| `EB_SCHEMA_REGISTRY_URL` | `"http://localhost:8081"` | Schema registry URL |
| `EB_KEY_SCHEMA_ID` | (required) | Schema ID for message keys |
| `EB_VALUE_SCHEMA_ID` | (required) | Schema ID for message values |
| `PROCESSING_GUARANTEE` | `"at_least_once"` | Kafka Streams processing guarantee |

### Schema Configuration
**File**: `pom.xml`

The application uses Avro schemas from the VA schemas repository:
- **Dependency**: `gov.va.eventbus:va-schemas`
- **Key Schema**: Primitive Long (Claim ID)
- **Value Schema**: `DecisionLetterAvailabilityRecord`

## Example Event Data

Based on test cases in `src/test/resources/PositiveTestCase1.json`:

```json
{
    "ActorApplicationId": {"string": "PositiveTestCase1"},
    "ActorStation": {"string": "a String"},
    "ActorUserId": {"string": "ACTOR-1000000587113"},
    "bie_ts": {"long": 1686330025031},
    "CaseAssignmentLocationId": {"long": 29},
    "CaseAssignmentStatusNumber": {"int": 22},
    "CaseId": {"long": 2992380},
    "ClaimantParticipantId": {"long": 123},
    "ClaimId": {"long": 1},
    "ClaimLifecycleStatus": "Authorized",
    "ClaimLifecycleStatusId": 29,
    "ClaimTypeCode": {"string": "010"},
    "ClaimJurisdictionLocationId": {"long": 29},
    "connector_ts": {"long": 1686330025031},
    "DateClosed": {"long": 1686330025031},
    "DateUpdated": {"long": 1686330025031},
    "LifecycleStatusReasonTypeCode": {"string": "a String"},
    "ProgramTypeCode": "a String",
    "ReasonDetailTypeCode": {"string": "a String"},
    "ReasonText": {"string": "a String"},
    "source_ts": {"long": 1686330025031},
    "StatusChangedByParticipantId": {"long": 29},
    "VeteranParticipantId": {"long": 123}
}
```

## Related Resources

- **Use Case Documentation**: [Decision Letter Availability Use Case](https://github.com/department-of-veterans-affairs/VES/blob/master/research/Event%20Bus/Use%20Cases/Benefits/Claims/Decision%20Letter%20Availability/Decision%20Letter%20Availability%20Use%20Case.md)
- **Architecture Diagrams**: [Event Bus Notify Architecture](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/event-bus-notify)
- **Eventbus Gateway**: [eventbus-gateway repo](https://github.com/department-of-veterans-affairs/eventbus-gateway)
- **Readiness Review**: [Checklist](https://github.com/department-of-veterans-affairs/va.gov-team-sensitive/issues/4076)

## Code Validation Results

**IMPORTANT**: After analyzing the actual Kafka Streams topology code, this application is a **pure filter** - it does NOT transform or select specific fields.

### Actual Data Flow
```java
// From DependencyProvider.java - Line ~460
builder.stream(inputTopic, Consumed.with(keySerde, valueSerde))
    .peek((k, v) -> LOG.debug("Observed event: {}", v))
    .filter(recordFilter)  // Only filters records, doesn't transform fields
    .peek((k, v) -> LOG.debug("Send event: {}", v))
    .to(outputTopic, Produced.with(keySerde, valueSerde));  // Same record structure
```

### What Actually Gets Posted to Kafka

**ALL FIELDS** from the input `DecisionLetterAvailabilityRecord` are posted to Kafka **unchanged** if the record passes the filters.

- **Key**: `ClaimId` (Long)
- **Value**: Complete `DecisionLetterAvailabilityRecord` with all ~22 fields intact
- **No field transformation**: Input record = Output record (when filters pass)

The filtering logic controls **WHICH** records get published, not **WHICH** fields are included.

## Summary

The Event Bus Decision Letter Filter Producer acts as a **selective passthrough filter**, ensuring that only relevant claim lifecycle events—those representing actual decision letter availability for veterans who filed their own claims—are published to the `decision_letter_availability` topic. The complete input record structure is preserved in the output, with filtering criteria ensuring high data quality and relevance for downstream consumers. 
