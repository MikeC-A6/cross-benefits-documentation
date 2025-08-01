# Google Analytics Campaign Implementation for Pension EP Codes

## Current Campaign Implementation Analysis

### How GA Campaigns Work in VA Email System

Based on analysis of the current vets-api implementation, Google Analytics campaigns are implemented at **two levels**:

#### 1. **VA Notify Email Templates** (Primary Implementation)
- UTM parameters are embedded directly in the email templates within VA Notify
- Templates receive personalization data from vets-api including `host` and dynamic content
- Each template can include campaign-specific UTM parameters in email links

#### 2. **Direct Email Templates** (Secondary - Limited Use)
- Some internal email templates (like STEM confirmations) include hardcoded UTM parameters
- Example from `stem_applicant_confirmation.html.erb`:
  ```html
  href="https://www.va.gov/education/after-you-apply/?utm_source=confirmation_email&utm_medium=email&utm_campaign=what_happens_after_you_apply"
  ```

### Current Decision Letter Email Implementation

From `EventBusGateway::LetterReadyEmailJob`:

```ruby
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

**Key Limitation**: Currently **no EP code information** is passed to the email template, making EP-specific campaign tracking impossible.

## Implementation Strategy for Pension EP Codes

### Option 1: EP-Specific Campaign Parameters (Recommended)

#### Changes Required:

1. **Modify EventBusGatewayController** to accept `ep_code`:
   ```ruby
   def send_email_params
     params.permit(:template_id, :ep_code)
   end
   ```

2. **Update LetterReadyEmailJob** to pass campaign data:
   ```ruby
   def perform(participant_id, template_id, ep_code = nil)
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
   ```

3. **Update VA Notify Template** to use campaign parameters:
   - Add placeholders in the template: `((utm_source))`, `((utm_medium))`, `((utm_campaign))`, `((utm_content))`
   - Template links would become:
     ```
     https://((host))/track/decision-letters/?utm_source=((utm_source))&utm_medium=((utm_medium))&utm_campaign=((utm_campaign))&utm_content=((utm_content))
     ```

4. **EventBus Gateway Integration**:
   - Extract EP code from Kafka events (`ClaimTypeCode` field)
   - Pass EP code to vets-api endpoint

### Option 2: EP-Specific Templates

Alternative approach would be to create separate VA Notify templates for each EP code:
- `template_id_ep120_pension_decision`
- `template_id_ep180_pension_decision`

Each template would have hardcoded, EP-specific UTM parameters.

## Campaign Naming Convention

### Recommended UTM Structure for Pension Claims:

- **utm_source**: `decision_letter_email` (consistent across all decision letters)
- **utm_medium**: `email` (consistent)
- **utm_campaign**: 
  - `pension_initial_decision_notification` (EP180)
  - `pension_reopened_decision_notification` (EP120)
- **utm_content**: 
  - `ep180_decision_letter`
  - `ep120_decision_letter`

### Campaign Benefits:

1. **Segmented Analytics**: Track pension vs. disability vs. other claim types
2. **Conversion Tracking**: Measure click-through rates by EP code
3. **User Journey Analysis**: Understand how different claim types engage with decision letters
4. **A/B Testing**: Test different messaging for pension vs. other claims

## Integration Points

### Required Changes Summary:

1. **EventBus Gateway**: Extract and send EP code from Kafka events
2. **EventBusGatewayController**: Accept `ep_code` parameter
3. **LetterReadyEmailJob**: Generate campaign parameters based on EP code
4. **VA Notify Template**: Add UTM parameter placeholders
5. **Feature Flags**: Gate EP-specific campaign tracking with feature flags

### Testing Strategy:

1. **Unit Tests**: Test campaign parameter generation logic
2. **Integration Tests**: Verify EP code flow from EventBus Gateway → vets-api → VA Notify
3. **Analytics Validation**: Confirm UTM parameters appear correctly in Google Analytics

This approach allows for **independent campaign tracking** for pension EP codes while maintaining backward compatibility with existing decision letter notifications.
