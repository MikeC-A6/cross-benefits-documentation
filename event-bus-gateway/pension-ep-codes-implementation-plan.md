# Pension EP Codes Implementation Plan
## EP120/EP180 Feature Flags + Google Analytics Campaign Tracking

## Overview

This document outlines the implementation plan to enable EP120 and EP180 pension claim decision letter notifications with granular feature flag controls and independent Google Analytics campaign tracking.

## Current State Analysis

### EventBus Gateway Flow
```
Kafka Events → EventBus Gateway → vets-api → LetterReadyEmailJob → VA Notify → Email
```

### Current Limitations
1. **No EP Code Information**: Only `template_id` passed to vets-api
2. **Global Feature Flag**: Single flag controls all email notifications  
3. **No Campaign Differentiation**: Cannot track pension vs. disability claims separately
4. **No Claim Type Filtering**: Cannot distinguish between different EP codes

### Current Implementation
```ruby
# EventBusGatewayController
def send_email
  if Flipper.enabled?(:event_bus_gateway_emails_enabled)
    EventBusGateway::LetterReadyEmailJob.perform_async(
      participant_id,
      send_email_params[:template_id]
    )
  end
  head :ok
end

# LetterReadyEmailJob  
def perform(participant_id, template_id)
  notify_client.send_email(
    recipient_identifier: { id_value: participant_id, id_type: 'PID' },
    template_id: template_id,
    personalisation: { 
      host: HOSTNAME_MAPPING[Settings.hostname] || Settings.hostname,
      first_name: get_first_name_from_participant_id(participant_id) 
    }
  )
end
```

## Implementation Strategy

### Core Changes Required

The implementation adds EP code as a parameter throughout the notification pipeline, enabling both feature flagging and campaign tracking.

## Phase 1: EventBus Gateway Updates

### Extract EP Code from Kafka Events
**File**: `eventbus-gateway/app/consumers/decision_letter_availability_consumer.rb`

```ruby
def send_email_notification(participant_id, template_id)
  # Extract EP code from the original Kafka event
  ep_code = extract_ep_code(@event_payload)
  
  payload = {
    template_id: template_id,
    ep_code: ep_code  # NEW: Include EP code
  }
  
  # Send to vets-api with EP code
  response = HTTParty.post(
    "#{ENV['API_URL']}/v0/event_bus_gateway/send_email",
    headers: { 'Authorization' => "Bearer #{access_token}" },
    body: payload
  )
end

private

def extract_ep_code(event_payload)
  # Extract first 3 digits of ClaimTypeCode
  event_payload['ClaimTypeCode']&.first(3)
end
```

## Phase 2: vets-api Controller Updates

### Update EventBusGatewayController
**File**: `app/controllers/v0/event_bus_gateway_controller.rb`

```ruby
class EventBusGatewayController < SignIn::ServiceAccountApplicationController
  def send_email
    ep_code = send_email_params[:ep_code]
    
    # Check global flag first
    return head :ok unless Flipper.enabled?(:event_bus_gateway_emails_enabled)
    
    # Check EP-specific flags
    return head :ok unless ep_code_enabled?(ep_code)
    
    EventBusGateway::LetterReadyEmailJob.perform_async(
      participant_id,
      send_email_params[:template_id],
      ep_code  # Pass EP code to job
    )
    
    head :ok
  end

  private

  def send_email_params
    params.permit(:template_id, :ep_code)  # NEW: Permit ep_code
  end

  def ep_code_enabled?(ep_code)
    case ep_code&.to_s
    when '120'
      Flipper.enabled?(:event_bus_gateway_ep120_pension_emails_enabled)
    when '180'
      Flipper.enabled?(:event_bus_gateway_ep180_pension_emails_enabled)
    else
      # For non-pension claims, use global flag only
      true
    end
  end
end
```

## Phase 3: Email Job Updates with Campaign Tracking

### Update LetterReadyEmailJob
**File**: `app/sidekiq/event_bus_gateway/letter_ready_email_job.rb`

