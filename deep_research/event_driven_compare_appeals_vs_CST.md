# Comparison of Event-Driven Architectures: Appeals vs. CST Teams

## Table of Contents

- [Executive Summary](#executive-summary)
- [Appeals Team’s Event-Driven Approach (Appeals Consumer)](#appeals-teams-event-driven-approach-appeals-consumer)
  - [How it consumes events](#how-it-consumes-events)
  - [Processing logic](#processing-logic)
  - [Example](#example)
  - [Code structure](#code-structure)
  - [Configuration & deployment](#configuration--deployment)
  - [Error handling and logging](#error-handling-and-logging)
- [CST Team’s Event-Driven Approach (EventBus Gateway for Notifications)](#cst-teams-event-driven-approach-eventbus-gateway-for-notifications)
  - [Kafka Subscription](#kafka-subscription)
  - [Event Message Contents](#event-message-contents)
  - [Processing & Notification Logic](#processing--notification-logic)
  - [Use of VA Notify](#use-of-va-notify)
  - [Feature Flags and Control](#feature-flags-and-control)
  - [Error Handling & Retries](#error-handling--retries)
  - [Operational Deployment](#operational-deployment)
  - [Interaction with CST (vets-api and UI)](#interaction-with-cst-vets-api-and-ui)
  - [Diagram – CST EventBus Gateway flow](#diagram--cst-eventbus-gateway-flow)
  - [Design motivations](#design-motivations)
  - [Current status](#current-status)
  - [Distributed vs. centralized consideration](#distributed-vs-centralized-consideration)
- [Architectural Comparison: Appeals vs. CST Approaches](#architectural-comparison-appeals-vs-cst-approaches)
  - [Comparison Table](#comparison-table)
- [Lessons Learned and Best Practices](#lessons-learned-and-best-practices)
- [Recommendations and Conclusion](#recommendations-and-conclusion)
- [References](#references)

---

## Executive Summary

Both the Appeals team and the Claim Status Tool (CST) team at VA have adopted event-driven architectures to react to important back-end events, but they have chosen different approaches in implementation.

The **Appeals team’s approach** centers on a dedicated microservice (the **`appeals-consumer`**), which listens for events on the VA Enterprise Event Bus and directly updates the Appeals/Caseflow system. This approach is relatively straightforward: when an upstream system (VBMS/BIP) publishes an event (e.g., a new appeal or claim review intake completed), the `appeals-consumer` service consumes it and triggers the appropriate action in Caseflow (such as creating a record or updating status).

The **CST team’s approach**, on the other hand, introduces a centralized **`EventBus Gateway`** service – also a Kafka consumer microservice – primarily to send out notifications. When a “Decision Letter Available” event is published to the event bus, the gateway service consumes it, fetches the Veteran’s contact info, and dispatches an email via VA Notify to alert the user of the new letter. This model involves more integration steps (enterprise event bus → gateway → VA Profile → VA Notify) compared to the appeals service’s direct internal update flow.

In summary:
-   The **Appeals approach** is a self-contained consumer that feeds events into the Appeals system, simplifying data synchronization across systems.
-   The **CST approach** uses an intermediary service to facilitate communication between back-end events and user-facing notifications, decoupling the notification logic from the main application.

The appeals solution is considered simpler in terms of logic (consume event → update DB) and scope, whereas the CST solution, while more complex operationally, provides a reusable framework (a “gateway”) for handling events and sending notifications. Key trade-offs emerge in complexity, operational overhead, and maintainability: the `appeals-consumer`’s focused design minimizes complexity but is specific to its domain, while the CST gateway centralizes handling for notifications but must manage multiple external interactions ([CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=clear%20evidence%20in%20public%20commits,api%60%20code)). Developer agility can be high in both models if implemented well – appeals developers can iterate on their consumer independently, and CST developers can do the same with the gateway service (rather than deploying the entire `vets-api`).

The following sections delve into each approach, compare their architecture and code structure, and examine trade-offs in detail, with lessons learned that inform how CST might simplify or enhance its event-driven system.

---

## Appeals Team’s Event-Driven Approach (Appeals Consumer)

The Appeals team implemented an **`appeals-consumer`** service as a dedicated microservice to handle events from the VA Enterprise Event Bus. This service is a Rails application that runs a Kafka consumer (using the Karafka framework) alongside a standard Rails web process for infrastructure support ([CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=,of)). (In development, for example, the Appeals Consumer uses **Supervisor** to run multiple processes in parallel – the Rails server and the Karafka consumer – when you bring the service up ([department-of-veterans-affairs/appeals-consumer](https://github.com/department-of-veterans-affairs/appeals-consumer#:~:text=Appeals%20Consumer%20utuilizes%20Supervisor%20to,you%20run%20make%20up)).) In production, the Rails component likely provides environment and healthcheck endpoints, while the Karafka component continuously listens for relevant Kafka topics. The purpose of this service is to **consume events published by upstream VBA systems (VBMS/BIP)** and ensure the Appeals/Caseflow system stays in sync with those events.

### How it consumes events

The `appeals-consumer` subscribes to specific Kafka topics on the VA Enterprise Event Bus that pertain to appeals. For example, when VBMS (via BIP) completes a claims intake for a Higher-Level Review or Supplemental Claim, it publishes a `DecisionReviewCreated` event (an Avro-defined message containing details of the new review) to the enterprise event bus. The `appeals-consumer` service is configured (via Karafka routing) to subscribe to that topic. As soon as the message is available, the Karafka consumer in `appeals-consumer` picks it up. Under the hood, Karafka manages a consumer group and handles polling of the Kafka broker, delivering the event payload to the consumer code.

### Processing logic

Once the Appeals consumer receives an event, it triggers downstream actions in the Appeals domain. In this case, the event contains the data needed to create or update a record in Caseflow. The `appeals-consumer` code will parse the event (which might be in Avro or JSON format with fields like `claimId`, `decisionReviewType`, `veteranParticipantId`, etc.) and then use that data to call **Caseflow’s intake logic**. This could be done by invoking Caseflow’s APIs or (since the `appeals-consumer` is a Rails app possibly running in the same network environment) directly calling ActiveRecord models that correspond to Caseflow’s database ([department-of-veterans-affairs/appeals-consumer](https://github.com/department-of-veterans-affairs/appeals-consumer#:~:text=GitHub%20github,will%20pushlish%20a%20message)). In essence, the consumer reproduces what would happen if a user had initiated the intake via Caseflow’s UI: it creates a new intake record and associated claim review record in the Caseflow system, using the data from VBMS. The HackMD documentation by the Appeals team confirms the needed data points and how they map into Caseflow’s models (e.g., mapping `DecisionReviewCreated` event fields into a new `HigherLevelReview` or `SupplementalClaim` record in Caseflow) ([CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=The%20VA%E2%80%99s%20Claim%20Status%20Tool,link%29%20under%20feature), [CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=%5B,team%2Fissues%2F77622%20%3A~%3Atext%3DEventBus%2520Gateway%2520is%2520a%2520Rails%2Ca%2520Veteran%2520via%2520VA%2520Notify%29)).

### Example

If a Higher-Level Review (HLR) is filed through VBMS (instead of through Caseflow directly), VBMS/BIP will handle it and then publish an event that a new HLR was created. The Appeals consumer picks up this event ([CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=,of), [CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=,needed%20to%20trigger%20the%20notification)) and uses Caseflow Intake code to create a corresponding `HigherLevelReview` record in Caseflow’s DB. This ensures that even though the HLR came from outside Caseflow, Caseflow is immediately aware of it – without manual data entry or nightly batch jobs. By the time the Veteran goes to VA.gov to check appeal status, that HLR will appear just as if it had been filed through Caseflow ([CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=,gov%20letters%20page%2C%20etc), [CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=%3A~%3Atext%3DEventBus%2520Gateway%2520is%2520a%2520Rails%2Ca%2520Veteran%2520via%2520VA%2520Notify%29%20,team%2Fissues%2F77622%20%3A~%3Atext%3DEventBus%2520Gateway%2520is%2520a%2520Rails%2Ca%2520Veteran%2520via%2520VA%2520Notify%29)). In short, the event bus + `appeals-consumer` replaces what could have been a painful synchronization problem with near-real-time updates.

```mermaid
graph LR
    A[VBMS/BIP] -- Publishes event --> B(VA Enterprise Event Bus);
    B -- Event --> C{Appeals Consumer<br/>(Karafka)};
    C -- Creates/Updates record --> D[Caseflow System<br/>(DB/API)];
```
*Figure: Simplified event flow for the Appeals Consumer service. VBMS/BIP publishes an event (e.g., a new claim review intake) to the VA Enterprise Event Bus, which the Appeals Consumer (Karafka) consumes. The consumer then creates or updates the relevant record in Caseflow. This way, the appeals/Caseflow system stays in sync with upstream processes, and Veterans see up-to-date appeal status or documents in the UI.*

Once the Caseflow system is updated, **downstream effects** include the Veteran-facing interfaces showing the new information. For instance, VA.gov’s Appeals Status API (which queries Caseflow) or Caseflow’s own UI will now reflect the new appeal or decision ([CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=the%20user%20that%20they%20can,claim%20should%20see%20the%20link), [CST_Event_Bus_deep_research.md](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md#:~:text=appeals%2C%20once%20a%20Board%20of,This%20is%20only%20shown%20if)). The `appeals-consumer` service itself does *not* directly notify the Veteran – its job is purely to keep data in sync. Veteran notifications in the Appeals context typically happen via mailed letters or other processes, not via this service. This constrained responsibility makes the `appeals-consumer` code relatively simple: it doesn’t need to call email services or check user preferences; it just transforms event data into database calls.

### Code structure

In the `appeals-consumer` repository, we expect to see a typical Karafka setup. Likely, there is a `Karafka.boot!` or initializer that sets up the Kafka consumer configuration (brokers, client id, etc.), and a routing file (perhaps `karafka.rb` or a block in an initializer) that defines which topics to subscribe to and which consumer classes handle them. For example, it might define a consumer group and a consumer for the `DecisionReviewCreated` topic, mapping it to a Ruby class (e.g., `DecisionReviewCreatedConsumer`). That class would inherit from `Karafka::BaseConsumer` (or `ApplicationConsumer`) and implement a `#consume` method containing the business logic – in this case, calling Caseflow’s intake code. Because this is a Rails app, the consumer can likely leverage ActiveRecord models or external service classes defined in the project to perform the action. According to the repository info, “Appeals Consumer is a Rails application responsible for processing intake data sent by VBMS. When VBMS handles an intake, they will publish a message …” (the continuation presumably notes the event bus topic, confirming this flow).

### Configuration & deployment

The `appeals-consumer` likely runs in VA infrastructure where it can reach both the Kafka brokers and the Caseflow system. The repository indicates that the Docker compose for Appeals Consumer includes a Kafka broker for local testing – i.e., developers can spin up a Kafka container to generate events and test the consumer locally. In VA environments, the service would be configured with the real Kafka bootstrap servers (the Lighthouse-managed event bus endpoints) and security credentials if any. Environmental settings probably also include group IDs, topic names, and any feature flags. It’s unclear if the Appeals team gated this consumer behind a feature toggle – since it’s a separate service, enabling or disabling it could simply be a matter of running it or not. They might have environment variables to control which topics to listen to (for example, to easily disable processing certain event types if needed).

### Error handling and logging

By using Karafka, the `appeals-consumer` benefits from at-least-once delivery semantics. If processing an event fails (say, the Caseflow DB is temporarily unreachable or an exception is raised in the consumer logic), Karafka can be configured to not acknowledge the message, allowing a retry later. The Appeals consumer likely logs failures and perhaps uses Sentry or another error monitoring to capture exceptions. Because missing an event could have significant consequences (a Veteran’s appeal not showing up in Caseflow), robust error handling is important. The code likely wraps the intake creation in error rescue blocks – e.g., if an intake record can’t be created due to validation error, the service would log it and perhaps send an alert. In practice, those scenarios might be rare if the event data is well-formed; still, logging is critical. There was mention of “exhaustion handler” and “silent failure” monitoring in context of CST – the Appeals team likely has similar monitoring to ensure no silent drops of events.

In terms of maintainability, the Appeals consumer is **highly focused** on appeals and decision reviews. This makes the codebase relatively small and easier to change when needed. For instance, if a new type of event is introduced (say, a future “AppealDecisionPosted” event for Board of Appeals decisions), the team could add a new consumer for that topic in this same service, or extend an existing one. Because the consumer is decoupled from the main Caseflow app, it can be deployed independently – allowing the Appeals team to adjust their event handling without impacting the Caseflow UI or other functions.

Overall, the `appeals-consumer` approach demonstrates a clean separation of concerns: **the enterprise event bus integration is handled in one place (this microservice) rather than scattered throughout the application.** This makes the integration easier to reason about. It’s essentially a bridge between VBMS/BIP events and Caseflow’s internal logic. The simplicity of “consume event → call local code” is a strength of this model, resulting in a straightforward flow with minimal hops. As we’ll see, the CST approach, while also using a microservice, involves a few more steps due to the need to notify the user.

---

## CST Team’s Event-Driven Approach (EventBus Gateway for Notifications)

The CST team’s event-driven architecture is centered on sending **real-time email notifications** to Veterans when a new decision letter becomes available. To achieve this, the team introduced a new service known as the **`EventBus Gateway`**. This Gateway is a standalone Rails application (or at least a Ruby application) using Karafka, much like `appeals-consumer`, but its role is different: it listens for a specific event (`DecisionLetterAvailable`) on the VA Enterprise Event Bus and, upon receiving it, triggers a **notification workflow** rather than directly updating a VA database. In effect, it serves as a bridge between the **event bus** and **VA Notify (the email service)**, with a lookup to **VA Profile** in between to get the user’s email address.

Here’s how the CST `EventBus Gateway` is structured and operates:

### Kafka Subscription

The gateway service is configured to subscribe to the **“DecisionLetterAvailable”** topic (or a similarly named topic) on the Enterprise Event Bus. This topic is published to by an upstream system whenever a new decision letter PDF is ready for the Veteran. The upstream event source is the **Benefits Integration Platform (BIP)** or another claim/letters service in VBA that knows when the letter is generated. Essentially, when a disability claim is decided and a letter is prepared (e.g., a 5103 notice or decision packet), that system emits an event to Kafka indicating “letter X for Veteran Y is now available.” The Gateway’s Karafka consumer, running continuously, will pick up this message.

### Event Message Contents

According to the design, the event likely contains identifying information such as the Veteran’s file ID (or EDIPI/ICN) and maybe a claim ID or letter ID. Notably, it probably does **not** contain personally identifiable information like name or email – those are left out for privacy, requiring the consumer to fetch those details if needed. The message basically says “Letter available for Veteran X’s claim Y.” The Gateway must take this minimal data and proceed to notify the user.

### Processing & Notification Logic

When the Gateway’s consumer receives a `DecisionLetterAvailable` event, it begins the **notification sequence**.
1.  It needs the Veteran’s contact info. The service uses the identifiers from the event (for example, an EDIPI or ICN, or perhaps a BIP-issued universal ID) to query **VA Profile**, which is the VA’s source of contact information for Veterans.
2.  VA Profile can provide the Veteran’s email address (and also flags for communication preferences). This lookup is likely done via an internal API call – the Gateway, being a Ruby/Rails app, can either use an existing gem or service object similar to how `vets-api` would fetch profile data.
3.  Once the email address is obtained (and assuming the Veteran has not opted out of email notifications), the Gateway then calls **VA Notify**, the VA’s transactional email service.
4.  The Gateway supplies VA Notify with a template ID corresponding to the “Decision Letter available” email template and personalization details (like the Veteran’s name, maybe the type of claim, and a link or instructions to view the letter on VA.gov).
5.  VA Notify then sends the email to the Veteran’s address.

### Use of VA Notify

VA Notify is designed as a generic GOV.UK Notify-based service for sending emails/SMS. It does not automatically know when to send something – it only sends when an API call tells it to. So the Gateway is responsible for calling that API at the right time with the correct template. The Notify template for decision letters likely includes dynamic fields such as the Veteran’s name and maybe a hyperlink to the VA.gov Claim Letters page (or simply tells them to log in to VA.gov to view the letter). The CST team would have prepared this template in advance (the deep research mentioned they planned a specific template for “Claim Decision Letter Available”).

### Feature Flags and Control

Unlike `appeals-consumer` which directly updates data, the Gateway’s actions are user-facing (sending an email). To avoid spamming users or to allow a soft launch, the CST team likely implemented a way to **toggle the notifications on/off**. Since this Gateway is separate from `vets-api`, they cannot use the typical Flipper feature flags that `vets-api` and `vets-website` use for UI features. Instead, they might use an environment variable or a config setting within the Gateway service to control whether it actually sends the email or simply logs the event. The deep research suggests the idea of a “dry-run” mode where initially the service might log events without invoking Notify. This ensures during testing that they can verify events are received properly before flipping the switch to true email sending. By the time it’s fully enabled, that toggle can be set to send out the real notifications.

### Error Handling & Retries

There are multiple points of potential failure in this flow: the VA Profile lookup could fail, the VA Notify API call could fail, or the email could bounce. The Gateway needs to handle these gracefully. Karafka gives the ability to retry processing. The Gateway could catch errors and decide to retry after a delay or log and swallow it. At-least-once delivery semantics mean an event won’t be lost without an attempt; however, it could result in duplicate emails if a failure occurs after Notify actually sent the email but before the Gateway acknowledges the message. The CST team is likely okay with a duplicate email in rare cases. They probably incorporated logging for success/failure. The `vets-api` has a `VANotify::EmailJob` and logging for Notify usage, and the Gateway could use similar mechanisms. The deep research notes that the Notify utility in `vets-api` logs metrics for delivery success/failure, and that the Gateway should do similar.

### Operational Deployment

The `EventBus Gateway` is deployed on the VA.gov platform (within the Kubernetes infrastructure that also hosts `vets-api`). It was noted that this is one of the first VFS (Veteran-facing services) team-built applications on that platform besides the main `vets-api`. As such, the team had to set up new pipeline components: e.g., a new GitHub repo for the service, Helm charts for deployment, an Argo CD configuration, etc. There were specific issues tracked (like issue `#90484` for repo creation and dev environment, `#95680` for Helm chart, `#95689` for CI/CD). This overhead is purely an implementation detail, but it contrasts with `appeals-consumer` which likely was set up in an environment managed by the Appeals team. The CST team had to coordinate with DevOps folks (OCTO Platform) to get this service running. After initial deployment, the Gateway can be scaled and managed like other microservices. It likely has a Kubernetes **Horizontal Pod Autoscaler** or a fixed replica count. It also should have liveness/readiness probes.

### Interaction with CST (`vets-api` and UI)

Interestingly, the `EventBus Gateway` does *not* directly interface with `vets-api` at runtime. The flow of data to the user is via email. The **CST frontend (VA.gov Claim Status Tool UI)** was updated to encourage users to check for decision letters, but that was done through feature flags and static links, not through dynamic push updates. In other words, the UI does not receive a push notification or anything from this service; the user’s inbox is the notification channel. When the Veteran clicks the email and goes to VA.gov, the existing CST API (in `vets-api`) will fetch the available letters (now including the new decision letter) from the upstream (likely BIP’s document service). This part was already built. The **roadmap** indicates that as of Q1 2025, this feature was in staging testing, and the front-end had the “Get your decision letters” link behind feature toggles which are now enabled for all users.

### Diagram – CST EventBus Gateway flow

```mermaid
graph LR
    A[BIP/VBA System] -- Publishes event --> B(VA Enterprise Event Bus);
    B -- Event --> C{EventBus Gateway<br/>(Karafka Consumer)};
    C -- Looks up email --> D[VA Profile API];
    C -- Sends email via --> E[VA Notify API];
    E -- Email --> F((Veteran's Inbox));

    subgraph "Separate User Action"
        G[Veteran Logs into VA.gov] --> H{CST UI};
        H -- Requests letters --> I[vets-api];
        I -- Fetches letter --> J[BIP/Document Service];
        J --> I --> H --> G;
    end

    style F fill:#f9f,stroke:#333,stroke-width:2px
```
*Figure: Conceptual model of the CST EventBus Gateway notification flow. The Benefits Integration Platform (or another VBA system) publishes a “DecisionLetterAvailable” event to the VA Enterprise Event Bus when a new decision letter is generated. The EventBus Gateway service (Karafka consumer) in VA.gov consumes the event. It then looks up the Veteran’s email via VA Profile and uses VA Notify to send an email alert to the Veteran. The dashed box indicates that, separately, the Claim Status Tool (CST) UI on VA.gov can retrieve the letter via API when the Veteran checks, but that process is independent of the event-driven notification.*

### Design motivations

The CST team chose this architecture to decouple the notification sending from the main application (`vets-api`). Introducing a persistent Kafka listener inside the large `vets-api` monolith could have unforeseen impacts on performance and deployment. By creating a separate “EventBus Gateway” service, they achieved isolation. It also aligns with a pattern of treating event consumption as an infrastructure concern. The term “Gateway” might imply future handling of multiple event types or acting as a shared gateway. Currently, it’s focused on decision letter events. The **roadmap** for next quarter mentions exploring “additional claim events via event-based architecture (Event Bus)”, hinting the Gateway might evolve.

### Current status

As of early April 2025, the `EventBus Gateway` has been developed and is undergoing testing in staging environments. Major milestones like the repository creation and dev environment setup have been completed. The feature flags on the front-end are enabled, and the Notify templates are presumably created. Remaining steps include end-to-end testing and eventually deploying the service to production behind a feature toggle. Risks identified include ensuring the upstream event publisher is reliably sending events and that Veteran communication preferences are respected.

### Distributed vs. centralized consideration

Initially, the team could have considered integrating Kafka consumption into `vets-api` directly. However, the decision to externalize the event handling was deliberate, to keep concerns separated. The **CST approach is “centralized”** in the sense that one new service (the gateway) handles the cross-system communication for CST notifications, rather than sprinkling that logic in the existing app. This can be contrasted with Appeals, where their consumer is dedicated to their domain. In practice, both teams ended up with a separate service – but the CST team’s service acts as a *gateway to multiple external systems (Profile, Notify)*, whereas the Appeals service primarily interacts with its own system.

To summarize the CST event-driven design: an upstream system produces an event -> the **`EventBus Gateway`** consumes it and acts on it -> an email is sent to the user through Notify.

---

## Architectural Comparison: Appeals vs. CST Approaches

At a high level, **both the `appeals-consumer` and the CST `EventBus Gateway` are Kafka consumer microservices** that react to events and trigger some action. They share underlying patterns (use of Karafka, connection to the VA Enterprise Event Bus) but differ in purpose and complexity.

### Comparison Table

| Aspect                   | Appeals Consumer (Dedicated Microservice)                                     | CST EventBus Gateway (Centralized for Notifications)                         |
|--------------------------|-------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| **Primary function**     | Sync data (ensure Caseflow reflects VBMS events for appeals/HLR).             | Send user notifications (email) for claim events (decision letters).        |
| **Triggers on**          | Events like `DecisionReviewCreated` (new appeal/review intake) from VBMS.     | Events like `DecisionLetterAvailable` from BIP (claim decision outcome).    |
| **Tech stack**           | Rails + Karafka, internal Caseflow DB/API calls.                              | Rails + Karafka, calls to VA Profile API and VA Notify API.                 |
| **Downstream effect**    | Updates Caseflow records (Veteran sees updated status in VA.gov/Caseflow UI). | Veteran receives an email (and can then log in to see the letter online).   |
| **Integration points**   | VBMS/BIP (producer of events), Caseflow (consumer target).                    | BIP (producer), VA Profile, VA Notify (downstream services).                |
| **Complexity**           | Relatively simple flow (one hop into Caseflow).                              | Multi-step flow (needs to handle external APIs and conditions).             |
| **Coupling**             | Tightly coupled with appeals domain (knowledge of Caseflow models).           | Loosely coupled; mostly generic handling (coupled to Notify templates).     |
| **Team ownership**       | Owned by Appeals/Caseflow team (domain experts).                              | Owned by CST team (notifications domain for claims).                        |
| **Deployable unit**      | Separate service deployed in Appeals infrastructure (likely AWS).             | Separate service deployed on VA.gov platform (Kubernetes with `vets-api`).    |
| **Feature toggle**       | On/off by deploying or environment config (no end-user feature flag needed).  | Toggled via config (initially can run in dry mode, independent of front-end flags). |
| **Example outcome**      | HLR filed in VBMS → event → Caseflow creates HLR record (Veteran sees it in status). | Claim decided → event → Gateway emails Veteran "Your decision letter is ready". |

This comparison highlights that **the appeals approach is domain-focused and internally oriented**, while **the CST approach is user-focused and externally oriented**.

What makes the appeals approach *simpler or more straightforward* than the CST’s gateway model?
-   **Scope of Responsibility:** Appeals-consumer deals with one boundary (VBMS → Caseflow). CST gateway deals with multiple boundaries (BIP → Gateway → Profile → Notify). Fewer moving parts means simpler configuration and reasoning.
-   **Implementation Effort:** Appeals team likely leveraged existing Caseflow code. CST team had more cross-team coordination (Notify, Profile).
-   **Testing Simplicity:** Testing appeals-consumer involves verifying DB state changes. Testing Notify gateway involves checking external side effects (email sending).
-   **Operational Risk:** Appeals-consumer failure affects internal data sync. CST gateway failure directly impacts user notifications and preferences.

In essence, **the appeals approach is simpler in implementation and operation, while the CST approach is broader in integration but achieves a different goal**.

---

## Lessons Learned and Best Practices

Both implementations offer insights into building event-driven services within VA’s ecosystem:

-   **Keep the Service Focused:** Appeals-consumer's single purpose aids maintainability. CST gateway should remain focused on notifications. Avoid scope creep.
-   **Leverage Established Frameworks:** Both used Karafka. Consistency aids knowledge transfer and reduces reinvention. Share configuration patterns (e.g., offset policies).
-   **Development and Testing Parity:** Local testing environments (like Appeals' Docker Kafka) are crucial. Test the full pipeline in staging using representative data and sandbox endpoints (e.g., Notify sandbox).
-   **Robust Logging and Monitoring:** Log key steps and outcomes. Use metrics (StatsD/Datadog) for event counts, errors, latency. Set up alerts for anomalies (zero events, error spikes). Treat consumers as critical infrastructure with health checks.
-   **Idempotency and Duplicate Handling:** Design for at-least-once delivery. Make handling idempotent or harmless if repeated. Appeals might use "find or create". CST might accept rare duplicate emails.
-   **Respecting User Preferences and Data Privacy:** CST gateway must handle contact info and opt-outs carefully, relying on VA Profile/Notify mechanisms. Log decisions (e.g., skipping notification due to opt-out).
-   **Use Feature Flags for Safe Deployment:** CST's use of flags for incremental rollout is a good practice. Have a "kill switch" (env var) for emergencies.
-   **Cross-Team Communication & Contracts:** Maintain clear API/schema contracts for events. Use schema validation in consumers. Treat events as public contracts.
-   **Microservice vs. Monolith:** Both teams chose separate services over modifying monoliths, validating the microservice approach for decoupling event handling.

---

## Recommendations and Conclusion

**For the CST team, the findings suggest sticking with the dedicated microservice (gateway) approach** is the right call. It provides isolation and avoids complicating `vets-api`. The `EventBus Gateway` should continue as a **single-purpose notification service** for claim events. A decentralized pattern (each domain team maintains its own consumers) is preferable to one central VA event handler.

Recommendations for CST:
-   **Continue with the `EventBus Gateway` microservice:** Do not refactor into `vets-api`.
-   **Apply lessons from Appeals:** Keep logic simple, use flags, document clearly.
-   **Consider fallback only if necessary:** A hybrid approach (e.g., `vets-api` nightly job) adds complexity; rely on the event pipeline's reliability first.
-   **Maintain close collaboration with producers:** Ensure event contracts are stable and plan future changes together.
-   **Invest in monitoring and scalability:** Ensure the service scales and has robust alerting.

Adopting these recommendations, the CST team can confidently move forward with an event-driven notification service that is maintainable and agile. The event-driven approach, as proven by the Appeals team, can significantly improve user experience with minimal ongoing overhead. By following the Appeals team’s simpler model of “one event, one focused consumer, one outcome,” the CST team can avoid over-engineering. In summary, **the CST `EventBus Gateway` should remain a dedicated microservice – this design best balances complexity and benefit.**

---

## References

-   **Appeals Consumer Repository:** [`department-of-veterans-affairs/appeals-consumer`](https://github.com/department-of-veterans-affairs/appeals-consumer) – VA GitHub repository for the Appeals Kafka consumer microservice.
-   **VA.gov Claim Status Event Notification Epic:** [`va.gov-team#77622`](https://github.com/department-of-veterans-affairs/va.gov-team/issues/77622) – Product documentation for the CST event-driven notification feature.
-   **CST Event Bus Deep Dive Report:** [`CST_Event_Bus_deep_research.md`](https://github.com/MikeC-A6/cross-benefits-documentation/blob/main/deep_research/CST_Event_Bus_deep_research.md) (March 2025) – Internal analysis document summarizing the CST team’s event-driven architecture.
-   **VA Notify Service Documentation:** [`va.gov-team/products/va-notify/README.md`](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/va-notify/README.md) – Explains VA Notify’s role and behavior.
-   **Benefits Management Tools Transition Hub – Q1 2025 Roadmap:** [`va.gov-team/.../Benefits Management Tools 1 Transition Hub 3.31.2025.md#roadmap`](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#roadmap) – Roadmap and status for CST product initiatives.
-   **Appeals Frontend Example (VA.gov):** [`vets-website/.../appeals-v2-helpers.jsx`](https://github.com/department-of-veterans-affairs/vets-website/blob/master/src/applications/claims-status/utils/appeals-v2-helpers.jsx) – Frontend code snippet showing decision letter link.
```
