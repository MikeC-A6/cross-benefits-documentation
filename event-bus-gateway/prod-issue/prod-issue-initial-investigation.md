## Analysis: Why PR #23455 Broke Event Bus Gateway → Vets-API Email Flow

Links for reference:
- PR: [115352 ep120 ep180 email notifications](https://github.com/department-of-veterans-affairs/vets-api/pull/23455)
- Revert PR: [Revert "115352 ep120 ep180 email notifications" (#23555)](https://github.com/department-of-veterans-affairs/vets-api/pull/23555)

### Executive summary
- **Breaking contract change**: `v0/event_bus_gateway/send_email` began requiring `ep_code` in the request body and returns 400 when missing.
- **Prod mismatch**: Event Bus Gateway in production did not yet send `ep_code` (its companion change was not deployed), resulting in immediate 400s after deploy.
- **Staging behavior**: Endpoint returned 200 but did not send emails when feature flags were off or when `ep_code` failed validation, leading to silent no-ops.

Together, these explain the observed 400s in prod and the 200/no-email in staging recorded in `decision_letter_notifications/prod_issue.md`.

### What changed in PR #23455
Key changes relevant to the integration:

- `app/controllers/v0/event_bus_gateway_controller.rb`
  - Added FEATURE_FLAG gating for `EP120` and `EP180`.
  - Added parameter validation: `ep_code` and `template_id` are now both required. Missing either returns 400 with an error message.
  - Only enqueues `EventBusGateway::LetterReadyEmailJob` if `decision_letter_enabled?` returns true; otherwise responds 200 without enqueueing.
  - `send_email_params` now permits `:ep_code`.

- `app/sidekiq/event_bus_gateway/letter_ready_email_job.rb`
  - Job signature updated to accept optional `ep_code` and added GA/Datadog tracking.

- `config/features.yml`
  - Introduced feature flags: `ep120_decision_letter_notifications`, `ep180_decision_letter_notifications`.

- Swagger and specs updated to document and test the new behavior, including 400s for missing params.

Sources: [PR #23455 Files](https://github.com/department-of-veterans-affairs/vets-api/pull/23455/files)

### Why this broke the Gateway → Vets-API connection
1) Breaking API request contract (immediate prod 400s)
- Before: Gateway sent `{ template_id }`.
- After PR: Vets-API requires `{ template_id, ep_code }`.
- The related Event Bus Gateway update (to add `ep_code`) had not been deployed to prod at the time (see `prod_issue.md`). Vets-API therefore returned 400 “ep_code is required.”

2) Silent no-op on staging (200 OK, no email)
- Controller now returns 200 even if no job is enqueued when the feature flag check fails.
- Two common ways this can happen:
  - Flags off: For `EP120`/`EP180`, `decision_letter_enabled?` returns false unless the respective Flipper flag is enabled. With flags off, the controller returns 200 without enqueueing.
  - Type/format validation failure: `decision_letter_enabled?` calls `valid_ep_code_format?`, which returns false unless `ep_code` is a non-empty String. If the Gateway sends a number (e.g., `120` as a numeric JSON value) rather than a string (`"120"` or `"EP120"`), validation fails → no enqueue → 200 with no email.

3) Backward compatibility gap for non-pension EP codes
- The PR made `ep_code` universally required for the same endpoint that previously did not require it. Even for existing decision letter flows (e.g., `EP020`, `EP110`), calls now fail if `ep_code` is absent. This explains staging regressions when testing MVP codes like `EP020`.

4) Inconsistent normalization/flag semantics for EP codes
- Feature flags are keyed only for `'EP120'` and `'EP180'` (with the `'EP'` prefix). If the Gateway sends `'120'` or `'180'` as strings, those bypass flag gating and will send emails even when flags are off. If it sends numbers (120/180), validation fails and no email is sent. This inconsistency increases the chance of environment-specific surprises.

### Mapping observed symptoms to changes
- Prod: “400s started around 11 yesterday” and correlated with merge time — consistent with missing `ep_code` being rejected by new validation. See discussion in `prod_issue.md` and PR timeline in [PR #23455](https://github.com/department-of-veterans-affairs/vets-api/pull/23455).
- Staging: “200 but not receiving emails” — consistent with either flags being off or `ep_code` failing the string validation, both of which skip enqueue but still return 200.
- Staging regression for `EP020`: Now requires `ep_code` for all codes; if Gateway didn’t include it or sent the wrong type, no job would be enqueued.

### Risk points introduced by PR #23455
- Cross-service coupling: Introduced a hard request contract change without a rollback/compat window.
- Silent success path: 200 OK even when work is skipped (flags/validation) makes failures hard to detect in clients.
- EP code ambiguity: The controller recognizes flags only for exact `'EP120'`/`'EP180'`, but allows other formats elsewhere; lack of normalization invites type/format mistakes.

### Recommendations
Short-term (to restore service safely):
- Keep the revert in place until Gateway and flags are aligned in all environments. See [Revert PR #23555](https://github.com/department-of-veterans-affairs/vets-api/pull/23555).

Hardening changes before re-landing:
- Backward compatibility window:
  - Make `ep_code` optional temporarily. If missing, continue prior behavior (enqueue) for legacy codes, and log a deprecation warning with rate limits.
- Normalize `ep_code` at the boundary:
  - Accept string or numeric; coerce to string; prepend `EP` where missing; left-pad numeric codes if needed; upcase. Use a single normalized value for gating and tracking.
- Fail loudly when skipping work:
  - If flags or validation cause skipping, return 202 with a machine-readable reason, or 409/412 depending on policy; alternatively return 200 with a clear response body indicating “not enqueued” and why. Add Datadog counters for skipped enqueues by reason.
- Feature flag semantics:
  - Gate by normalized code set, not raw input; consider allowing a configuration list rather than hardcoding in the controller.
- Deployment plan:
  - Stage rollout: deploy Gateway first (including `ep_code` and correct type/format), then enable flags, then deploy Vets-API change. Validate in staging with flags toggled and a suite of representative EP codes.

### Files and changes of interest
- `app/controllers/v0/event_bus_gateway_controller.rb`: new param requirements, gating, and decision path.
- `app/sidekiq/event_bus_gateway/letter_ready_email_job.rb`: new job signature and tracking.
- `config/features.yml`: added `ep120_decision_letter_notifications`, `ep180_decision_letter_notifications`.
- `spec/*`: updated tests documenting 400s for missing params and the new enqueue shapes.
See [PR #23455 files](https://github.com/department-of-veterans-affairs/vets-api/pull/23455/files).

### Appendix: context excerpts
- Team discussion captured in `decision_letter_notifications/prod_issue.md` indicates:
  - Prod had not deployed the Gateway change that adds `ep_code`.
  - Staging initially had flags off; flipping flags affected results, but MVP code `EP020` still regressed.
  - Decision to revert to restore email delivery while continuing investigation.