```ruby
def perform(participant_id, template_id, ep_code = nil)
  # Add EP code to metrics/logging for observability
  StatsD.increment('event_bus_gateway.email.sent', tags: ["ep_code:#{ep_code}"])
  
  personalisation_data = {
    host: HOSTNAME_MAPPING[Settings.hostname] || Settings.hostname,
    first_name: get_first_name_from_participant_id(participant_id)
  }
  
  # Add campaign parameters for pension claims
  if ep_code.in?(['120', '180'])
    personalisation_data.merge!(campaign_params_for_pension(ep_code))
  end
  
  notify_client.send_email(
    recipient_identifier: { id_value: participant_id, id_type: 'PID' },
    template_id: template_id,
    personalisation: personalisation_data
  )
rescue => e
  record_email_send_failure(e, ep_code)
end

private

def campaign_params_for_pension(ep_code)
  {
    utm_source: 'decision_letter_email',
    utm_medium: 'email',
    utm_campaign: campaign_name_for_ep(ep_code),
    utm_content: "ep#{ep_code}_decision_letter"
  }
end

def campaign_name_for_ep(ep_code)
  case ep_code
  when '120'
    'pension_reopened_decision_notification'
  when '180'
    'pension_initial_decision_notification'
  else
    'decision_letter_notification'
  end
end

def record_email_send_failure(error, ep_code = nil)
  error_message = 'LetterReadyEmailJob errored'
  context = { message: error.message, ep_code: ep_code }
  
  ::Rails.logger.error(error_message, context)
  StatsD.increment('event_bus_gateway.email.failed', tags: ["ep_code:#{ep_code}"])
end
```

## Phase 4: VA Notify Template Updates

### Update Email Template
Add UTM parameter placeholders to the decision letter email template in VA Notify:

**Template placeholders to add:**
- `((utm_source))`
- `((utm_medium))`  
- `((utm_campaign))`
- `((utm_content))`

**Template links become:**
```
https://((host))/track/decision-letters/?utm_source=((utm_source))&utm_medium=((utm_medium))&utm_campaign=((utm_campaign))&utm_content=((utm_content))
```

## Phase 5: Feature Flag Configuration

### Create New Feature Flags
```ruby
# Via Flipper admin interface or console
Flipper[:event_bus_gateway_ep120_pension_emails_enabled].disable
Flipper[:event_bus_gateway_ep180_pension_emails_enabled].disable

# Existing global flag remains
Flipper[:event_bus_gateway_emails_enabled] # Already exists
```

### Campaign Tracking Structure

**UTM Parameter Structure for Pension Claims:**
- **utm_source**: `decision_letter_email` (consistent across all decision letters)
- **utm_medium**: `email` (consistent)
- **utm_campaign**: 
  - `pension_initial_decision_notification` (EP180)
  - `pension_reopened_decision_notification` (EP120)
- **utm_content**: 
  - `ep180_decision_letter`
  - `ep120_decision_letter`

## Testing Strategy

### Unit Tests for vets-api
```ruby
# spec/controllers/v0/event_bus_gateway_controller_spec.rb

describe 'EP-specific feature flags' do
  context 'when EP120 flag is disabled' do
    before do
      allow(Flipper).to receive(:enabled?)
        .with(:event_bus_gateway_emails_enabled).and_return(true)
      allow(Flipper).to receive(:enabled?)
        .with(:event_bus_gateway_ep120_pension_emails_enabled).and_return(false)
    end

    it 'does not send email for EP120 claims' do
      expect(EventBusGateway::LetterReadyEmailJob).not_to receive(:perform_async)
      
      post '/v0/event_bus_gateway/send_email', 
           params: { template_id: '5678', ep_code: '120' },
           headers: service_account_auth_header
      
      expect(response).to have_http_status(:ok)
    end
  end

  context 'when EP120 flag is enabled' do
    before do
      allow(Flipper).to receive(:enabled?)
        .with(:event_bus_gateway_emails_enabled).and_return(true)
      allow(Flipper).to receive(:enabled?)
        .with(:event_bus_gateway_ep120_pension_emails_enabled).and_return(true)
    end

    it 'sends email for EP120 claims' do
      expect(EventBusGateway::LetterReadyEmailJob)
        .to receive(:perform_async).with('1234', '5678', '120')
      
      post '/v0/event_bus_gateway/send_email',
           params: { template_id: '5678', ep_code: '120' },
           headers: service_account_auth_header
      
      expect(response).to have_http_status(:ok)
    end
  end
end

# Test campaign parameter generation
describe 'campaign parameter generation' do
  it 'generates correct UTM parameters for EP180' do
    job = EventBusGateway::LetterReadyEmailJob.new
    params = job.send(:campaign_params_for_pension, '180')
    
    expect(params[:utm_campaign]).to eq('pension_initial_decision_notification')
    expect(params[:utm_content]).to eq('ep180_decision_letter')
  end
end
```

### Integration Tests
1. **End-to-End Flow**: Verify EP code flows from EventBus Gateway → vets-api → VA Notify
2. **Feature Flag Logic**: Test that emails are properly gated by EP-specific flags
3. **Campaign Tracking**: Confirm UTM parameters are correctly populated in emails
4. **Analytics Validation**: Verify UTM parameters appear in Google Analytics

## Deployment Strategy

