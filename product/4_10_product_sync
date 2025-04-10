# Product Sync Prep - 4/10/2025

## Lighthouse Migration

### Migrate to LightHouse Benefits Documents API from VBMS_Connect

-   **Status:** Currently at 75% rollout due to higher than expected 504 errors during high traffic times.
-   Lighthouse contract was on a code freeze until 4/8.
-   **Next steps needed:** Follow up with Lighthouse team in this thread to determine if a fix has been implemented in order to go to 100%.

### Migrate to Lighthouse from EVSS for letters generation

-   **Status:** Previously implemented at 25% but rolled back.
-   **Blockers:**
    -   “Concerns with how Lighthouse is defining whether a user is a Veteran or Dependent, and specifically, whether the dependent is a beneficiary that should have access to the benefit summary letter. Lighthouse & VA.gov teams will meet with MPI team to discuss logic”
    -   “The healthcare eligibility center requested that we pause migration until the ACA letter content is updated and corrected. Lighthouse plans to update the letter content after their code freeze is lifted on 4/8.”

**Summary:** Very little technical work required - mostly needs follow-up and monitoring.

---

## Component Migration

### Background

-   CST’s current `va-file-input` component is considered an “imposter component” and needs to be upgraded to a USWDS V3 component (`va-file-input-multiple`).
-   **November 2024:** Need identified for conditional display of slots in the V3 component.
-   **Nov/Dec 2024:** Several other issues identified (double event emission, duplicate file selection).
-   **January 2025:** Platform team resolved the initially identified issues.
-   **February 6, 2025:** New issue identified: `va-file-input-multiple` is unable to conditionally render slots as needed by CST (e.g., show doc type for all, password for some).
-   **March 2025:** Platform team acknowledged the limitation, planned a "proof of concept" for April to find a better solution.

**Summary:** There is currently a dependency on the platform team to fix outstanding bugs with the `va-file-input` component before it can replace the current component in CST.

### Assumption(s) to be verified

-   `va-file-input` component is the only component in CST that needs to be updated to USWDS V3.
-   Once we migrate to the new component, the “component migration” goal for Q2, 2025 will effectively be completed.

### Potential Pathway

-   The BMT1 team has members with skills to update the File input component to meet conditional rendering needs.
-   **Proposal:** Prove out an end-to-end implementation by:
    1.  Forking the `component-library` repo.
    2.  Fixing the `va-file-input-multiple` component locally.
    3.  Hooking up the fixed component locally to the CST.
-   **Potential value:**
    -   Could serve as a reference implementation for the platform team, potentially speeding up resolution.
    -   Might be a candidate for the platform team to merge.
    -   Allows the team to gain experience delivering an end-to-end solution.

---

## Decision Letter Notifications (event-based architecture)

### Assumption(s) to be verified

-   Accountability to continue making progress with the decision letter notifications is currently with BMT 2.

---

## Document Status Feature

-   Seek clarity on what is needed here.
-   Does the “files” tab in staging represent the “document status feature?”
-   If so, what work remains to begin rolling out this feature into production? (assuming it isn’t already)

---

## Load Time Performance Improvement Discovery

-   Need more clarity about specific pain points with load times.
    -   Is it initial claim list load?
    -   Claim detail load? Both?
-   Any previous research / metrics available?
-   **Recommendation:** Make this the first area of focus for BMT1.
    -   Allows us to get up to speed on the user journey through the CST app.
    -   Identify data sources for metrics (Datadog, GA, DOMO).
    -   Bring engineers up to speed on the code base.
-   Consider a 1 week “discovery sprint” where the entire team is focused on producing a readout of findings.
