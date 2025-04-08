# Technical Onboarding: `va-file-input-multiple` in the VA Claims Status Tool

## Table of Contents

- [Background](#background)
- [Progress Summary of Issues & Resolutions](#progress-summary-of-issues--resolutions)
- [Code Architecture of `va-file-input-multiple`](#code-architecture-of-va-file-input-multiple)
  - [Overview](#overview)
  - [Structure](#structure)
  - [Slot Behavior](#slot-behavior)
  - [Props and Attributes](#props-and-attributes)
  - [Events](#events)
  - [Styling and Classes](#styling-and-classes)
  - [Internal Implementation Notes](#internal-implementation-notes)
  - [Mermaid Diagram – File Upload Flow](#mermaid-diagram--file-upload-flow)
- [Integration Points with the Claims Status Tool](#integration-points-with-the-claims-status-tool)
- [Outstanding Issues and Future Development Path](#outstanding-issues-and-future-development-path)
- [Recommendations and Next Steps](#recommendations-and-next-steps)
- [Conclusion](#conclusion)
- [Sources](#sources)
- [Footnotes](#footnotes)

---

## Background

The VA Claims Status Tool (CST) needs to allow Veterans to upload **multiple supporting documents** (evidence files) for a claim. Historically, the CST used an older “V1” file upload component for single files, repeated for each document, to gather required metadata like **document type** and an **encryption password** (for encrypted PDFs). The VA.gov Design System recently introduced a **multi-file upload web component**, `va-file-input-multiple`, to handle multiple attachments within a single UI component ([File input - VA.gov Design System](https://design.va.gov/components/form/file-input#:~:text=This%20guidance%20covers%20two%20web,components)).

The goal is to adopt this new component for consistency and improved UX, while preserving the CST’s requirements (document type for every file, and a password field only for encrypted files). VA.gov tracking issue **`#87835`** (`[CST][ENG] Update File Uploader to use the new va-file-input-multiple component`) was opened to manage this transition and is still in progress ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=This%20was%20referenced%20Nov%2027%2C,2024)).

**Project Goals:** Migrate the CST’s file uploader to use `va-file-input-multiple` (the USWDS v3 Design System component) so that users can add multiple files in one interface. Each file entry must include a “**What type of document is this?**” dropdown, and if a file is detected as an encrypted PDF, a “**PDF password**” input should appear for that file ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=Our%20Use%20Case%3A%20When%20using,not%20supporting%20the%20two%20fields)). The overarching aim is a cleaner implementation that aligns with the design system, rather than maintaining a custom or legacy solution.

**Key Challenges:** Early integration attempts revealed several blockers in `va-file-input-multiple` that prevented the CST team from achieving parity with the old workflow. Notably, the component initially did not support conditionally showing extra fields per file, causing an “all-or-nothing” dilemma with the additional form inputs. The CST team engaged the Platform Design System team to address these issues ([department-of-veterans-affairs/vets-website#33296](https://github.com/department-of-veterans-affairs/vets-website/pull/33296#:~:text=multiple,we%20can%20continue%20this%20work)). This document will walk through those issues, the current implementation status of fixes, and what remains to be done for a successful integration.

---

## Progress Summary of Issues & Resolutions

-   **Nov 2024:** As the CST team began integration, they identified needed enhancements to `va-file-input-multiple`. The Design System team opened issue **`#3551`** to allow **conditional slot rendering** ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=Description)). They noted that slots were appended to all file inputs via `componentDidRender` in a `forEach` loop ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=Currently%20there%20is%20no%20way,loop)). Solutions proposed: a prop or a callback rule ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=The%20two%20possible%20solutions%20I,can%20think%20of%20is)). Linked to CST epic `#87835` ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=This%20was%20referenced%20Nov%2027%2C,2024)).

-   **Late Nov 2024:** Additional bugs logged:
    -   **Event Emissions:** `va-file-input-multiple` emitted its change event twice per upload (tracked in issue `#3549`).
    -   **Duplicate File Selection:** Allowed same file upload repeatedly (issue `#3550`).
    -   Documented in CST integration PR notes ([department-of-veterans-affairs/vets-website#33296](https://github.com/department-of-veterans-affairs/vets-website/pull/33296#:~:text=%2A%20va,documentation%233549)).

-   **Dec 2024:** CST team opened draft PR **`vets-website#33296`** implementing `va-file-input-multiple` and listing blockers. Stated: “Found several bugs while attempting to implement. Spoke with the platform design team, waiting for them to resolve before we can continue this work.” ([department-of-veterans-affairs/vets-website#33296](https://github.com/department-of-veterans-affairs/vets-website/pull/33296#:~:text=multiple,we%20can%20continue%20this%20work)). Development paused.

-   **Late Dec 2024 – Jan 2025:** Design System team delivered fixes:
    -   **Double Event Emission Fixed:** Custom change event (`vaMultipleChange`) now fires once per action ([component-library Releases](https://github.com/department-of-veterans-affairs/component-library/releases#:~:text=%2A%20va,ataker%20%20in%20%20184)). Addressed issue `#3549`.
    -   **Conditional Slots Implemented:** PR **`#1458`** added `slotFieldIndexes` prop (array of file indexes) to control slot rendering ([department-of-veterans-affairs/component-library#1458](https://github.com/department-of-veterans-affairs/component-library/pull/1458#:~:text=Adds%20a%20property%20to%20allow,on%20specific%20file%20input%20fields)). E.g., `slotFieldIndexes="[1,3]"` shows slot content only for the 2nd and 4th files ([component-library PR #1458 Commit](https://github.com/department-of-veterans-affairs/component-library/pull/1458/commits/dbc64251a2ab29facb19d45a352ef8fa8157165f#:~:text=match%20at%20L412%20%3CVaFileInputMultiple%20slot,code)). Default (null/absent) renders for all files ([component-library PR #1458 Commit](https://github.com/department-of-veterans-affairs/component-library/pull/1458/commits/dbc64251a2ab29facb19d45a352ef8fa8157165f#:~:text=it,multiple%3E%3Cspan)).
    -   **Event Payload Expanded (Breaking Change):** PR **`#1460`** modified `vaMultipleChange` event detail structure ([department-of-veterans-affairs/component-library#1460](https://github.com/department-of-veterans-affairs/component-library/pull/1460#:~:text=BREAKING%20CHANGE)). Now emits an object `{ action, file, state }` instead of just an array of file info ([department-of-veterans-affairs/component-library#1460](https://github.com/department-of-veterans-affairs/component-library/pull/1460#:~:text=New%20Data%20Structure%3A), [department-of-veterans-affairs/component-library#1460](https://github.com/department-of-veterans-affairs/component-library/pull/1460#:~:text=Old%20Data%20Structure%3A)).
    -   **Other Improvements:** Added support for showing previously uploaded files (PR `#1461`) and a `statusText` prop (PR `#1463`) ([What’s new? - VA.gov Design System](https://dev-design.va.gov/3852/about/whats-new#:~:text=%2A%20va,input%3A%20add%20statusText%20prop%20by%C2%A0%40powellkerry%C2%A0in%C2%A0%231463)).

-   The above changes were included in **Component Library `v49.0.0` (Jan 2025)** ([component-library Releases](https://github.com/department-of-veterans-affairs/component-library/releases#:~:text=%2A%20va,powellkerry%20%20in%20%20152), [component-library Releases](https://github.com/department-of-veterans-affairs/component-library/releases#:~:text=%2A%20va,powellkerry%20%20in%20%20155)).

-   **Feb 2025:** CST team found **new blocking issue**: `slotFieldIndexes` prop still insufficient. Needed to show *docType* dropdown for **all** files, but *password* input **only for some**. Single slot forces showing both or neither per file index ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=Our%20Use%20Case%3A%20When%20using,not%20supporting%20the%20two%20fields)). Also encountered issues rendering slot content in React initially (prop name confusion: `slot-field-indexes` vs `slotFieldIndexes`) ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=Created%20this%20pr%20that%20I,multiple)). Documented in **`vets-design-system-documentation#3785`** ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=match%20at%20L220%20Our%20Use,was%20written%20makes%20it%20so), [department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=the%20%60va)). CST remains on older solution / "imposter" component ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=We%20are%20haivng%20to%20use,user%20the%20new%20uplaod%20component), [`va.gov-team#104547`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547#:~:text=The%20file%20input%20is%20an,imposter%20component)).

-   **Mar 2025:** Design System team acknowledged limitations of current slot rendering logic. Decided to **refactor/enhance** the component for more flexible per-file custom fields ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=micahchiang%20%20%20commented%20,104)). Proof-of-concept planned for April 2025 ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=caw310%20%20%20commented%20,108)). CST team reiterated conditional slot rendering as the blocker ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=pmclaren19%20%20%20commented%20,109)). Another team (Pre-Need) also blocked, using imposter component ([`va.gov-team#104547`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547#:~:text=MichelleDieudonne%20%20%20commented%20,86)).

-   **Current Status (April 2025):** `va-file-input-multiple` is available with initial fixes, but insufficient for CST's conditional slot needs. Issue `#3785` remains open pending Platform team solution. CST integration epic `#87835` is blocked. CST uses legacy/imposter component in production ([`va.gov-team#104547`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547#:~:text=The%20file%20input%20is%20an,imposter%20component), [`va.gov-team#104547`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547#:~:text=We%20are%20currently%20experiencing%20this,for%20work%20as%20capacity%20allows)).

---

## Code Architecture of `va-file-input-multiple`

### Overview

`va-file-input-multiple` is a Web Component (built with the VA.gov Component Library) intended to handle multi-file uploads one file at a time (in line with USWDS guidelines). It encapsulates the markup, state, and behavior for selecting files and displaying a list of selected files. It is a **“multiple files” variation** of the standard file input ([File input - VA.gov Design System](https://design.va.gov/components/form/file-input#:~:text=This%20guidance%20covers%20two%20web,components)).

> [!NOTE]
> Unlike a typical `<input type="file" multiple>`, VA’s implementation explicitly **does not allow selecting multiple files at once** due to usability considerations (some users don’t know how to multi-select, and iOS doesn’t support it) ([File input - VA.gov Design System](https://design.va.gov/components/form/file-input#:~:text=,files%20in%20a%20file%20browser)). Users add files one by one.

### Structure

Visually and in the DOM, `va-file-input-multiple` consists of:
-   An outer container with optional header/label and hint text.
-   A **file selection control** (button like “Upload file” or “Add another file”) tied to an `<input type="file">`.
-   A **list of file item entries** (“file cards”) displayed for each selected file. Each card shows:
    -   File name, size, optional icon.
    -   **Change File** action.
    -   **Delete** action.
    -   Error message (if applicable, shown within the card) ([File input - VA.gov Design System](https://design.va.gov/components/form/file-input#:~:text=file%20input%20area,property)).
-   **Slotted content (Additional info slot):** Developer-provided markup injected into each file card after selection.

![Example file upload card with dropdown](https://design.va.gov/images/components/file-input/file-input-multiple-additional-info.png)
*Figure 1: Example of a file upload card with an additional form input (document type dropdown) shown below the file info.* ([File input - VA.gov Design System](https://design.va.gov/components/form/file-input))

### Slot Behavior

-   Content placed inside `<va-file-input-multiple>…</va-file-input-multiple>` is treated as **additional form inputs** for each file.
-   When a file card is rendered, the component appends a clone of the slotted content into that card's UI ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=Currently%20there%20is%20no%20way,loop)).
-   This happens in `componentDidRender` by iterating over file items and injecting slot nodes ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=Currently%20there%20is%20no%20way,loop)).
-   **Default:** Slot content renders for **all** file items.
-   **Conditional Rendering (v49.0.0+):** The **`slotFieldIndexes`** property (array of 0-based indices) controls which file cards receive the slot content ([department-of-veterans-affairs/component-library#1458](https://github.com/department-of-veterans-affairs/component-library/pull/1458#:~:text=Adds%20a%20property%20to%20allow,on%20specific%20file%20input%20fields)).
    -   If `slotFieldIndexes` is null/absent, slot renders for all items ([component-library PR #1458 Commit](https://github.com/department-of-veterans-affairs/component-library/pull/1458/commits/dbc64251a2ab29facb19d45a352ef8fa8157165f#:~:text=it,multiple%3E%3Cspan)).
    -   If `slotFieldIndexes="[1]"`, only the second file card gets the slot content.
-   **Limitation:** The *same* slot content is cloned for each specified file. It doesn't support different content per file or conditionally showing *parts* of the slot content. This is the current blocker for CST ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=Our%20Use%20Case%3A%20When%20using,not%20supporting%20the%20two%20fields)).

### Props and Attributes

Key configuration properties:
-   `label` (string): Label text.
-   `buttonText` (string): Text on the upload button.
-   `accept` (string): Allowed file types (e.g., `".pdf,.jpg"`).
-   `name` (string): Name attribute for underlying inputs.
-   `error` (string): Error message to display.
-   `required` (boolean): Indicates if at least one file is required.
-   `headerText` / `headerSize`: Render label as a heading ([File input - VA.gov Design System](https://design.va.gov/components/form/file-input#:~:text=Header%20label)).
-   `enableAnalytics` (boolean): Fire analytics events.
-   `slotFieldIndexes` (array<number>): Indices of files to render slot content for.

### Events

-   **`vaMultipleChange`**: Primary custom event fired when the file list changes (add, remove, replace).
-   **Event Detail (v49.0.0+):**
    ```javascript
    // Example structure
    detail = {
      action: "FILE_ADDED" | "FILE_REMOVED" | "FILE_REPLACED", // Likely values
      file: File,      // File object involved
      index: number,   // Index related to the action
      state: [         // Array of all current files in the list
        { file: File, changed: boolean },
        // ... more files
      ]
    }
    ```
    (See [department-of-veterans-affairs/component-library#1460](https://github.com/department-of-veterans-affairs/component-library/pull/1460#:~:text=Old%20Data%20Structure%3A) for details).
-   Parent application listens via `onVaMultipleChange` (JSX) or `addEventListener` ([Storybook Example](https://design.va.gov/storybook/?path=/story/uswds-va-file-input-multiple--custom-validation#:~:text=Components%20%2F%20File%20input%20multiple,files%2C%20and%20dynamically%20set%20errors)).

### Styling and Classes

-   Uses USWDS 3.0 styling via VA design system.
-   Standard margins (e.g., `vads-u-margin-bottom--3`).
-   File cards styled as bordered boxes with icons.
-   Styles encapsulated in shadow DOM.

### Internal Implementation Notes

-   Manages an internal array (state) of files.
-   Slot injection is done programmatically in `componentDidRender` by cloning provided slot nodes into specified file card containers ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=Currently%20there%20is%20no%20way,loop)).
-   Limitation: Cannot pass unique data per file into the slot content ([department-of-veterans-affairs/vets-design-system-documentation#3551](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551#:~:text=Currently%20there%20is%20no%20way,loop)).

### Mermaid Diagram – File Upload Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as va-file-input-multiple
    participant App as Claims Status Tool (App)

    U->>C: Choose a file (via file picker)
    C->>C: Add new file to internal list and render file card
    C->>C: Attach slot content to new file card (if applicable via slotFieldIndexes)
    C-->>App: Emit **vaMultipleChange** event with updated file list state
    App->>App: Handle event: Update Redux state with new file (incl. docType, encrypted flag)
    opt Conditional Slot Update
        App->>App: Determine which file indices need password field
        App->>C: Update **slotFieldIndexes** prop based on encrypted flags
        C->>C: Re-render file cards; attach slot content only to specified file(s)
    end
```
*Diagram: Sequence illustrating user interaction, component events, application state updates, and conditional slot rendering based on application logic.*

---

## Integration Points with the Claims Status Tool

The `va-file-input-multiple` will be integrated into the **Claims Status Tool’s “Files” or “Evidence” section**.

![Current CST file upload UI](https://user-images.githubusercontent.com/90410014/221887001-d550a8f8-10a8-44cf-83ac-91797d6a276f.png)
*Figure 2: Current CST file upload UI (legacy implementation) – showing per-file doc type dropdown and conditional password field.* ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785))

Integration steps and considerations:

-   **Invoking the Component:** Replace custom/legacy upload fields with `<va-file-input-multiple>` in React JSX. Provide necessary props (`label`, `accept`, etc.) and slot content (doc type dropdown, password input).
    ```jsx
    <va-file-input-multiple
        label="Upload additional evidence"
        button-text="Upload file"
        accept=".pdf,.jpg,.png,.docx"
        onVaMultipleChange={this.handleFilesChanged}
        slotFieldIndexes={this.state.passwordRequiredIndexes}
    >
      {/* Slot content: doc type dropdown and password input */}
      <div className="additional-inputs">
        <va-select label="What type of document is this?" required>
           {/* options */}
        </va-select>
        <va-text-input label="PDF password" type="password" required></va-text-input>
      </div>
    </va-file-input-multiple>
    ```

-   **Capturing Events:** Handle the `vaMultipleChange` event (likely via `onVaMultipleChange` prop) to update application state.

-   **Managing Application State:** Maintain a Redux/React state array for uploaded files, storing the `File` object plus custom metadata: `docType`, `isEncrypted`, `password` ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=Things%20to%20NOTE%3A%20we%20take,as%20well)). Sync this state on every `vaMultipleChange` event.

-   **Determining Encryption:** Application logic needed to identify encrypted PDFs (backend check, client-side check, or user input) to set the `isEncrypted` flag in state, which then drives the `slotFieldIndexes` prop.

-   **Two-field Slot Strategy:** Current component limitation requires placing both fields in one slot. A workaround might be needed until the component is enhanced (e.g., always show slot and hide password field via CSS/logic, or temporarily omit doc type for non-password files – neither ideal). The blocker remains needing finer control over slot content visibility.

-   **Current Workaround in Integration:** CST likely uses a legacy or "imposter" component in production/staging until the official component meets requirements ([`va.gov-team#104547`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547#:~:text=The%20file%20input%20is%20an,imposter%20component)).

-   **Upload Logic:** The component handles client-side selection only. Actual file upload to the server (via Lighthouse API) is triggered by a separate "Submit" button, using the files stored in application state.

-   **Validation:** Application must validate that required metadata (doc type for all, password if encrypted) is provided before submission. Error messages might be shown via the component's `error` prop or alongside individual fields.

-   **Upstream/Platform Constraints:** CST team cannot modify the web component directly; enhancements must come from the Platform Design System team via the `component-library`.

-   **Redux and Form Integration:** Integrate within a container component connected to Redux for state management. Capture user input from slotted fields (doc type, password) to update Redux state.

---

## Outstanding Issues and Future Development Path

Primary blocker: **Support for conditional slot content per file item** (`#3785`). Current `slotFieldIndexes` is insufficient for CST's needs.

Potential solutions being explored by Platform team:
-   **Multiple Named Slots or Template Per Item:** Allow different slots for always-shown vs. conditional content.
-   **Built-in Support for Common Use Cases:** Add component props like `encrypted` to handle password fields internally (less flexible).
-   **Refactoring Component Structure:** Treat each file card as a configurable sub-component.

Timeline: Platform team planned proof-of-concept in April 2025, aiming for a solution definition soon after ([department-of-veterans-affairs/vets-design-system-documentation#3785](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785#:~:text=caw310%20%20%20commented%20,108)).

Secondary issues to verify:
-   Handling of duplicate file uploads (`#3550`).
-   Accessibility testing of the integrated component with custom slots.

Upstream Coordination: CST team needs to monitor `#3785`, test pre-release versions, and provide feedback.

---

## Recommendations and Next Steps

**For the New Engineer on CST:**

1.  **Familiarize with Current CST Upload:** Review existing code (`applications/claims-status/`) to understand legacy implementation and state management.
2.  **Upgrade Component Library:** Ensure `vets-website` uses `@department-of-veterans-affairs/component-library` v49.x or later.
3.  **Implement Component (behind toggle):** Integrate `<va-file-input-multiple>` with slotted fields. Handle `vaMultipleChange` event to sync Redux state. Calculate `slotFieldIndexes` based on `isEncrypted` flag in state.
4.  **Test Basic Flows:** Add/remove/change files (regular and encrypted). Verify events and state updates. Test duplicate file handling.
5.  **Address Known Bugs/Workarounds:** Document behavior with current slot limitations. Implement temporary workaround if feasible and approved by Design (e.g., always show slot, hide password via CSS). Prefer waiting for official fix.
6.  **Stay Coordinated with Platform Team:** Monitor `#3785`. Test pre-release solutions from Platform. Provide feedback.
7.  **Testing & QA:** Perform end-to-end, regression, accessibility (screen reader, keyboard nav), and browser/mobile testing once integrated.
8.  **Rollout Plan:** Use feature flags for phased rollout (staging -> production). Monitor analytics and error logs.
9.  **Cleanup:** Remove legacy/imposter code after successful deployment. Update documentation.

---

## Conclusion

Integrating `va-file-input-multiple` will align the CST with VA.gov Design System standards, improving maintainability. The primary blocker is the component's current limitation in handling conditional per-file slot content. Resolution is pending from the Platform Design System team. The new engineer should monitor progress on issue `#3785`, prepare the integration based on the current component version, and be ready to adapt to the forthcoming solution. Successful integration will require careful state management, event handling, testing, and close collaboration with the Platform team.

---

## Sources

-   [VA.gov Design System: File Input](https://design.va.gov/components/form/file-input) - Component documentation.
-   [VA.gov Component Library Releases](https://github.com/department-of-veterans-affairs/component-library/releases) - Release notes for fixes/features.
-   GitHub Issues & PRs:
    -   [`vets-design-system-documentation#3551`](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3551) - Initial request for conditional slots.
    -   [`vets-design-system-documentation#3785`](https://github.com/department-of-veterans-affairs/vets-design-system-documentation/issues/3785) - Current blocking issue for CST.
    -   [`component-library#1458`](https://github.com/department-of-veterans-affairs/component-library/pull/1458) - PR implementing `slotFieldIndexes`.
    -   [`component-library#1460`](https://github.com/department-of-veterans-affairs/component-library/pull/1460) - PR with breaking change to `vaMultipleChange` event.
    -   [`vets-website#33296`](https://github.com/department-of-veterans-affairs/vets-website/pull/33296) - CST team's draft integration PR.
    -   [`va.gov-team#104547`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/104547) - Staging review finding on "imposter" component.
    -   [`vets-website#34887`](https://github.com/department-of-veterans-affairs/vets-website/pull/34887) - Accessibility fix for focus management.
-   [Storybook Example - Custom Validation](https://design.va.gov/storybook/?path=/story/uswds-va-file-input-multiple--custom-validation) - Shows event handling example.