### Step 1: Deploy vets-api Changes
1. Deploy controller and job changes with feature flags **disabled**
2. Monitor for any issues with existing functionality
3. Verify backward compatibility (ep_code parameter is optional)

### Step 2: Update VA Notify Template
1. Add UTM parameter placeholders to decision letter template
2. Test template rendering with sample data

### Step 3: Deploy EventBus Gateway Changes  
1. Update EventBus Gateway to send EP codes
2. Monitor vets-api logs to confirm EP codes are being received

### Step 4: Enable Feature Flags Gradually
1. **Enable EP180 first** (lower risk, cleaner data - initial pension claims)
2. Monitor for 24-48 hours
3. **Enable EP120 carefully** with monitoring for survivor scenario edge cases

### Step 5: Validate Campaign Tracking
1. Monitor Google Analytics for new campaign data
2. Verify UTM parameters are tracking correctly
3. Confirm segmentation works between pension and other claim types

## Monitoring and Observability

### Metrics to Track
```ruby
# Email sending metrics by EP code
StatsD.increment('event_bus_gateway.email.sent', tags: ["ep_code:#{ep_code}"])
StatsD.increment('event_bus_gateway.email.failed', tags: ["ep_code:#{ep_code}"])
StatsD.increment('event_bus_gateway.email.feature_flag_blocked', tags: ["ep_code:#{ep_code}"])

# Feature flag usage metrics
StatsD.increment('flipper.flag_checked', tags: ["flag:event_bus_gateway_ep120_pension_emails_enabled"])
StatsD.increment('flipper.flag_checked', tags: ["flag:event_bus_gateway_ep180_pension_emails_enabled"])
```

### Enhanced Logging
```ruby
# Enhanced logging with EP code context
Rails.logger.info('EventBusGateway email request', {
  participant_id: participant_id,
  template_id: template_id,
  ep_code: ep_code,
  feature_flag_enabled: ep_code_enabled?(ep_code),
  campaign_params: ep_code.in?(['120', '180']) ? campaign_params_for_pension(ep_code) : nil
})
```

### Google Analytics Dashboard
Monitor the following campaign metrics:
- **Email Click-through Rates** by EP code
- **Conversion Funnel** from email → decision letter viewing  
- **User Segmentation** pension vs. disability claims
- **Geographic Distribution** of pension claim engagement

## Rollback Plan

### Immediate Response (< 5 minutes)
1. **Disable feature flags** immediately if issues occur
   - `Flipper[:event_bus_gateway_ep120_pension_emails_enabled].disable`
   - `Flipper[:event_bus_gateway_ep180_pension_emails_enabled].disable`

### Medium-term Response (< 30 minutes) 
1. EventBus Gateway continues working without EP codes (backward compatible)
2. Existing decision letter notifications remain unaffected

### Full Rollback (if necessary)
1. Roll back vets-api code changes
2. Roll back EventBus Gateway changes
3. Revert VA Notify template changes

## Benefits of This Implementation

### Feature Flag Benefits
1. **Granular Control**: Enable/disable EP120 and EP180 independently
2. **Safe Rollout**: Test EP180 first, then EP120 with survivor considerations  
3. **Backward Compatible**: Existing functionality continues to work
4. **Fast Rollback**: Feature flags provide immediate disable capability

### Campaign Tracking Benefits
1. **Segmented Analytics**: Track pension vs. disability claims separately
2. **Conversion Tracking**: Measure engagement rates by claim type
3. **User Journey Analysis**: Understand how different EP codes interact with decision letters
4. **A/B Testing Capability**: Test messaging strategies per claim type

### Combined Benefits
1. **Single Implementation**: One EP code parameter enables both features
2. **Observable**: Clear metrics and logging by EP code  
3. **Scalable**: Framework extends to future EP codes easily
4. **Data-Driven**: Campaign insights inform product decisions

## Success Metrics

### Technical Metrics
- **Email Delivery Rate**: >99% for enabled EP codes
- **Error Rate**: <0.1% for email sending
- **Feature Flag Response Time**: <100ms for flag checks

### Campaign Metrics  
- **Click-through Rate**: Baseline measurement for pension emails
- **Conversion Rate**: Email → decision letter viewing
- **Segmentation Quality**: Clear differentiation between claim types in GA

### Business Metrics
- **Volume Targets**: 
  - EP180: ~614 notifications/week
  - EP120: ~147 notifications/week  
- **User Satisfaction**: Maintained or improved email engagement rates

## Conclusion

This unified implementation provides both granular feature flag control and sophisticated campaign tracking for pension claims. The approach is backward compatible, incrementally deployable, and includes proper monitoring and rollback capabilities.

The EP code parameter addition creates a foundation for future enhancements while solving the immediate need for pension claim notification control and analytics insights.
