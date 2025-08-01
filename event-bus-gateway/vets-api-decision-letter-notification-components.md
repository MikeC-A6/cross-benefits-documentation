# Vets-API Decision Letter Notification Components

## Overview

This document details all the components within the `vets-api` repository that participate in the decision letter email notification process. The notification flow is triggered by the EventBus Gateway (external service) and processed through multiple interconnected components in vets-api.

## Architecture Flow

```
EventBus Gateway → EventBusGatewayController → LetterReadyEmailJob → VA Notify → VANotifyEmailStatusCallback
                                          ↓
                                    BGS Services (for user lookup)
```

## Core Components

### 1. EventBusGatewayController
**File**: `app/controllers/v0/event_bus_gateway_controller.rb`

```ruby
# frozen_string_literal: true

module V0
  class EventBusGatewayController < SignIn::ServiceAccountApplicationController
    service_tag 'event_bus_gateway'

    def send_email
      if Flipper.enabled?(:event_bus_gateway_emails_enabled)
        EventBusGateway::LetterReadyEmailJob.perform_async(
          participant_id,
          send_email_params[:template_id]
        )
      end
      head :ok
    end

    private

    def participant_id
      @participant_id ||= @service_account_access_token.user_attributes['participant_id']
    end

    def send_email_params
      params.permit(:template_id)
    end
  end
end
```

**Key Responsibilities:**
- **Entry Point**: Receives HTTP requests from the EventBus Gateway service
- **Authentication**: Inherits from `ServiceAccountApplicationController` for service-to-service authentication
- **Feature Flag**: Checks `event_bus_gateway_emails_enabled` flipper flag before processing
- **Job Dispatch**: Extracts participant ID from service account token and queues email job
- **Parameter Handling**: Permits and validates the `template_id` parameter

