## Root Cause Analysis: Decision Letter Email Notification Breakage

### Summary
- **Issue**: Emails for decision letter notifications stopped due to a breaking request contract change between the Event Bus Gateway and Vets-API.
- **Impact**: Calls to `POST /v0/event_bus_gateway/send_email` that did not include `ep_code` began returning 400. Calls that did include `ep_code` for pension codes (EP120/EP180) returned 200 but did not enqueue when feature flags were disabled.
- **Root cause**: Vets-API endpoint `POST /v0/event_bus_gateway/send_email` began requiring a new field, `ep_code`, and added feature-flag gating. Gateway in production was not yet sending `ep_code` at the time of the Vets-API deployment.
- **Status**: Change was reverted on 2025-08-14T15:54:51Z via Vets-API [PR #23555](https://github.com/department-of-veterans-affairs/vets-api/pull/23555).

References:
- Vets-API change: [PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Event Bus Gateway change: [PR #122](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- Revert (Vets-API): [PR #23555](https://github.com/department-of-veterans-affairs/vets-api/pull/23555)

### High-level details
- Vets-API PR #23455 introduced:
  - Required params: `template_id` and `ep_code`; missing either returns 400.
  - Feature flags for pension codes: `ep120_decision_letter_notifications`, `ep180_decision_letter_notifications`.
  - Silent success: returns 200 even if email job is skipped due to flags/validation.
- Event Bus Gateway PR #122 added:
  - Support for pension claim codes (120, 180).
  - New payload to Vets-API: includes `ep_code` as an `'EP' + first 3` string (e.g., `120XYZ` → `EP120`).

### Root cause
- The Vets-API change altered the request contract by making `ep_code` mandatory before the Gateway change was active everywhere. Calls without `ep_code` immediately failed with 400. Where `ep_code` was present but flags were disabled, calls returned 200 but did not enqueue emails.

### Contributing factors
- Lack of backward compatibility window on the Vets-API endpoint.
- Silent 200 response when enqueue is skipped (flags off or validation failure), obscuring failures.
- Tight coupling on exact `ep_code` values for feature flags (`EP120`, `EP180`).

### Corrective and preventive actions (proposed)
- Re-land with a compatibility window: treat missing `ep_code` as legacy behavior for a fixed period with deprecation logs.
- Normalize `ep_code` input (coerce to upper-cased string with `EP` prefix) to reduce format errors.
- Return explicit “not enqueued” responses or add structured body/metrics when gating prevents work.
- Coordinate rollout order: deploy Gateway first, enable flags, then deploy Vets-API changes.

### Detailed timeline (UTC)
- 2025-08-07T15:20:40Z — Vets-API PR #23455 opened. [link](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- 2025-08-07T15:29:00Z — Event Bus Gateway PR #122 opened. [link](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- 2025-08-12T19:29:07Z — Vets-API PR #23455 merged. [link](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- 2025-08-12T19:38:22Z — Event Bus Gateway PR #122 merged. [link](https://github.com/department-of-veterans-affairs/eventbus-gateway/pull/122)
- 2025-08-14T12:41:30Z — Revert PR #23555 opened. [link](https://github.com/department-of-veterans-affairs/vets-api/pull/23555)
- 2025-08-14T15:54:51Z — Revert PR #23555 merged. [link](https://github.com/department-of-veterans-affairs/vets-api/pull/23555)

Note: Service impact started after the Vets-API deployment of PR #23455 when callers were not yet providing `ep_code`, and persisted where flags were disabled or input validation failed.


