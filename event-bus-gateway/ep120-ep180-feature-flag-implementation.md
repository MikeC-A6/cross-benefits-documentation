# EP120/EP180 Feature Flag Implementation for vets-api

## Current State Analysis

### Existing EventBusGatewayController Implementation
```ruby
# app/controllers/v0/event_bus_gateway_controller.rb
class EventBusGatewayController < SignIn::ServiceAccountApplicationController
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

  def send_email_params
    params.permit(:template_id)
  end
end
```

### Current Limitations
1. **No EP Code Information**: Only receives `template_id` parameter
2. **Global Feature Flag**: Single flag controls all email notifications
3. **No Claim Type Filtering**: Cannot distinguish between EP120/EP180 vs other claims

## Implementation Options

### Option 1: Add EP Code Parameter (Recommended)

#### Changes to EventBus Gateway
**File**: `eventbus-gateway/app/consumers/decision_letter_availability_consumer.rb`

```ruby
# BEFORE: Only sends template_id
def send_email_notification(participant_id, template_id)
  payload = { template_id: template_id }
  # ... send to vets-api
end

# AFTER: Include EP code
def send_email_notification(participant_id, template_id, ep_code)
  payload = { 
    template_id: template_id,
    ep_code: ep_code  # NEW: Extract from original Kafka event
  }
  # ... send to vets-api
end
```

#### Changes to vets-api Controller
**File**: `app/controllers/v0/event_bus_gateway_controller.rb`

```ruby
class EventBusGatewayController < SignIn::ServiceAccountApplicationController
  def send_email
    ep_code = send_email_params[:ep_code]
    
    # Check global flag first
    return head :ok unless Flipper.enabled?(:event_bus_gateway_emails_enabled)
    
    # Check EP-specific flags
    return head :ok unless ep_enabled?(ep_code)
    
    EventBusGateway::LetterReadyEmailJob.perform_async(
      participant_id,
      send_email_params[:template_id],
      ep_code  # Pass EP code to job for logging/metrics
    )
    
    head :ok
  end

  private

  def send_email_params
    params.permit(:template_id, :ep_code)  # NEW: Permit ep_code
  end

  def ep_enabled?(ep_code)
    case ep_code
    when '120'
      Flipper.enabled?(:event_bus_gateway_ep120_pension_emails_enabled)
    when '180'  
      Flipper.enabled?(:event_bus_gateway_ep180_pension_emails_enabled)
    else
      true  # Default: allow other claim types if global flag is enabled
    end
  end
end
```

#### Feature Flag Configuration
**Files**: `config/features.yml` or via Flipper Admin

```yaml
# New feature flags
event_bus_gateway_ep120_pension_emails_enabled: false
event_bus_gateway_ep180_pension_emails_enabled: false

# Existing global flag
event_bus_gateway_emails_enabled: true
```

### Option 2: Template ID Encoding (Alternative)

If adding parameters is not feasible, encode EP information in template IDs:

```ruby
# Template ID naming convention
# Format: {base_template_id}_{ep_code}
# Examples:
#   - "decision_letter_001_120" (EP120 pension)
#   - "decision_letter_001_180" (EP180 pension)  
#   - "decision_letter_001_110" (disability)

def ep_enabled?(template_id)
  ep_code = extract_ep_from_template(template_id)
  
  case ep_code
  when '120'
    Flipper.enabled?(:event_bus_gateway_ep120_pension_emails_enabled)
  when '180'
    Flipper.enabled?(:event_bus_gateway_ep180_pension_emails_enabled)
  else
    true
  end
end

def extract_ep_from_template(template_id)
  # Extract EP code from template_id suffix
  template_id.split('_').last if template_id.include?('_')
end
```

## Recommended Implementation Plan

### Phase 1: Implement EP Code Parameter Approach

#### Step 1: Update vets-api Controller
```ruby
# app/controllers/v0/event_bus_gateway_controller.rb

def send_email
  ep_code = send_email_params[:ep_code]
  
  # Global feature flag check
  return head :ok unless Flipper.enabled?(:event_bus_gateway_emails_enabled)
  
  # EP-specific feature flag check
  return head :ok unless ep_code_enabled?(ep_code)
  
  EventBusGateway::LetterReadyEmailJob.perform_async(
    participant_id,
    send_email_params[:template_id],
    ep_code
  )
  
  head :ok
end

private

def send_email_params
  params.permit(:template_id, :ep_code)
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
```

