# VA Claims Status Tool API Architecture

## Overview

This document traces the complete API call flow from the VA.gov Claims Status Tool frontend to the various backend services and external APIs. The Claims Status Tool allows veterans to check the status of their VA claims, appeals, and related documents online.

## High-Level Architecture

```
┌─────────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   Claims Status     │────│    vets-api      │────│   External APIs     │
│   Frontend (React)  │    │   (Rails API)    │    │   (Lighthouse,      │
│                     │    │                  │    │    Caseflow, etc.)  │
└─────────────────────┘    └──────────────────┘    └─────────────────────┘
```

## Frontend (vets-website)

**Location**: [`https://github.com/department-of-veterans-affairs/vets-website/tree/main/src/applications/claims-status`](https://github.com/department-of-veterans-affairs/vets-website/tree/main/src/applications/claims-status)

### Key Frontend Components

#### 1. Actions (`actions/index.js`)

The frontend makes the following API calls:

##### **Benefits Claims APIs**
- **GET `/benefits_claims`** - Fetch list of all claims
- **GET `/benefits_claims/{id}`** - Fetch specific claim details
- **POST `/benefits_claims/{id}/submit5103`** - Submit 5103 evidence waiver

##### **Appeals APIs** 
- **GET `/appeals`** - Fetch appeals data from Caseflow

##### **STEM Claims APIs**
- **GET `/education_benefits_claims/stem_claim_status`** - Fetch STEM claim status

##### **Document Upload APIs**
- **POST `/benefits_claims/{id}/benefits_documents`** - Upload documents to specific claim

#### 2. Constants (`constants.js`)

Defines claim type codes for different benefit types:
- **Disability compensation claims**: EP010, EP020, EP110, etc.
- **Pension claims**: EP120 (reopened), EP180 (initial), EP190, etc.
- **Dependency claims**: EP130 variants for adding/removing dependents
- **Survivors pension**: EP190 variants

## Backend API (vets-api)

**Location**: [`https://github.com/department-of-veterans-affairs/vets-api`](https://github.com/department-of-veterans-affairs/vets-api)

### 1. Benefits Claims Controller

**File**: `app/controllers/v0/benefits_claims_controller.rb`

#### **GET `/v0/benefits_claims` (index)**
- **Service**: `BenefitsClaimService.new(@current_user.icn).get_claims`
- **Purpose**: Retrieves all claims for the authenticated user
- **Response Processing**:
  - Updates claim type language for user-friendly display
  - Adds failed upload indicators if feature flag enabled
  - Logs claim analytics to DataDog
  - Tracks claims in EVSS database

#### **GET `/v0/benefits_claims/{id}` (show)**
- **Service**: `BenefitsClaimService.new(@current_user.icn).get_claim(id)`
- **Purpose**: Retrieves detailed information for a specific claim
- **Response Processing**:
  - Manual status overrides for certain tracked items
  - Suppresses evidence requests based on feature flags
  - Adds document upload capabilities based on BIRLS ID
  - Includes evidence submissions if feature flag enabled

#### **POST `/v0/benefits_claims/{id}/submit5103`**
- **Service**: `BenefitsClaimService.new(@current_user.icn).submit5103(id, tracked_item_id)`
- **Purpose**: Submits 5103 evidence waiver for a claim
- **Payload**: `{ "trackedItemId": number }`

### 2. Appeals Controller

**File**: `app/controllers/v0/appeals_controller.rb`

#### **GET `/v0/appeals` (index)**
- **Service**: `appeals_service.get_appeals(current_user)` 
- **Purpose**: Retrieves appeals data from Caseflow
- **Inherits**: From `AppealsBaseController`

### 3. Service Layer

#### Benefits Claims Service
**Location**: `lib/lighthouse/benefits_claims/service.rb`
- **Purpose**: Interfaces with Lighthouse Benefits Claims API
- **Authentication**: Uses user's ICN (Integration Control Number)
- **Features**:
  - Claim retrieval and status checking
  - Document upload coordination
  - 5103 evidence waiver submission

#### Caseflow Service  
**Location**: `lib/caseflow/service.rb`
- **Purpose**: Interfaces with Caseflow for appeals data
- **API Endpoints**:
  - `/api/v2/appeals` - Main appeals endpoint
  - `/api/v3/decision_reviews` - Decision reviews
  - `/api/v3/decision_reviews/legacy_appeals` - Legacy appeals
- **Authentication**: Token-based using `Settings.caseflow.app_token`

## External API Dependencies

### 1. Lighthouse Benefits Claims API
- **Purpose**: Modern VA benefits claims system
- **Data**: Current claims status, tracked items, documents
- **Authentication**: ICN-based

### 2. Caseflow API
- **Base URL**: Configured via `Settings.caseflow.host`
- **Endpoints**:
  - **V2 Appeals**: `/api/v2/appeals`
  - **V3 Decision Reviews**: `/api/v3/decision_reviews`
  - **Legacy Appeals**: `/api/v3/decision_reviews/legacy_appeals`
- **Authentication**: Bearer token authentication

### 3. Legacy Appeals API (v0)
**URL**: [`https://developer.va.gov/explore/api/legacy-appeals/docs?version=current`](https://developer.va.gov/explore/api/legacy-appeals/docs?version=current)
- **Purpose**: Legacy appeals system integration
- **Status**: Referenced but appears to be transitioning to Caseflow

### 4. BGS (Benefits Gateway Service)
- **Purpose**: Veteran file number lookup
- **Integration**: `BGS::People::Request.new.find_person_by_participant_id`

### 5. EVSS (Enterprise Veteran Self Service)
- **Purpose**: Legacy claims system (being phased out)
- **Database**: Local EVSS claims tracking table
- **Status**: Claims are being migrated to Lighthouse

## Complete API Flow

### 1. Get All Claims Flow
```
Frontend → GET /v0/benefits_claims 
         → BenefitsClaimsController#index
         → BenefitsClaimsService.get_claims(icn)
         → Lighthouse Benefits Claims API
         → Process & enhance response
         → Return to frontend
```

### 2. Get Specific Claim Flow  
```
Frontend → GET /v0/benefits_claims/{id}
         → BenefitsClaimsController#show  
         → BenefitsClaimsService.get_claim(id, icn)
         → Lighthouse Benefits Claims API
         → Apply business logic & feature flags
         → Return enhanced claim data
```

### 3. Get Appeals Flow
```
Frontend → GET /v0/appeals
         → AppealsController#index
         → CaseflowService.get_appeals(user)
         → Caseflow API /api/v2/appeals
         → Return appeals data
```

### 4. Submit Evidence Waiver Flow
```
Frontend → POST /v0/benefits_claims/{id}/submit5103
         → BenefitsClaimsController#submit5103
         → BenefitsClaimsService.submit5103(id, tracked_item_id)
         → Lighthouse Benefits Claims API
         → Return submission confirmation
```

## Key Data Flow Features

### Authentication & Authorization
- **User Authentication**: Session-based via VA.gov login
- **API Authorization**: ICN (Integration Control Number) used for Lighthouse
- **Service Authentication**: Bearer tokens for external services

### Claim Types & EP Codes
The system handles various claim types identified by EP (End Product) codes:
- **010**: Initial disability compensation claims  
- **020**: Reopened disability compensation claims
- **110**: Appeals-related claims
- **120**: Reopened pension claims  
- **180**: Initial pension claims
- **130**: Dependency-related claims

### Document Management
- **Upload Endpoint**: `/v0/benefits_claims/{id}/benefits_documents`
- **Evidence Tracking**: Links documents to specific tracked items
- **Status Monitoring**: Tracks upload success/failure states
- **Feature Flags**: `cst_show_document_upload_status` controls visibility

### Feature Flags & Business Logic
- **RV1 Override**: `cst_override_reserve_records_website` - Renames Reserve Records Request status
- **Evidence Suppression**: `cst_suppress_evidence_requests_website` - Hides certain evidence requests  
- **Document Status**: `cst_show_document_upload_status` - Shows upload status information

## Monitoring & Analytics

### DataDog Integration
The system logs comprehensive analytics including:
- **Claim Types**: EP codes and claim categories
- **API Performance**: Request latency and success rates
- **User Activity**: Claim access patterns
- **Evidence Requests**: Types and frequencies

### Error Handling
- **Service Timeouts**: 20-second default timeout for external APIs
- **Circuit Breakers**: Prevents cascade failures using Faraday middleware
- **Monitoring**: Centralized logging via Rails logger and external monitoring

## Security & Data Protection
- **Authentication**: Multi-factor authentication via VA.gov
- **Authorization**: ICN-based access control
- **Data Encryption**: HTTPS for all API communications  
- **PII Protection**: Veterans' personal information handled per VA security standards

## Migration Strategy
The system is actively migrating from legacy EVSS to modern Lighthouse APIs:
- **Claims Data**: Moving from EVSS to Lighthouse Benefits Claims
- **Document Uploads**: Transitioning to Lighthouse Benefits Documents
- **Appeals**: Consolidating through Caseflow API
- **Dual Operations**: Currently supports both systems during transition

This architecture provides veterans with a unified interface to check their claims and appeals status while the VA modernizes its underlying systems.
