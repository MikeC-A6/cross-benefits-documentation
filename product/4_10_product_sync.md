# Product Sync Prep - 4/10/2025

---

## Lighthouse Migration

### Migrate to LightHouse Benefits Documents API from VBMS_Connect for retrieving claim letters

- Not started, but hopefully within a month or so we can 

### Migrate to LightHouse Benefits Documents API from EVSS document upload service for evidence upload

-   **Status:** Currently at 75% rollout due to higher than expected 504 errors during high traffic times.
-   Lighthouse contract was on a code freeze until 4/8.
-   **Next steps needed:** Follow up with Lighthouse team in [this thread](https://dsva.slack.com/archives/C02CQP3RFFX/p1743599304727799?thread_ts=1742584692.152089&cid=C02CQP3RFFX) to determine if a fix has been implemented in order to go to 100%.

### Migrate to Lighthouse from EVSS for letters generation

-   **Status:** Previously implemented at 25% but [rolled back](https://dsva.slack.com/archives/C04KHCT3ZMY/p1743012653750079).
-   **Blockers:**
    -   “Concerns with how Lighthouse is defining whether a user is a Veteran or Dependent, and specifically, whether the dependent is a beneficiary that should have access to the benefit summary letter. Lighthouse & VA.gov teams will meet with MPI team to discuss logic”
    -   “The healthcare eligibility center requested that we pause migration until the ACA letter content is updated and corrected. Lighthouse plans to update the letter content after their code freeze is lifted on 4/8.”

**Summary:** For the second and third, there is little technical work required - mostly needs follow-up and monitoring. For the first, BMT1 will take on this work

---

## Component Migration

-   CST’s current `va-file-input` component is considered an “[imposter component](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547)” and needs to be upgraded to a USWDS V3 component.
-   **November 2024:** Need identified for the USWDS V3 VADS component to allow for conditional display of the slot.
-   **Nov/Dec 2024:** Several other issues identified.
-   **January 2025:** Identified issues resolved and closed by the platform team.
-   **February 6, 2025:** Another [issue](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785) was identified where `va-file-input-multiple` is unable to conditionally render slots.
-   **March 2025:** Platform team opened an investigation ticket, followed up by BMT1. Platform team responded in a GitHub comment in March that they were working on a “proof of concept” before working on the ticket, aiming for completion “in April”.

**Summary:** There is currently a dependency on the platform team to fix outstanding bugs with the `va-file-input` component before it can replace the current component.

### Assumption(s) to be verified

-   `va-file-input` component is the only component in CST that needs to be updated to USWDS V3.
-   Once we migrate to the new component, the “component migration” goal for Q2, 2025 will effectively be completed.

### Potential Pathway

-   The BMT1 team has team members with the skills and experience to update the [File input](https://design.va.gov/components/form/file-input#:~:text=This%20guida) component in a way that aligns with the current implementation, but is extended to meet the need to conditionally render (multiple) slots.
-   **Proposal:** Prove out an end-to-end implementation by:
    1.  Forking the `component-library` repo.
    2.  Fixing the `va-file-input-multiple` component locally.
    3.  Hooking up the fixed component locally to the CST.
-   **Potential value:**
    -   Could serve as a reference implementation for the platform team, potentially speeding up resolution.
    -   *Might* be a candidate for the platform team to accept to merge in as the new component.
    -   Allows the team to build experience delivering an end-to-end solution.

---

## Decision Letter Notifications (event-based architecture)

### Assumption(s) to be verified

-   Accountability to continue making progress with the decision letter notifications is **currently** with BMT 2.

---

## Document Status Feature

-   Seek clarity on what is needed here.
-   Does the “files” tab in staging represent the “document status feature?”
-   If so, what work remains to begin rolling out this feature into production? (assuming it isn’t already)

---

## Load Time Performance Improvement Discovery

-   Need more clarity about specific pain points with load times:
    -   Is it an initial claim list load?
    -   Claim detail load? Both?
-   Any previous research / metrics available?
-   **Recommendation:** **Make this the first area of focus for BMT1**
    -   Allows us to get up to speed on the user journey through the CST app.
    -   Identify data sources for metrics (Datadog, GA, DOMO).
    -   Bring engineers up to speed on the code base.
-   Consider a 1 week “discovery sprint” where the entire team is focused on producing a readout of findings.