#### Step 2: Update LetterReadyEmailJob
```ruby
# app/sidekiq/event_bus_gateway/letter_ready_email_job.rb

def perform(participant_id, template_id, ep_code = nil)
  # Add EP code to metrics/logging for observability
  StatsD.increment('event_bus_gateway.email.sent', tags: ["ep_code:#{ep_code}"])
  
  notify_client.send_email(
    recipient_identifier: { id_value: participant_id, id_type: 'PID' },
    template_id:,
    personalisation: {
      host: HOSTNAME_MAPPING[Settings.hostname] || Settings.hostname,
      first_name: get_first_name_from_participant_id(participant_id)
    }
  )
rescue => e
  record_email_send_failure(e, ep_code)
end

private

def record_email_send_failure(error, ep_code = nil)
  error_message = 'LetterReadyEmailJob errored'
  context = { message: error.message, ep_code: ep_code }
  
  ::Rails.logger.error(error_message, context)
  StatsD.increment('event_bus_gateway.email.failed', tags: ["ep_code:#{ep_code}"])
end
```

#### Step 3: Create Feature Flags
Using Flipper admin interface or configuration:

```ruby
# Via console or migration
Flipper[:event_bus_gateway_ep120_pension_emails_enabled].disable
Flipper[:event_bus_gateway_ep180_pension_emails_enabled].disable
```

#### Step 4: Update EventBus Gateway
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

### Phase 2: Testing Strategy

#### Unit Tests for vets-api
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
```

## Deployment Strategy

### Step 1: Deploy vets-api Changes
1. Deploy controller changes with feature flags **disabled**
2. Monitor for any issues with existing functionality
3. Verify backward compatibility (ep_code parameter is optional)

### Step 2: Deploy EventBus Gateway Changes
1. Update EventBus Gateway to send EP codes
2. Monitor vets-api logs to confirm EP codes are being received

### Step 3: Enable Feature Flags
1. Enable EP180 first (lower risk, cleaner data)
2. Monitor for 24-48 hours
3. Enable EP120 with careful monitoring for survivor scenario

### Step 4: Rollback Plan
1. Disable feature flags immediately if issues occur
2. EventBus Gateway continues working without EP codes (backward compatible)
3. Roll back code changes if necessary

## Monitoring and Observability

### Metrics to Track
```ruby
# Email sending metrics by EP code
StatsD.increment('event_bus_gateway.email.sent', tags: ["ep_code:#{ep_code}"])
StatsD.increment('event_bus_gateway.email.failed', tags: ["ep_code:#{ep_code}"])
StatsD.increment('event_bus_gateway.email.feature_flag_blocked', tags: ["ep_code:#{ep_code}"])

# Feature flag usage metrics
StatsD.increment('flipper.flag_checked', tags: ["flag:event_bus_gateway_ep120_pension_emails_enabled"])
```

### Logging Enhancements
```ruby
# Enhanced logging with EP code context
Rails.logger.info('EventBusGateway email request', {
  participant_id: participant_id,
  template_id: template_id,
  ep_code: ep_code,
  feature_flag_enabled: ep_code_enabled?(ep_code)
})
```

## Benefits of This Approach

1. **Granular Control**: Can enable/disable EP120 and EP180 independently
2. **Safe Rollout**: Can test EP180 first, then EP120 with survivor considerations
3. **Backward Compatible**: Existing functionality continues to work
4. **Observable**: Clear metrics and logging by EP code
5. **Fast Rollback**: Feature flags provide immediate disable capability

## Conclusion

The recommended approach adds EP code as a parameter and implements EP-specific feature flags. This provides the granular control needed to safely roll out pension claim notifications while avoiding the survivor notification issue for EP120 claims.

The implementation is backward compatible and can be deployed incrementally with proper monitoring and rollback capabilities.