**Critical Detail for Survivors Question:**
The `participant_id` is extracted directly from the service account access token's `user_attributes`. This participant ID comes from the original Kafka event, which for EP120 survivor pension claims would be the Veteran's participant ID (since the claim is filed under the Veteran's record).

### 2. LetterReadyEmailJob
**File**: `app/sidekiq/event_bus_gateway/letter_ready_email_job.rb`

```ruby
# frozen_string_literal: true

require 'sidekiq'

module EventBusGateway
  class LetterReadyEmailJob
    include Sidekiq::Job
    include SentryLogging

    sidekiq_options retry: 0
    NOTIFY_SETTINGS = Settings.vanotify.services.benefits_management_tools
    HOSTNAME_MAPPING = {
      'dev-api.va.gov' => 'dev.va.gov',
      'staging-api.va.gov' => 'staging.va.gov',
      'api.va.gov' => 'www.va.gov'
    }.freeze

    def perform(participant_id, template_id)
      notify_client.send_email(
        recipient_identifier: { id_value: participant_id, id_type: 'PID' },
        template_id:,
        personalisation: { host: HOSTNAME_MAPPING[Settings.hostname] || Settings.hostname,
                           first_name: get_first_name_from_participant_id(participant_id) }
      )
    rescue => e
      record_email_send_failure(e)
    end

    private

    def notify_client
      VaNotify::Service.new(NOTIFY_SETTINGS.api_key,
                            { callback_klass: 'EventBusGateway::VANotifyEmailStatusCallback' })
    end

    def get_first_name_from_participant_id(participant_id)
      bgs = BGS::Services.new(external_uid: participant_id, external_key: participant_id)
      person = bgs.people.find_person_by_ptcpnt_id(participant_id)
      if person
        person[:first_nm].capitalize
      else
        raise StandardError, 'Participant ID cannot be found in BGS'
      end
    end

    def record_email_send_failure(error)
      error_message = 'LetterReadyEmailJob errored'
      ::Rails.logger.error(error_message, { message: error.message })
      StatsD.increment('event_bus_gateway', tags: ['service:event-bus-gateway', "function: #{error_message}"])
    end
  end
end
```

**Key Responsibilities:**
- **Email Orchestration**: Coordinates the email sending process through VA Notify
- **BGS Integration**: Looks up person details using the participant ID to get first name
- **Personalization**: Builds personalized email content with recipient's first name and appropriate hostname
- **Error Handling**: Logs failures and emits metrics for monitoring
- **No Retry**: Configured with `retry: 0` to prevent duplicate emails

**Critical Details for Survivors Question:**
1. **Participant ID Usage**: Uses the participant ID to identify the email recipient in VA Notify
2. **BGS Lookup**: Makes a BGS call using `find_person_by_ptcpnt_id(participant_id)` to get the person's first name
3. **Direct ID Mapping**: The participant ID passed to VA Notify is the same one from the original event

### 3. ServiceAccountApplicationController
**File**: `app/controllers/sign_in/service_account_application_controller.rb`

```ruby
# frozen_string_literal: true

module SignIn
  class ServiceAccountApplicationController < ActionController::API
    include SignIn::Authentication
    include SignIn::ServiceAccountAuthentication
    include Pundit::Authorization
    include ExceptionHandling
    include Headers
    include ControllerLoggingContext
    include SentryLogging
    include SentryControllerLogging
    include Traceable

    before_action :authenticate_service_account
    skip_before_action :authenticate

    private

    attr_reader :current_user
  end
end
```

**Key Responsibilities:**
- **Base Controller**: Provides service account authentication for EventBus Gateway
- **Security**: Implements service-to-service authentication via access tokens
- **Authorization**: Includes Pundit for authorization checks
- **Observability**: Includes logging and tracing capabilities

### 4. BGS People Service
**File**: `app/services/bgs/people/service.rb`

```ruby
# frozen_string_literal: true

module BGS
  module People
    class Service
      class VAFileNumberNotFound < StandardError; end

      include SentryLogging

      attr_reader :ssn, :participant_id, :common_name, :email, :icn

      def initialize(user)
        @ssn = user.ssn
        @participant_id = user.participant_id
        @common_name = user.common_name
        @email = user.email
        @icn = user.icn
      end

      def find_person_by_participant_id
        raw_response = service.people.find_person_by_ptcpnt_id(participant_id, ssn)
        if raw_response.blank?
          log_exception_to_sentry(VAFileNumberNotFound.new, { icn: }, { team: Constants::SENTRY_REPORTING_TEAM })
        end
        BGS::People::Response.new(raw_response, status: :ok)
      rescue => e
        log_exception_to_sentry(e, { icn: }, { team: Constants::SENTRY_REPORTING_TEAM })
        BGS::People::Response.new(nil, status: :error)
      end

      private

      def service
        @service ||= BGS::Services.new(external_uid: icn, external_key:)
      end

      def external_key
        @external_key ||= begin
          key = common_name.presence || email
          key.first(Constants::EXTERNAL_KEY_MAX_LENGTH)
        end
      end
    end
  end
end
```

**Key Responsibilities:**
- **BGS Integration**: Provides interface to BGS (Benefits Gateway Services) for person lookups
- **Person Resolution**: Looks up person details by participant ID
- **Error Handling**: Logs exceptions to Sentry with appropriate team routing

**Note**: In the LetterReadyEmailJob, BGS is called directly via `BGS::Services.new()` rather than through this service class.

### 5. VANotifyEmailStatusCallback
**File**: `app/sidekiq/event_bus_gateway/va_notify_email_status_callback.rb`

```ruby
# frozen_string_literal: true

module EventBusGateway
  class VANotifyEmailStatusCallback
    def self.call(notification)
      status = notification.status

      add_metrics(status)
      add_log(notification) if notification.status != 'delivered'
    end

    def self.add_log(notification)
      context = {
        notification_id: notification.notification_id,
        source_location: notification.source_location,
        status: notification.status,
        status_reason: notification.status_reason,
        notification_type: notification.notification_type
      }

      Rails.logger.error(name, context)
    end

    def self.add_metrics(status)
      case status
      when 'delivered'
        StatsD.increment('api.vanotify.notifications.delivered')
        StatsD.increment('callbacks.event_bus_gateway.va_notify.notifications.delivered')
      when 'permanent-failure'
        StatsD.increment('api.vanotify.notifications.permanent_failure')
        StatsD.increment('callbacks.event_bus_gateway.va_notify.notifications.permanent_failure')
      when 'temporary-failure'
        StatsD.increment('api.vanotify.notifications.temporary_failure')
        StatsD.increment('callbacks.event_bus_gateway.va_notify.notifications.temporary_failure')
      else
        StatsD.increment('api.vanotify.notifications.other')
        StatsD.increment('callbacks.event_bus_gateway.va_notify.notifications.other')
      end
    end
  end
end
```

**Key Responsibilities:**
- **Status Tracking**: Receives callbacks from VA Notify about email delivery status
- **Metrics Collection**: Emits StatsD metrics for different delivery statuses
- **Error Logging**: Logs non-delivered notifications for debugging

## Data Flow Analysis

### Participant ID Resolution Chain

1. **Source**: Kafka event contains Veteran's participant ID (for EP120 survivor pension claims)
2. **EventBus Gateway**: Forwards participant ID in service account token
3. **EventBusGatewayController**: Extracts participant ID from token
4. **LetterReadyEmailJob**: Uses participant ID for:
   - BGS lookup to get first name
   - VA Notify recipient identification
5. **VA Notify**: Sends email to the profile associated with that participant ID

### Critical Finding for Survivors Question

The system operates on a **single participant ID principle**:

- For EP120 survivor pension claims, the participant ID in the Kafka event is the **Veteran's participant ID**
- This participant ID flows through the entire system unchanged
- BGS lookups use the Veteran's participant ID
- VA Notify attempts to send emails to the Veteran's participant ID
- Since the Veteran is deceased, this participant ID likely doesn't have an active VA.gov email profile

## Configuration and Dependencies

### Feature Flags
- `event_bus_gateway_emails_enabled`: Controls whether emails are sent

### External Dependencies
- **BGS (Benefits Gateway Services)**: Person lookup and validation
- **VA Notify**: Email delivery service
- **EventBus Gateway**: External service that triggers notifications

### Routing Configuration
The controller is likely mapped to an endpoint like `/v0/event_bus_gateway/send_email` based on Rails conventions.

## Security and Authentication

- Uses service account authentication via SignIn framework
- Requires valid service account access tokens
- Participant ID is extracted from authenticated token, preventing tampering

## Monitoring and Observability

- StatsD metrics for email delivery status
- Sentry error tracking for failures
- Rails logging for debugging
- Service tagging for identification in logs

## Conclusion

The vets-api notification system is designed around participant IDs as the primary identifier. For EP120 survivor pension claims, the use of the Veteran's participant ID throughout the entire flow means that:

1. **BGS lookups** will return the deceased Veteran's information
2. **VA Notify** will attempt to send emails to the Veteran's profile
3. **Email delivery** will fail because deceased Veterans don't have active email profiles

This technical architecture naturally prevents survivors from receiving notifications for EP120 claims, which aligns with the access control principle that survivors cannot access documents in the Veteran's eFolder.
