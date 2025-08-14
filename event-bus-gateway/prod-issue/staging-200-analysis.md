## Staging 200 OK but No Email Sent — Deep Analysis

### Scope
Why would `POST /v0/event_bus_gateway/send_email` return 200 in staging but no decision-letter email is actually sent?

### Verified behaviors from the PRs
- Vets-API change (required fields + gating):
  - `ep_code` and `template_id` are required; missing either returns 400. Otherwise, endpoint returns 200. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
  - For `EP120` and `EP180`, email enqueue occurs only when their feature flags are enabled (`ep120_decision_letter_notifications`, `ep180_decision_letter_notifications`). If disabled, endpoint still returns 200 but does not enqueue. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
  - For other EP codes (e.g., `EP010`, `EP020`, `EP110`), no feature flag gating applies; if params are valid, job is enqueued. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

- Event Bus Gateway change (payload sent):
  - Gateway now posts `ep_code` as `'EP' + first 3` digits from the Kafka event `ClaimTypeCode` (e.g., `020XYZ` → `EP020`). Source: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
  - Gateway includes `template_id` from `ENV["EMAIL_TEMPLATE_ID"]`. Source: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)

### 200 with no email — code paths that explicitly allow this
- Pension codes with flags disabled:
  - If `ep_code` is `EP120` or `EP180` and the corresponding Flipper flag is disabled, the controller returns 200 and does not enqueue any job. This is by design in the new logic. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

### 200 but no email — plausible causes outside explicit gating
For non-flagged codes like `EP020` (or for flagged codes when flags are enabled), a 200 should be followed by enqueue → job execution. If email still doesn’t arrive:

- Sidekiq execution not occurring
  - Workers not running or queue misconfiguration would prevent the enqueued `EventBusGateway::LetterReadyEmailJob` from executing. The PR did not change queue names or sidekiq options, so this would be environmental. Source (no queue change evident in diff): [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

- VA Notify send failure at runtime
  - The job calls `VaNotify::Service.new(...).send_email(...)`. If the Notify API key, template ID, or VANotify environment config is invalid in staging, the job rescues and records a failure; the controller has already returned 200. Source: `letter_ready_email_job.rb` changes in [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

- Incorrect `template_id` supplied by Gateway in staging
  - Gateway reads `ENV["EMAIL_TEMPLATE_ID"]`. If this env var is unset, wrong, or points to a non-existent/disabled template in VANotify, the job will raise during send and be rescued (no email). The API still returns 200. Source: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)

- `ep_code` validation failure silently preventing enqueue
  - The controller requires `ep_code` to be a present String. If it’s not, `decision_letter_enabled?` returns false and no job enqueues, but the controller still returns 200. The Gateway posts `ep_code` as a string like `EP020`, so this should not occur when using Gateway as the caller. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

- GA tracking does not block email
  - Tracking is executed after the VANotify send; errors are rescued and logged and do not prevent sending. Source: `letter_ready_email_job.rb` in [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

### Diagnostics to pinpoint the staging failure
Run these targeted checks in staging:

- Verify Flipper flags for pension codes
  - Ensure `ep120_decision_letter_notifications` and/or `ep180_decision_letter_notifications` are enabled when testing those codes. Otherwise, 200 with no enqueue is expected. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)

- Confirm job enqueue in Rails logs
  - Look for `EventBusGateway::LetterReadyEmailJob.perform_async(...)` invocation lines around request time. If missing, inspect request params captured in logs and validate `ep_code` presence and format.

- Check Sidekiq queues and processed counts
  - Inspect the queue containing `EventBusGateway::LetterReadyEmailJob` for growing backlog or non-processing. Validate that workers are running with access to Notify credentials.

- Inspect VANotify call outcomes
  - Search for `record_email_send_failure` log entries or VANotify API response errors around the test time. An invalid `template_id` or API key would explain 200/no-email.

- Validate Gateway staging configuration
  - Confirm `ENV["EMAIL_TEMPLATE_ID"]` is set to a valid VANotify template ID for staging.
  - Ensure Gateway posts `ep_code` as expected by checking request logs (e.g., `EP020`, `EP110`, `EP120`, `EP180`). Source: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)

### Bottom line
- A 200 with no email for `EP120`/`EP180` with flags disabled is expected per the new design in Vets-API.
- For other codes (e.g., `EP020`), 200/no-email indicates a downstream failure (Sidekiq not processing or VANotify send failure) or a pre-enqueue validation/gating result that prevented job enqueue (less likely given Gateway’s payload shape).

### References
- Vets-API: [PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Event Bus Gateway: [PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)

