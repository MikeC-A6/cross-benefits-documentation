## Decision Letter Notifications: Avro Schema + Staging Findings

### Context
Goal: explain staging cases where `POST /v0/event_bus_gateway/send_email` returns 200 but no email is sent, using the Avro schema and the two integration PRs as ground truth.

References:
- Vets-API change (required params, gating, tracking): [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Event Bus Gateway change (derive and send `ep_code`): [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- Decision letter Avro schema: `decision_letter_availability-value.avsc` ([link](https://github.com/department-of-veterans-affairs/ves-event-bus-infra-argocd-applications-vault/blob/main/deploy/schema-registry/schemas/decision_letter_availability-value.avsc))

### What the schema tells us
- Many fields are nullable unions; e.g., `ClaimTypeCode` is `{"type": ["null","string"]}`. The deserializer should yield a Ruby String (e.g., `"020XYZ"`) or `nil`.
- A sample payload you provided uses the union-wrapped form for strings (e.g., `"ClaimTypeCode":{"string":"040SCR"}`), which the gateway deserializer must convert to a plain String before substring logic is applied.

### Gateway behavior (confirmed in code)
- Relevant claim codes filter:
  - `RELEVANT_CLAIM_CODES = %w[010 110 020 120 180]`
  - Only events whose `ClaimTypeCode[0..2]` is in that list are processed. Otherwise, the consumer does nothing. Source: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- Derivation of `ep_code` as a String:
  - `ep_code: "EP#{claim_type_code[0..2]}"` (string interpolation guarantees a String). Tests assert string payloads like `'EP010'`, `'EP120'`, `'EP180'`. Source: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- `template_id` is taken from `ENV["EMAIL_TEMPLATE_ID"]` and POSTed with `ep_code` to Vets-API.

### Vets-API behavior (confirmed in code)
- Required params and return codes:
  - Requires `template_id` and `ep_code`. Missing either → 400. Otherwise returns 200. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Gating semantics:
  - Only `EP120` and `EP180` are gated by feature flags. All other codes (e.g., `EP020`) are “always enabled.” Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Enqueue vs. response:
  - If gating/validation passes, `EventBusGateway::LetterReadyEmailJob.perform_async(...)` is called. The controller always returns `head :ok` (200) at the end. Source: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Runtime failures are rescued in the job:
  - `VaNotify::Service#send_email(...)` exceptions are rescued; failures are logged/metriced via `record_email_send_failure`. The 200 has already been sent by the controller.

### Explaining two observed staging test modes
1) Kafka-driven test with sample `ClaimTypeCode: 040SCR`
   - Gateway behavior: derives `ep_code = "EP040"`, which is NOT in `RELEVANT_CLAIM_CODES`.
   - Result: gateway consumer does not POST to Vets-API; no email expected by design.
   - Action: when testing Kafka, ensure `ClaimTypeCode` begins with one of 010, 110, 020, 120, 180.

2) Direct POST test with `ep_code: "EP020"`
   - Vets-API behavior: `EP020` is not gated; with a present String `ep_code` and `template_id`, controller enqueues the job and returns 200.
   - If 200 but no email: causes are post-controller, most likely:
     - Sidekiq not processing the queue (workers down, queue mismatch).
     - VANotify send failure (invalid API key/template/personalisation) rescued inside the job.
     - Misconfigured `EMAIL_TEMPLATE_ID` in gateway leading to a send error.
   - Validation edge cases are unlikely for gateway-originated requests because `ep_code` is explicitly constructed as a String.

### Checklist to isolate staging 200/no-email
- Confirm the test mode:
  - Kafka test: `ClaimTypeCode` must start with a relevant code; verify deserializer yields a plain String (not union wrapper).
  - Direct POST: verify request body includes `template_id` and `ep_code: "EP020"`.
- Vets-API logs:
  - Check for `EventBusGateway::LetterReadyEmailJob.perform_async(participant_id, template_id, "EP020")` and subsequent job execution logs.
  - Search for `record_email_send_failure` entries and VANotify error details.
- Sidekiq:
  - Ensure staging workers are up and processing the queue containing `EventBusGateway::LetterReadyEmailJob`.
- VANotify:
  - Validate staging `EMAIL_TEMPLATE_ID` exists/enabled and personalisation keys match (`host`, `first_name`).

### Bottom line
- The schema and gateway code explain why 040SCR would never send: it’s filtered out before any POST.
- For EP020 direct POST, 200 means validation passed and enqueue should occur; no email indicates Sidekiq or VANotify runtime issues, not parameter validation.

### Citations
- Vets-API changes and gating: [vets-api PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Event Bus Gateway consumer and payload: [eventbus-gateway PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- Avro schema: `decision_letter_availability-value.avsc` ([link](https://github.com/department-of-veterans-affairs/ves-event-bus-infra-argocd-applications-vault/blob/main/deploy/schema-registry/schemas/decision_letter_availability-value.avsc))


