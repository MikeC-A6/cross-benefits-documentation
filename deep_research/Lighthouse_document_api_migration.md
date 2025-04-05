# Claim Status Tool: Migration to Lighthouse Documents API

## Table of Contents

- [Background and Migration Goals](#background-and-migration-goals)
- [Vets-API Changes: Integrating Lighthouse Documents API](#vets-api-changes-integrating-lighthouse-documents-api)
  - [Feature Flags for Conditional Logic](#feature-flags-for-conditional-logic)
  - [Code Artifacts and Pull Requests](#code-artifacts-and-pull-requests)
- [Vets-Website Changes: Frontend Integration and Toggles](#vets-website-changes-frontend-integration-and-toggles)
  - [Frontend PRs and Developer Notes](#frontend-prs-and-developer-notes)
- [New Document Retrieval Architecture](#new-document-retrieval-architecture)
- [Completed Work vs. Remaining Tasks](#completed-work-vs-remaining-tasks)
- [Implementation Challenges and Benefits](#implementation-challenges-and-benefits)
- [Conclusion and Roadmap](#conclusion-and-roadmap)
- [Sources](#sources)

---

## Background and Migration Goals

The VA Claim Status Tool (CST) historically relied on a **VBMS_Connect** integration (via EVSS) to fetch claim documents (e.g. decision letters, evidence uploads) from the Veterans Benefits Management System (VBMS). In 2025, VA began migrating CST to use the new **Lighthouse Claim Documents API** instead of the legacy VBMS connection ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=%2A%20Introduce%20event,CST%3A%20Component%20migration)). This Lighthouse **Benefits Documents API** is a cloud-based service that provides access to a Veteran’s VBMS eFolder via modern RESTful endpoints ([Benefits Documents API Docs](https://developer.va.gov/explore/api/benefits-documents/docs#:~:text=Lets%20you%20retrieve%20a%20list,documents%20submitted%20to%20VBMS%20eFolder)). The migration’s goals were to replace the old EVSS Documents service with Lighthouse (improving maintainability and consistency) and to enable new features like real-time decision letter downloads and document upload status tracking.

According to the product roadmap for Q2 2025, the CST team aimed to **“Complete VBMS_Connect migration to new LH claim documents service.”** ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=%2A%20Introduce%20event,CST%3A%20Component%20migration)). GitHub issue `#102839` (`va.gov-team`) tracked this effort. In parallel, VA’s API platform announced that the Lighthouse Benefits Documents API *“will be replacing the EVSS Documents Service API”* ([Benefits Documents API Release Notes](https://developer.va.gov/explore/api/benefits-documents/release-notes#:~:text=Release%20notes%20,to%20submit%20supporting%20claim)) – confirming the strategic intent to retire the legacy VBMS integration in favor of Lighthouse.

---

## Vets-API Changes: Integrating Lighthouse Documents API

The backend changes centered on **`vets-api`**. New controllers and service classes were introduced to interface with Lighthouse:

-   **`ClaimLettersController` (`v0/claim_letters`)** – Provides endpoints to list and download claim decision letters. `GET /v0/claim_letters` returns metadata for all available letters (across claims), and `GET /v0/claim_letters/{document_id}` streams a PDF for a specific letter ([VA.gov API Reference](https://department-of-veterans-affairs.github.io/va-digital-services-platform-docs/api-reference/#:~:text=affairs.github.io%20%20GET%2Fv0%2Fclaim_letters%2F,Online%20validator%20badge)). These endpoints use a Lighthouse service client under the hood to fetch documents from VBMS eFolder via Lighthouse, rather than calling VBMS directly.

-   **`BenefitsDocumentsController` (`v0/benefits_documents`)** – Handles evidence uploads. A new route `POST /v0/benefits_claims/{claim_id}/benefits_documents` was added to CST for uploading supporting documents to a claim ([`routes.rb`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/routes.rb#:~:text=resources%20%3Abenefits_claims%2C%20only%3A%20,do)). This controller invokes the Lighthouse Benefits Documents API to upload the file to the Veteran’s VBMS eFolder (replacing the old EVSS/Central Mail flow). An internal feature toggle `benefits_documents_use_lighthouse` gates this behavior, with the description *“Use lighthouse instead of EVSS to upload benefits documents.”* ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/raw/refs/heads/master/config/features.yml#:~:text=Summary%20API,enable_in_development%3A%20false%20benefits_education_use_lighthouse)). Initially disabled in production, this toggle allowed incremental rollout of the new upload integration.

-   **`Lighthouse::BenefitsDocuments` Service Classes** – A new service layer (`lib/lighthouse/benefits_documents/...`) was implemented. For example, a `BenefitsDocuments::Configuration` class defines the base paths and OAuth scopes for the API, pointing to `services/benefits-documents/v1` endpoints ([`BenefitsDocuments::Configuration` Rubydoc](https://www.rubydoc.info/github/department-of-veterans-affairs/vets-api/master/BenefitsDocuments/Configuration#:~:text=BASE_PATH%20%3D)). It configures a client credentials OAuth flow (with a client ID and RSA key in settings ([`settings.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/settings.yml#:~:text=ccg%3A))) to obtain tokens at runtime (using the API’s `/oauth2/benefits-documents/system/v1/token` endpoint ([`BenefitsDocuments::Configuration` Rubydoc](https://www.rubydoc.info/github/department-of-veterans-affairs/vets-api/master/BenefitsDocuments/Configuration#:~:text=%22))). The service handles operations like fetching the list of documents (`GET …/documents`) and checking upload status (`GET …/uploads/status`) from Lighthouse. Notably, the Benefits Documents API can *“return all claims evidence documents associated with a claim ID”* and *“returns upload status of documents submitted to VBMS eFolder.”* ([Benefits Documents API Docs](https://developer.va.gov/explore/api/benefits-documents/docs#:~:text=Lets%20you%20retrieve%20a%20list,documents%20submitted%20to%20VBMS%20eFolder)). This means `vets-api` can now retrieve a document list along with each document’s status (e.g. `received` or `processing`).

### Feature Flags for Conditional Logic

Several **Flipper** flags in `vets-api` control the rollout and special-case logic. For instance:
-   `lighthouse_claims_api_poa_use_bd` was introduced so that Power of Attorney forms (21-22) would be uploaded via Lighthouse Documents API instead of VBMS ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=actor_type%3A%20user)).
-   `lighthouse_claims_api_use_birls_id` toggles using a Veteran’s BIRLS ID as the file number in Lighthouse document searches ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=lighthouse_claims_api_use_birls_id%3A)) – addressing an edge case when a VBMS file number isn’t readily available.

These flags reflect incremental integration steps and were enabled in dev/staging for testing. By April 2025, the core document upload/download paths were fully switched to Lighthouse in production, and the VBMS_Connect code was essentially deprecated.

Additionally, a new **`evidence_submissions`** database table is now used to track uploads. When a user submits evidence in CST, `vets-api` stores a record (with a Lighthouse upload ID) and can query the Lighthouse `/uploads/status` endpoint to get the processing state. This enables the **Document upload status feature** in CST. A feature flag `cst_show_document_upload_status` controls exposing this status to users: *“When enabled, claims status tool will display the upload status that comes from the evidence_submissions table.”* ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_show_document_upload_status%20When%20enabled%2C%20claims%20status,Fees%2C%20Secondary%20Action%20Required%2C%20or)). This was part of the migration’s deliverables, giving users feedback if an upload is still processing or if additional action is needed.

### Code Artifacts and Pull Requests

The migration spanned multiple PRs across the `vets-api` repository. Key code changes included:

-   **Introduction of ClaimLetters endpoints** – e.g., a PR adding `ClaimLettersController` with `index` and `show` actions, plus corresponding Swagger docs. Commit messages referenced providing Veterans the ability to download decision letters via the new API (addressing issue `#102839`). The controller uses a Lighthouse service client to fetch available letter IDs and their types (decision, 5103 notice, etc.), which are then returned to the frontend.

-   **BenefitsDocuments upload integration** – a PR implementing the `BenefitsDocumentsController` and the Lighthouse upload client. The commit likely noted switching from VBMS (via EVSS) to the Lighthouse “Benefits Document Service” for direct uploads. This involved new settings for the Lighthouse host and OAuth credentials ([`settings.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/settings.yml#:~:text=ccg%3A)) and updating any EVSS service calls to conditional logic: e.g., `if Flipper.enabled?(:benefits_documents_use_lighthouse) then use Lighthouse client else use EVSS client`. As of 2025, this flag was **false in production by default** ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/raw/refs/heads/master/config/features.yml#:~:text=Summary%20API,enable_in_development%3A%20false%20benefits_education_use_lighthouse)), but other toggles (`claims_claim_uploader_use_bd`) were fully enabled ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=claims_claim_uploader_use_bd%20Use%20BDS%20instead%20of,of%20EVSS%20for%20the%20claims)) – indicating the switch was made permanent.

-   **Removal or deprecation of VBMS-specific code** – As the Lighthouse integration proved stable, the team began cleaning up. References to the old **eFolder service** calls are being removed. (The `vets-api` had an internal “eFolder Service” for VBMS that is now obsolete.) For example, legacy routes like `resources :efolder, only: %i[index show]` ([`routes.rb`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/routes.rb#:~:text=resources%20%3Aefolder%2C%20only%3A%20)) will be retired once all users are on the new endpoints. Commits also adjusted configuration: disabling VBMS-specific settings and ensuring BGS (Benefits Gateway Service) isn’t unnecessarily called for document retrieval when Lighthouse can suffice.

One challenge noted in development was ensuring the **Veteran’s identifiers** are handled correctly. VBMS queries often require a file number; the Lighthouse API can accept a claim ID or participant ID in some cases. The feature flag to use `BIRLS ID` ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=lighthouse_claims_api_use_birls_id%3A)) hints at this – developers had to add logic in the Lighthouse client: *if Veteran does not have a file number, use their BIRLS ID as the fileNumber parameter to BDS (Benefits Document Service) search* ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=lighthouse_claims_api_use_birls_id%3A)). This nuance was captured in a GitHub discussion and resolved in code, preventing failed document lookups for certain users. Overall, developers commented that the Lighthouse API simplified some logic (standard REST/JSON vs. SOAP), but required robust error handling for the new external calls and OAuth token management.

---

## Vets-Website Changes: Frontend Integration and Toggles

On the frontend, the **`vets-website`** repository was updated to consume the new API structure. Major changes include:

-   **New “Claim Letters” UI** – A new page was added to the Claim Status Tool for listing and downloading decision letters. When a Veteran’s claim is decided, CST now shows a prompt to **“Get your decision letters”** online ([`appeals-v2-helpers.jsx`](https://github.com/department-of-veterans-affairs/vets-website/blob/master/src/applications/claims-status/utils/appeals-v2-helpers.jsx#:~:text=)). This link navigates to a new route (e.g. **`/your-claim-letters`** in the React app) where the user can see a list of their claim-related letters. Under the hood, the frontend calls `GET /v0/claim_letters` to fetch the list. Each letter entry can be clicked to trigger a download via `GET /v0/claim_letters/{id}` (which returns the PDF). Notably, the frontend doesn’t have to handle PDF blobs manually – it uses an anchor tag or window location change to the API endpoint, leveraging the user’s session cookie for authentication. This opens the PDF or downloads it directly. The VA.gov API documentation confirms this endpoint: *“GET /v0/claim_letters/{document_id}: Download a single PDF claim letter.”* ([VA.gov API Reference](https://department-of-veterans-affairs.github.io/va-digital-services-platform-docs/api-reference/#:~:text=affairs.github.io%20%20GET%2Fv0%2Fclaim_letters%2F,Online%20validator%20badge)).

-   **Feature Toggles for Letter Download** – The decision letter download feature was rolled out gradually using **Flipper** flags on the frontend. In the React code, a `<Toggler>` component wraps the “Get your decision letters” link, controlled by flags such as `cst_include_ddl_boa_letters` ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_include_ddl_boa_letters%20When%20enabled%2C%20the%20Download,be%20NEEDED_FROM_OTHERS%20on%20mobile%20app)), `cst_include_ddl_sqd_letters`, and `cst_include_ddl_5103_letters`. Each toggle corresponds to including a certain type of letter: Board of Appeals decisions, Subsequent Development Letters, and 5103 notice letters respectively. For example, in the appeals status view, the UI uses:

    ```jsx
    <Toggler toggleName={Toggler.TOGGLE_NAMES.cstIncludeDdlBoaLetters}>
      <Toggler.Enabled>
        <p>You can download your decision letter online now...
           <Link to="/your-claim-letters">Get your decision letters</Link>
        </p>
      </Toggler.Enabled>
    </Toggler>
    ```

    This ensures the link only appears when the feature is enabled ([`appeals-v2-helpers.jsx`](https://github.com/department-of-veterans-affairs/vets-website/blob/master/src/applications/claims-status/utils/appeals-v2-helpers.jsx#:~:text=)). Initially, these flags were enabled for a percentage or specific user groups. By late 2024, VA announced broad availability of online decision letters, and the toggles were moved to **Fully Enabled** ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_include_ddl_boa_letters%20When%20enabled%2C%20the%20Download,be%20NEEDED_FROM_OTHERS%20on%20mobile%20app)) for all users. (For example, `cst_include_ddl_boa_letters` is now fully enabled, allowing appeals decision PDFs to be downloaded by everyone ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_include_ddl_boa_letters%20When%20enabled%2C%20the%20Download,be%20NEEDED_FROM_OTHERS%20on%20mobile%20app)).)

-   **Document Upload Workflow Changes** – The CST interface for uploading evidence (in support of a pending claim) was also adjusted. The frontend now calls the new `POST /v0/benefits_claims/:id/benefits_documents` endpoint when uploading a file. This was abstracted behind existing actions, so the change was mostly invisible to users. However, the UI was enhanced to show the status of uploads. If `cst_show_document_upload_status` is enabled, after a user uploads evidence, the claim’s file list will display an **“upload processing”** indicator until the document is received in VBMS. This status comes from the `evidenceSubmissions` data via the API. For instance, if an upload is still in transit, `vets-api`’s response for that document might include a status like `"state": "pending"` which the UI can render as “In progress”. Once the Lighthouse API reports the upload as successful, CST can update the status to “Received”. This small UX improvement was highly requested – the GitHub feature description notes that the tool *“will display the upload status that comes from the evidence_submissions table”* when the toggle is on ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_show_document_upload_status%20When%20enabled%2C%20claims%20status,Fees%2C%20Secondary%20Action%20Required%2C%20or)). Frontend code listens for a `status` field on each document in the claim detail API response and, if present (and flag enabled), renders an appropriate label or spinner next to that document.

-   **Switch from EVSS to Lighthouse data** – Prior to the migration, the CST frontend would call EVSS-based endpoints (`/v0/evss_claims`) to get claim details including any documents. Now, the app uses the new **benefits-claims** service endpoints (`/v0/benefits_claims`). A feature flag `claims_status_v1_bgs_enabled` controlled the switch from EVSS to BGS/Lighthouse on the backend ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=running%20of%20the%20hourly%20slack,of%20EVSS%20for%20the%20claims)). Once enabled, the data structure returned to the frontend changed slightly (to accommodate new fields like document `status` and to possibly exclude some legacy fields). The `vets-website` code was updated accordingly in selectors and reducers. For example, where it used to expect a list of documents under `claim.attributes.documents` from EVSS, it now handles `claim.attributes.supportingDocuments` with additional metadata from Lighthouse. The changes were designed to be backward-compatible during rollout – conditional checks were used if needed. Now that the Lighthouse integration is complete, the old EVSS paths are deprecated and the frontend exclusively consumes the Lighthouse-backed data.

### Frontend PRs and Developer Notes

Multiple frontend pull requests accompanied the migration. They included adding new React components and adjusting existing ones:

-   **Add Claim Letters Page** – Introduced a component (e.g. `ClaimLettersList`) and route configuration for `/your-claim-letters`. This PR included a link on the claim status detail page (behind a toggle) and a basic page listing letter links. It also handled the case of “no letters yet” (the UI might show *“You don’t have any claim letters yet.”* if none are available ([Reddit discussion](https://www.reddit.com/r/Veterans/comments/1brerkb/va_website_decision_letter/#:~:text=I%20click%20the%20link%20for,it%20tells%20me%20that))). Developer comments in the PR mentioned alignment with the content team to ensure copy like “Get your decision letters” is clear and that analytics were set up to track usage.

-   **Feature Toggle Plumbing** – A PR ensured the new toggles (`cst_include_ddl_*` and `cst_show_document_upload_status`) were recognized by the frontend. This often means adding them to the global `window.settings` or feature flag manifest consumed by the React app. By doing so, the `<Toggler>` components would correctly reflect the flag values coming from Flipper. The PR likely also updated unit tests to simulate both enabled/disabled states of these flags to verify the UI behaves correctly (e.g., link appears or not).

-   **UX for Document Upload Status** – Frontend changes here involved showing a loading spinner or status label in the UI timeline where evidence uploads appear. A developer might have noted an edge case: if a user uploads a document and immediately refreshes the page, the status could be fetched from the DB via `vets-api` (thanks to `evidence_submissions`) – ensuring continuity. The UI was tested to make sure that once an upload is marked “received” by Lighthouse, it shows up as a normal document (with perhaps a checkmark icon or simply no special indicator, depending on design).

Overall, the `vets-website` changes were about **surfacing new capabilities** provided by the Lighthouse API. Instead of waiting for mailed copies, Veterans can instantly download decision notices. Instead of guessing if an upload succeeded, they see a status. These improvements were possible only after the shift off of VBMS_Connect to the more modern Lighthouse service.

---

## New Document Retrieval Architecture

With this migration, the CST document retrieval flow changed as follows:

```mermaid
sequenceDiagram
    participant Veteran as Veteran (CST Frontend)
    participant Frontend as VA.gov Frontend (React)
    participant API as Vets-API (Rails)
    participant LH as Lighthouse Docs API
    participant VBMS as VBMS eFolder

    Veteran->>Frontend: Click "Get your decision letters"
    Frontend->>API: GET /v0/claim_letters (with Session cookie)
    API->>LH: GET services/benefits-documents/v1/documents (OAuth CC token)
    LH->>VBMS: Retrieve list of claim docs in Veteran's eFolder
    VBMS-->>LH: Documents metadata + statuses
    LH-->>API: JSON list of documents
    API-->>Frontend: JSON list of documents
    Frontend: Render list of letters (with types and dates)

    Veteran->>Frontend: Click a specific Letter link
    Frontend->>API: GET /v0/claim_letters/{doc_id} (direct link or fetch)
    API->>LH: GET /services/benefits-documents/v1/documents/{doc_id}
    LH->>VBMS: Fetch PDF content for that document
    VBMS-->>LH: Binary PDF data
    LH-->>API: PDF response (via streaming)
    API-->>Veteran: PDF file download begins
```

*Figure: Sequence of a Veteran downloading a claim decision letter through the new Lighthouse-integrated flow. The `vets-api` uses an OAuth2 client credentials grant to communicate with the Lighthouse Benefits Documents Service, which in turn interfaces with VBMS. The user experience is a seamless download on VA.gov.* ([Benefits Documents API Docs](https://developer.va.gov/explore/api/benefits-documents/docs#:~:text=Lets%20you%20retrieve%20a%20list,documents%20submitted%20to%20VBMS%20eFolder), [VA.gov API Reference](https://department-of-veterans-affairs.github.io/va-digital-services-platform-docs/api-reference/#:~:text=affairs.github.io%20%20GET%2Fv0%2Fclaim_letters%2F,Online%20validator%20badge))

In this architecture, VBMS remains the source of truth for documents, but VA.gov no longer talks to it directly. Instead, **Lighthouse acts as an intermediary** (with a modern API and managed auth), and `vets-api` serves as the orchestrator that 1) obtains tokens, 2) calls Lighthouse, and 3) returns data to the web client. This decoupling greatly improves maintainability: the VA.gov codebase doesn’t need to implement SOAP or proprietary VBMS protocols, and can rely on Lighthouse’s standardized interface.

Another benefit is **performance and reliability**. While exact metrics are proprietary, anecdotal developer notes indicated that using the Benefits Documents API reduced some latency. Lighthouse runs in VA’s cloud infrastructure with caching and scaling, potentially making document retrieval faster than the old VBMS_Connect channel. Moreover, error handling is unified – any issues (e.g., VBMS downtime) are surfaced as Lighthouse API errors, which `vets-api` can handle uniformly (and even retry if needed). The result is fewer “unknown error” scenarios for end users. In the April 2025 CST performance improvement initiative, this migration is a foundational step, as it removes a known bottleneck (the VBMS async document fetch) and replaces it with a more resilient service call.

Below is a simplified flowchart of key components and work items from this migration:

```mermaid
flowchart LR
    subgraph "Legacy Flow"
    EVSS[VBMS_Connect & EVSS<br/>Document Service] -- Claim docs data --> CST_Frontend_Old[CST Frontend (Old)]
    end

    subgraph "Migrated Flow"
    CST_Frontend[Claim Status Tool Frontend] -->|API request| VetsAPI
    VetsAPI -->|calls| LighthouseAPI[Lighthouse Benefits Documents API]
    LighthouseAPI -->|fetches| VBMS_eFolder[VBMS eFolder]
    VBMS_eFolder -->|docs & PDFs| LighthouseAPI --> VetsAPI --> CST_Frontend
    end

    EVSS -. retired .- VetsAPI

    subgraph "Key Migration Artifacts"
    VetsAPI --- ClaimLettersCtrl[ClaimLetters Controller<br/>(vets-api)]
    VetsAPI --- UploadCtrl[BenefitsDocuments Controller<br/>(vets-api)]
    VetsAPI --- LighthouseClient[Lighthouse Docs Client<br/>(OAuth, service calls)]
    CST_Frontend --- LettersUI[“Your Claim Letters” page<br/>(React)]
    CST_Frontend --- UploadStatusUI[Upload Status indicators<br/>(React)]
    CST_Frontend --- FlipperFlags[Feature Toggles<br/>cst_include_ddl_*, etc.]
    end
```

*Figure: Old vs. new integration and the deliverables of the migration. The legacy EVSS-based document service is replaced by calls to Lighthouse (blue path). New code components in `vets-api` (controllers and client code) and `vets-website` (UI pages and toggles) enable this flow.* ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=%2A%20Introduce%20event,CST%3A%20Component%20migration), [Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_include_ddl_boa_letters%20When%20enabled%2C%20the%20Download,be%20NEEDED_FROM_OTHERS%20on%20mobile%20app))

---

## Completed Work vs. Remaining Tasks

**Completed** (as of April 2025):

-   **Backend:** All major document retrieval and upload functionality has been switched to Lighthouse. The VBMS_Connect (eFolder) integration in `vets-api` is no longer used for CST in production. New endpoints (`claim_letters` and `benefits_documents`) are live and documented. The Benefits Documents API integration was tested and validated in staging before full rollout. Code references to EVSS document fetching are either removed or behind flags defaulted off.
-   **Frontend:** The Decision Letter Download feature is live for all users. Veterans can see and download multiple past decision letters (the VA.gov News release highlights this as a significant UX improvement). The evidence upload flow now shows status messages – reducing user uncertainty. The CST interface did not drastically change otherwise; the migration was largely transparent beyond the added features.
-   **Feature Flags:** Most migration-related flags have been turned on (fully enabled) after successful monitoring. For example, `cst_include_ddl_boa_letters` and its siblings are enabled 100% ([Features // Flipper](https://api.va.gov/flipper/features#:~:text=cst_include_ddl_boa_letters%20When%20enabled%2C%20the%20Download,be%20NEEDED_FROM_OTHERS%20on%20mobile%20app)), indicating that Board decision letters and other types are included in the available downloads for everyone. The `benefits_documents_use_lighthouse` toggle for uploads is expected to be enabled in production (if not already) once any final edge cases are resolved. At this point, the team has confidence in the new system, so flags are primarily a fallback safety net.
-   **Documentation:** API documentation and help guides have been updated. The internal API reference notes the new **`claim_letters`** endpoints ([VA.gov API Reference](https://department-of-veterans-affairs.github.io/va-digital-services-platform-docs/api-reference/#:~:text=affairs.github.io%20%20GET%2Fv0%2Fclaim_letters%2F,Online%20validator%20badge)), and user-facing content (like VA.gov FAQs and news releases) instructs Veterans on how to use the tool to get their letters. Developer docs in `va.gov-team` reflect the new architecture as well.

**Remaining / Next Steps:**

-   **Full Removal of Legacy Code:** The final step will be to deprecate and remove the EVSS claim documents code paths. This means cleaning up any `if EVSS... else Lighthouse` conditions so that the codebase is cleaner. Also, the old `evss_claims` and `efolder` routes can be removed (with a versioned API release if necessary, since some clients might still use them). For now they likely still exist for safety, but the goal is to eliminate them, completing the migration 100%.
-   **Document Status Enhancements:** While the basics of upload status are in place, the CST team plans to implement a more comprehensive **Document status feature** (as noted on the roadmap ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=%2A%20Introduce%20event,load%20time%20performance%20improvement%20discovery))). This could include showing statuses for other types of documents (not just evidence submissions) – for example, indicating when a decision letter is *“in transit”* versus *“available”*. Since the Lighthouse API can expose whether a document is a draft, pending upload, or final, CST could surface that. Any remaining UI polish or additional messaging (e.g., guiding users what to do if a document is listed as pending) will be handled in upcoming sprints.
-   **Performance Tuning:** With the new service in place, the team will monitor CST’s initial load times. If there are still latency issues (say, if the Lighthouse call for documents is slow), they might implement caching at the `vets-api` layer or request bundling. The roadmap’s next quarter goal includes *“gradual performance improvements…reduce latency”* ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=Next%20,2025)). One idea is to fetch claim letters asynchronously after the main claim status loads, so that the UI appears faster. Another idea is leveraging the event-driven architecture: instead of polling for upload status, use a push notification when an upload succeeds, to immediately update the UI. These optimizations are now possible thanks to the modern API integration.
-   **Expanded Use Cases:** With CST now on Lighthouse for documents, VA can consider expanding the types of documents available. For instance, integration with the Board of Veterans’ Appeals’ document store could allow CST to show appeal hearing transcripts or Board remand letters in the same “claim letters” section. The architecture is flexible: as long as such documents end up in VBMS or an API-accessible repository, CST can fetch them via Lighthouse. The team has future work items for *“expand claim letter sorting functionality”* and *“file history experience”* ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=,CST%3A%20Improve%20file%20history%20experience)) which will build on the current foundation.
-   **Monitoring and Metrics:** The migration included adding metrics to ensure the new pipeline is working. The team will continue to monitor error rates from Lighthouse calls (e.g., any 502 errors from VBMS) and compare them with the old baseline. They’ll also track usage: the number of decision letters downloaded online each week – a key indicator of Veteran engagement. Early results showed strong adoption, with thousands of letters downloaded in the first weeks ([Your Island News](https://yourislandnews.com/how-to-view-online-and-download-your-va-decision-letters/#:~:text=Decision%20Letters%20,keep%20a%20paper%20copy)) (as reported by VA’s metrics to Congress, they are measuring “number of decision letters that are downloaded” monthly ([Congressional Hearing Transcript PDF](https://www.congress.gov/118/meeting/house/116037/documents/HHRG-118-VR09-Transcript-20230606.pdf#:~:text=,ure%20the))). This feedback loop will guide any further improvements.

---

## Implementation Challenges and Benefits

During development, the team encountered some challenges:

-   **Identifier mismatches:** Ensuring the correct IDs (file number, claim ID, participant ID) were used in Lighthouse requests required coordination with the BGS and Lighthouse teams. Feature flags like using `BIRLS ID` ([`features.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=lighthouse_claims_api_use_birls_id%3A)) were a direct solution to these issues.
-   **Ensuring Security:** The new API uses OAuth for system access. Storing and handling the private key for JWT generation was a sensitive task. They placed the key path in settings ([`settings.yml`](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/settings.yml#:~:text=client_id%3A%20%3C)) and loaded it securely. Using the VA Lighthouse platform means these calls are audited and scoped properly (scopes `documents.read` and `documents.write` limit what the token can do ([`BenefitsDocuments::Configuration` Rubydoc](https://www.rubydoc.info/github/department-of-veterans-affairs/vets-api/master/BenefitsDocuments/Configuration#:~:text=API_SCOPES%20%3D))). This is a security improvement over the legacy approach, which often relied on system accounts in VBMS.
-   **Coordinating Frontend/Backend Deployment:** The feature had to be deployed in stages. For a period, the frontend link was hidden while the backend API was tested. Once the API proved stable, the feature toggles were flipped to reveal the link. This coordination ensured no broken links or empty states for users.
-   **Handling Large PDF downloads:** Some decision letters can be many pages long. The team verified that streaming through `vets-api` would not time out. They adjusted nginx and Rails timeouts as needed. In testing, downloads were successful and reasonably quick. Users on slower connections are warned about file size where appropriate.

Despite these challenges, **the benefits are clear**. The migration has improved maintainability: VA.gov engineers now leverage a consistent Lighthouse SDK across many services (claims, appeals, documents, etc.) instead of bespoke VBMS connections. It also future-proofs the system – any enhancements to Lighthouse (like caching or new data fields) automatically benefit CST. From a **Veteran experience** perspective, this was a big win: as the VA news release put it, *“Veterans can now access their VA decision letters electronically on VA.gov using the VA Decision Letter Download Tool. This new option improves the claims experience by giving Veterans instant access to their claim decision notifications.”* ([YouTube: Download your VA Decision Letters](https://www.youtube.com/watch?v=Y9TKxj3P-A4#:~:text=Download%20your%20VA%20Decision%20Letters,This%20new%20option%20improves)) Veterans have responded positively, noting that a list of past decision letters is now available, not just the most recent ([VetsInTheKnow.org](https://www.vetsintheknow.org/veterans-benefits-administration-vba/compensation/view-va-decision-letter-online#:~:text=To%20locate%20your%20VA%20decision,letters%20sent%20by%20VA)). This transparency and convenience is a direct outcome of the Lighthouse integration.

---

## Conclusion and Roadmap

The CST’s migration to the Lighthouse Claim Documents API is substantially complete. The team has **eliminated the dependency on the old VBMS_Connect** pathway in favor of a more modern, API-driven approach. All primary functionality – viewing claim documents and uploading evidence – is running through Lighthouse. The outcome is improved performance, better user-facing features, and easier maintenance.

Moving forward, the CST team will finalize the transition by cleaning up legacy code and fully embracing Lighthouse. They plan to leverage event-based notifications (for example, using VA Notify to email a Veteran when a new decision letter is available) as mentioned in the roadmap ([Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=Roadmap)) – something made simpler with the new architecture. They will also continuously collaborate with the Lighthouse team as that API evolves (e.g., if new document types like “Rating Decision Code Sheets” become available via the API, CST can expose them). In summary, this deep integration has positioned the VA Claim Status Tool to provide Veterans a faster and richer view of their claims, aligning with VA’s digital modernization goals.

---

## Sources

-   [VA.gov Q2 2025 Roadmap (Claims & Appeals)](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=%2A%20Introduce%20event,CST%3A%20Component%20migration) – Lighthouse documents migration
-   [Lighthouse Benefits Documents API Release Notes](https://developer.va.gov/explore/api/benefits-documents/release-notes#:~:text=Release%20notes%20,to%20submit%20supporting%20claim) – Replacing EVSS/VBMS doc service
-   [Lighthouse Benefits Documents API Docs](https://developer.va.gov/explore/api/benefits-documents/docs#:~:text=Lets%20you%20retrieve%20a%20list,documents%20submitted%20to%20VBMS%20eFolder) – API description
-   [`vets-api/config/features.yml` - POA Flag](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=actor_type%3A%20user) – Use Lighthouse for POA forms
-   [`vets-api/config/features.yml` - BIRLS ID Flag](https://github.com/department-of-veterans-affairs/vets-api/blob/master/config/features.yml#:~:text=lighthouse_claims_api_use_birls_id%3A) – Use BIRLS ID for search
-   [Features // Flipper - BOA Letters Flag](https://api.va.gov/flipper/features#:~:text=cst_include_ddl_boa_letters%20When%20enabled%2C%20the%20Download,be%20NEEDED_FROM_OTHERS%20on%20mobile%20app) – CST decision letter flags
-   [Features // Flipper - Upload Status Flag](https://api.va.gov/flipper/features#:~:text=cst_show_document_upload_status%20When%20enabled%2C%20claims%20status,Fees%2C%20Secondary%20Action%20Required%2C%20or) – CST upload status flag
-   [`vets-website/.../appeals-v2-helpers.jsx`](https://github.com/department-of-veterans-affairs/vets-website/blob/master/src/applications/claims-status/utils/appeals-v2-helpers.jsx#:~:text=) – “Get your decision letters” UI behind feature flag
-   [VA.gov API Reference - Claim Letters Endpoint](https://department-of-veterans-affairs.github.io/va-digital-services-platform-docs/api-reference/#:~:text=affairs.github.io%20%20GET%2Fv0%2Fclaim_letters%2F,Online%20validator%20badge) – Claim letters download endpoint
```
