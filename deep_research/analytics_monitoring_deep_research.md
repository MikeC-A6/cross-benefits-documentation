# Data Analytics and Observability for the VA Claims Status Tool

## Table of Contents

- [Executive Summary](#executive-summary)
- [Datadog Integration and Usage in CST](#datadog-integration-and-usage-in-cst)
  - [Backend Monitoring and APM with Datadog](#backend-monitoring-and-apm-with-datadog)
  - [Real User Monitoring (RUM) for Frontend Performance](#real-user-monitoring-rum-for-frontend-performance)
  - [Datadog Dashboards, Alerts, and Effectiveness](#datadog-dashboards-alerts-and-effectiveness)
- [Google Analytics and Application Insights](#google-analytics-and-application-insights)
  - [GA Integration and Configuration](#ga-integration-and-configuration)
  - [Data from GA Supporting Improvements](#data-from-ga-supporting-improvements)
- [Domo Dashboards and Reporting](#domo-dashboards-and-reporting)
  - [How Domo is Integrated for CST](#how-domo-is-integrated-for-cst)
- [Key KPIs and Metrics for Monitoring Claims Status](#key-kpis-and-metrics-for-monitoring-claims-status)
- [Recommendations for an Enhanced Monitoring & Analytics Strategy](#recommendations-for-an-enhanced-monitoring--analytics-strategy)
- [Conclusion](#conclusion)
- [Sources](#sources)
- [Footnotes](#footnotes)

---

## Executive Summary

The VA Claims Status Tool (CST) leverages multiple data analytics and monitoring platforms to ensure a reliable and user-friendly experience for Veterans checking their claim, decision review, or appeal status online. Over time, the tool’s team has integrated **Datadog** (for application performance monitoring, error tracking, and Real User Monitoring), **Google Analytics** (for user behavior and traffic analysis), and **Domo** (for executive dashboards aggregating key metrics like customer satisfaction). These tools work in concert to provide both technical observability and product insights:

-   **Datadog** – Used extensively for backend **APM** (traces, infrastructure metrics) and for defining alerts on error rates and anomalous traffic. It also includes **Real User Monitoring (RUM)** on the frontend to capture page load performance, JavaScript errors, and even custom user interaction events. Datadog dashboards and monitors have enabled the CST team to quickly detect issues (e.g. “silent” upload failures, spikes in errors) and measure performance improvements[^1][^2]. Alerts are sent to VA Slack channels, with an on-call rotation triaging issues in real time[^3][^4].

-   **Real User Monitoring (RUM)** – As part of Datadog’s suite, RUM provides deep insight into real users’ frontend experiences. The CST team added Datadog RUM to the client application, allowing visualization of page load times, frontend errors, and user actions in the Claims Status interface[^5]. This has been effective in correlating client-side performance with backend events and understanding how users interact with features (e.g. pagination controls or accordions are instrumented as custom RUM actions[^6][^7]). Such instrumentation supports data-driven UI improvements and helps verify that new features don’t degrade the user experience.

-   **Google Analytics (GA)** – GA has historically been used to track user behavior and traffic volumes for VA.gov. The Claims Status Tool is no exception: page views and user flows for the `/track-claims` pages are captured in GA, and the team has access to a GA dashboard specific to the product[^8]. This provides metrics like number of unique users, sessions, page bounce rates, and click events (via configured event tracking) for the CST. GA data helps the team understand usage patterns (e.g. how many veterans check status weekly), identify drop-off points in flows, and gauge the impact of content or design changes on user engagement.

-   **Domo** – Domo is used on VA.gov as a business intelligence tool to aggregate data from sources like GA (and potentially customer feedback) into executive-friendly dashboards. A **“CSAT Domo”** dashboard exists for the Claims Status Tool, allowing stakeholders to filter metrics by the tool’s URL (`/track-claims`)[^8]. “CSAT” refers to Customer Satisfaction, indicating this dashboard presents user satisfaction scores specific to the tool alongside usage statistics. In the VA context, Domo provides an **executive overview of web analytics**, tracking metrics such as site visits, feature adoption, and login methods[^9]. For the CST, this likely includes overall usage trends and user satisfaction ratings collected from feedback surveys. Domo consolidates these insights, enabling product owners and executives to monitor the tool’s success at a high level without needing direct access to raw GA or Datadog data.

**In summary**, the VA Claims Status Tool has developed a robust analytics and monitoring ecosystem. Datadog (with RUM) ensures engineering teams have real-time observability into system health and user-facing performance, Google Analytics captures broad user behavior data, and Domo surfaces key performance indicators and satisfaction metrics for leadership. This multi-layered approach has improved the team’s ability to detect issues early, prioritize enhancements based on data, and ultimately deliver a reliable service to veterans. The following sections provide detailed analysis of each tool’s integration, examples from the codebase and configuration, and how the metrics inform both technical and product decisions, followed by recommendations for a best-in-class monitoring strategy going forward.

---

## Datadog Integration and Usage in CST

Datadog has become a cornerstone of the Claims Status Tool’s observability stack, used both on the backend and frontend. Its integration is evident in the code and the team’s documentation, indicating a focus on **performance monitoring, error tracking, and alerting** for the CST and related benefit tools.

### Backend Monitoring and APM with Datadog

On the server side (the `vets-api` that powers VA.gov), the Datadog APM integration tracks requests and background jobs for the Claims Status APIs. The `vets-api` repository includes the Datadog Ruby gem, and it’s configured to instrument the application. For example, the `Rakefile` explicitly initializes Datadog tracing for background tasks[^10], and a recent update replaced the older `ddtrace` gem with the newer `datadog` library for improved compatibility[^11]. This means incoming requests to key endpoints (e.g., retrieving claim status or submitting evidence) are measured for latency, throughput, and errors as distributed traces.

The CST team has created Datadog dashboards and saved log/trace queries for the relevant endpoints. In the project’s engineering docs, a Datadog README enumerates **useful queries** targeting the Claims Status services[^12][^13]. For instance:

-   **EVSS vs. Lighthouse API Calls** – Historically, the CST relied on an internal service (EVSS) for claims data. As they migrate to the new Lighthouse API, Datadog queries are set up for both. The README shows separate queries for EVSS endpoints (e.g. `V0::EVSSClaimsAsyncController#index` and `#show`) and Lighthouse endpoints (`V0::BenefitsClaimsController#index/#show`)[^12][^14]. This allows the team to monitor performance and error rates on the legacy vs. new services in parallel. By tracking both, they can validate that the Lighthouse integration performs as well as or better than EVSS, and detect any discrepancies in data or latency between the two sources.

-   **5103 Submissions and Document Uploads** – The tool supports submitting VA Form 21-5103 (waiver of evidence waiting period) and uploading supporting documents for claims. These have been pain points in the past (e.g. “silent failures” where an upload fails without notifying the user). Datadog monitors were created to specifically watch these actions. The README highlights queries for the 5103 submission endpoint (`.../submit5103`) and the document upload endpoint (`.../evss_claims/*/documents`)[^15][^13]. By filtering logs for these paths in production, the team can see if submissions are received and if any errors occur during uploads. In fact, a dedicated Datadog dashboard named **“Zero Silent Failures – Document Uploads”** was implemented[^16], underscoring the importance of catching any instance where a document upload doesn’t succeed. This has been critical in driving the **“Eliminate Silent Failures” initiative**[^17] – an effort that led to code changes ensuring upload errors are surfaced and retried, with Datadog used to verify the efficacy of those fixes (the dashboard shows the count of silent failures trending toward zero).

-   **Error Tracking and Alerts** – Datadog’s log monitoring is also configured to trigger **error alerts** when certain thresholds are exceeded. The team’s internal monitoring handbook describes that any error related to a specific action (such as an API call failure) above a threshold (often just 1) will trigger an alert[^18]. For example, if an unexpected spike of claim detail fetch failures occurs, Datadog will flag it. These alerts funnel into Slack channels (`#benefits-management-tools-notifications` among others) for immediate team awareness[^3]. An engineer on triage duty confirms the alert and opens a bug ticket if needed, ensuring prompt follow-up[^19][^20]. This process has proven effective – for instance, if the Claims Status Tool’s traffic suddenly drops or a surge of errors appears (which could indicate an upstream outage), the team is notified within minutes.

In terms of **performance metrics**, Datadog APM captures the response times of the CST’s API endpoints. The team can visualize the distribution of request durations and identify any slow downstream calls (such as to VA backends). The Datadog queries for controller actions (e.g. `#index` for listing claims) can be turned into charts that show average and p95/p99 latency. If an endpoint’s performance regresses after a deployment, the team will see it on the dashboard and can correlate it with that release. Datadog’s tracing also helps with **root cause analysis** – for example, if a specific external dependency (like an EVSS SOAP service) is slow, traces will highlight that segment of the request. All of these capabilities have given the CST engineering team confidence to make changes and quickly catch any negative impacts.

### Real User Monitoring (RUM) for Frontend Performance

Datadog’s RUM is integrated into the client-side React application of the Claims Status Tool to capture what real users experience in their browsers. The presence of `@datadog/browser-rum` and `@datadog/browser-logs` in the frontend dependencies[^21] confirms that VA.gov’s front-end (`vets-website`) is instrumented for RUM and log collection. Indeed, the Transition Hub lists an active **RUM Dashboard** for the Claim Status Tool[^2]. This dashboard allows the team to monitor front-end metrics such as page load times, Core Web Vitals, and user session traces specific to the CST pages.

**How RUM is used:** Upon loading the Claims Status Tool, the application initializes Datadog RUM with a client token and application ID (set in configuration), enabling data capture for that session. Every route change (for example, from the claims list to a claim detail page) triggers a RUM view event that measures the load performance. This includes timestamps for First Contentful Paint, DOM load, and other timing breakdowns. Datadog RUM also automatically logs any front-end JavaScript errors or console errors. For CST, this means if a user encounters a blank page or a button that doesn’t respond due to a JS error, the team will see the error stack trace in Datadog tied to the user’s session – invaluable for troubleshooting issues that occur only in certain browsers or user contexts.

Additionally, the CST team uses RUM’s **custom action tracking** to gather analytics on specific UI interactions. The frontend codebase has references to `datadogRum.addAction(...)` for various user actions on the Claim Status pages[^6][^7]. These actions are defined in a constants file (e.g. `dataDogActionNames`) and include events like:

-   *Pagination Clicks*: When a user navigates through paginated content on the claim detail (such as toggling between groups of evidence or updates), the code calls `datadogRum.addAction(dataDogActionNames.detailsPage.GROUPING_PAGINATION)` to log that event[^22]. This tells Datadog “a user did X on the details page,” which is then countable in the RUM analytics view.

-   *Accordion Toggles*: The claim status detail page often has expandable sections (accordions) for things like status history or additional information. The team instrumented these with actions such as `REFILLS_ACCORDION` and `STATUS_INFO_DROPDOWN` when users expand or collapse them[^6][^23]. By tracking this, they can see what percentage of users are interacting with certain pieces of the UI. For example, if hardly anyone expands the “Status information” dropdown, perhaps that information could be made more prominent by default – a product insight gleaned from technical monitoring data.

These RUM custom events blur the line between traditional analytics and observability, effectively supplementing Google Analytics. They are especially useful because they tie into Datadog’s session replay and trace correlation features. The team can filter RUM data to just the Claim Status Tool’s application and see **user flows and behavior** in context. If a particular sequence of actions tends to lead to an error (say, users who rapidly paginate through updates encounter an issue), Datadog RUM can highlight that pattern. Moreover, RUM data is correlated with APM traces – meaning if a user’s click triggered an API call that errored out, the front-end and back-end pieces can be linked for full-stack troubleshooting.

From a performance standpoint, RUM has been instrumental in the CST team’s recent focus on improving load times. The product roadmap for Q2 2025 includes “Complete initial load time performance improvement discovery”[^24], indicating the team analyzed RUM metrics to identify slowdowns (perhaps the initial claims list was loading slower than desired). Having real user performance data (as opposed to just lab tests) allowed them to pinpoint that the first meaningful paint of the Claim Status page might be lagging on certain devices or networks. They could then target optimizations (like code-splitting, caching, or API throughput improvements) and later verify via RUM that these changes reduced load times for end users.

In summary, Datadog RUM’s integration in the VA.gov frontend provides **end-to-end observability** for the Claims Status Tool: from the moment a user lands on the page (and how fast it renders) to every click or error in the browser, all tied back to the supporting API calls. This has greatly enhanced the team’s ability to detect UX issues and ensure that new UI features are actually used and performing as intended. As Datadog’s own documentation notes, RUM allows one to “visualize load times, frontend errors, and page dependencies” with full-stack context[^5] – the CST team is leveraging exactly that, within the constraints of VA’s secure environment.

### Datadog Dashboards, Alerts, and Effectiveness

Throughout 2024 and into 2025, Datadog has proven its worth by catching issues and guiding improvements for the Claims Status Tool. A few concrete outcomes highlight its effectiveness:

-   **Silent Failure Detection**: As mentioned, the team discovered that some document uploads were failing without notifying users. By configuring a Datadog log monitor for the document upload endpoint and seeing a pattern of failures with no corresponding front-end error, the team quantified the scope of the problem. They created the “silent failure” dashboard and set an alert for any occurrence of that pattern[^16]. This led to engineering changes (adding backend checks and front-end error handling) to eliminate silent failures. The success of this fix was measured in Datadog – the silent failure count dropped to near zero, and any regression would page the team immediately.

-   **Traffic Anomaly Alerts**: The CST typically has usage patterns. The team set up **anomaly detection** monitors in Datadog that watch the volume of requests to the claims status endpoints, comparing current traffic to historical baselines. According to their monitoring handbook, if traffic deviates by more than a certain threshold (two standard deviations) from the norm for that time/day, an alert is triggered[^4]. This has helped catch outages or widespread user login issues. They initially used simple static “low traffic” alerts as a backstop and then introduced these more dynamic anomaly alerts to reduce false alarms[^25]. Maintaining both ensures high sensitivity to real problems while avoiding noise.

-   **Error Budget and Reliability**: Over time, by tracking error rates through Datadog, the CST team has a clear picture of their error budget. Monitors like *“EVSS::RequestDecision has exhausted its Sidekiq queue”*[^26] were set up to catch specific failure modes. When such alerts fired, the team used the data from Datadog to diagnose issues. By resolving underlying issues, these monitors eventually stop firing, indicating improved stability. The feedback loop of **monitor -> alert -> fix -> monitor clears** demonstrates effective use of observability to improve reliability.

-   **Cross-team Visibility**: Because the Datadog dashboards are accessible to others in VA, stakeholders can view the CST’s health. The Transition Hub explicitly links to a **“Benefits – Claim Status Tool Statistics” dashboard**[^1] and a **“Claim Status Tool Errors” dashboard**[^27] in Datadog. These offer a one-stop view of key metrics for anyone monitoring VA.gov products. This shared observability builds trust in the platform and speeds up incident resolution through collaboration.

Overall, Datadog’s integration has given the Claims Status Tool a **near real-time pulse**. It has reduced the mean time to detect and resolve issues, provided data to justify enhancements, and yielded insights into user behavior through RUM.

---

## Google Analytics and Application Insights

While Datadog covers the technical telemetry, **Google Analytics (GA)** focuses on **user behavior analytics** for the Claims Status Tool. GA has been in use on VA.gov for years, primarily to track how users find the site, navigate through pages, and engage with content. For the CST, GA provides a complementary perspective: where Datadog might tell the team “X number of API calls failed,” GA can tell them “Y% of users didn’t complete the task.”

### GA Integration and Configuration

The VA.gov frontend uses a Tag Manager setup to integrate Google Analytics. This means that page views and certain events are sent to GA’s property for VA.gov. In the context of the Claims Status Tool:

-   **Page Views and Sessions**: Each distinct page or state (Claims list, specific Claim detail, supporting evidence upload page, etc.) is tracked as a pageview in GA. The URLs like `/track-claims/your-claims` and `/track-claims/your-claims/[claimId]` are logged. The team has access to these metrics via GA dashboards. The Transition Hub references a **Google Analytics dashboard** for the product[^8], which likely filters GA data to just the CST-related pages. Through that, one can see daily active users, total visits, average time on page, etc.

-   **Event Tracking**: Google Analytics on VA.gov often uses a `recordEvent` utility to send custom events for interactions that aren’t pure page loads (e.g., button clicks). In the CST, relevant events might include clicking the *“View details”* link, starting an evidence upload, or submitting a 5103 waiver. These events would carry labels like `event: 'claims-status', action: 'view_detail'`. This allows the team to analyze funnels and drop-offs.

-   **Demographics and Acquisition**: GA can tell the CST team about users’ device type, browser, and general location. Acquisition channels are less relevant since users typically reach the CST by logging into VA.gov, but GA would still log if they came from an email link or search.

One important use of GA is measuring **overall engagement** and **satisfaction indirectly**. Metrics like **bounce rate** and **time on page** are watched. GA also facilitates A/B testing and analysis of new features.

### Data from GA Supporting Improvements

GA data has been actively used:

-   The **“CSAT Domo” dashboard filtering by `/track-claims`** strongly implies GA feeds are used to calculate **Customer Satisfaction (CSAT) scores** for the tool[^8]. Survey results (e.g., from Medallia) can be tied to GA usage stats in Domo, allowing product managers to see metrics like “Claims Status Tool – Satisfaction 82%”.
-   **Usage Growth and Adoption**: GA provides trends in usage, justifying continued investment and helping capacity planning.
-   **Content & UX Changes**: GA’s **Behavior Flow** reports show how users move through the site, potentially highlighting friction points (e.g., login drop-offs).

In essence, Google Analytics serves as the **voice of the user in aggregate**. Combined with Datadog (the **voice of the system**), the CST team gets a 360° view.

> [!NOTE]
> GA alone doesn’t catch everything (like silent API errors), which is why Datadog+RUM is crucial alongside it. Conversely, RUM might show perfect technical performance, but GA could reveal low feature engagement, highlighting usability issues.

---

## Domo Dashboards and Reporting

To communicate the performance of the Claims Status Tool to VA business stakeholders and track high-level KPIs, the team utilizes **DOMO**. Domo is a cloud-based business intelligence platform. VA’s Technical Reference Model notes that “Domo Workbench provides an executive dashboard to forecast, track and display web analytics in a graphic interface,” tracking metrics like website visits and process adoption[^9]. Within the CST context, Domo acts as the aggregator of data from GA, Datadog, and other sources into tailored visuals.

### How Domo is Integrated for CST

The Transition Hub explicitly references a *Domo analytics dashboard for CSAT* related to the Claims Status Tool[^8]. Although the Domo instance (`va-gov.domo.com`) is not publicly accessible, we can infer its integration points:

-   **Data Inputs**: Likely, Google Analytics data and Medallia/ForeSee survey data are piped into Domo. Domo’s Workbench tool can fetch data via API[^9]. Datasets in Domo might include “Daily Track Claims Page Views” and “Track Claims Daily CSAT Score.”
-   **Dashboard Cards**: On the CSAT Domo dashboard, cards might show **Daily Users vs. CSAT**, **Top User Feedback Themes**, or **Login Method Adoption** trends[^9].
-   **Executive Metrics**: Domo likely presents KPIs such as:
    -   Total Claims Status Checks (from GA)
    -   Unique Users (from GA)
    -   Customer Satisfaction Score (from surveys, linked via GA/Domo)[^8]
    -   Task Success Rate (from surveys or inferred)
    -   Error Rates / Uptime (potentially summarized from Datadog)
    -   Median Load Time (potentially from RUM)
    -   User Engagement (e.g., Avg. claims viewed per session)
-   **Historical Trends**: Domo shows trends over time and allows filtering by date range or user segments.

In practice, the CST team uses the Domo dashboard for product reviews and reporting to leadership. It serves as a **report card**, turning telemetry into **actionable intelligence**. By filtering by the CST’s URL (`/track-claims`)[^8], Domo isolates this product’s metrics, crucial for large platforms like VA.gov.

---

## Key KPIs and Metrics for Monitoring Claims Status

Based on the implementation, the CST team tracks or could track metrics across several categories:

-   **User Traffic & Engagement** (from GA):
    -   Daily/Monthly Active Users
    -   Sessions and Page Views[^8]
    -   Bounce Rate
    -   Pages per Session

-   **Feature Usage Metrics** (from GA Events / Datadog RUM):
    -   Claims Detail View Rate
    -   Supporting Document Uploads (Success Rate, Volume) - Monitored via Datadog[^16]
    -   5103 Waiver Submissions - Monitored via Datadog[^15]
    -   Download Letters Clicks
    -   UI Interaction Events (Accordion Toggles, Pagination Clicks)[^6][^7]

-   **Performance Metrics** (from Datadog APM & RUM):
    -   Page Load Time (FCP, LCP, TTI)[^5]
    -   API Response Time (Average, p99)[^12]
    -   Frontend Rendering Time
    -   Throughput and Capacity (Requests per minute, Anomaly Detection)[^4]

-   **Reliability and Error Metrics** (from Datadog):
    -   Error Rate (Backend API errors, Frontend JS errors)[^18]
    -   Uptime (Availability)
    -   “Silent” Failure Count[^16]
    -   Slack Alert Frequency (Internal metric)

-   **User Satisfaction and Feedback** (from Surveys / Domo):
    -   Customer Satisfaction (CSAT) Score[^8]
    -   Net Promoter Score (NPS) (if collected)
    -   Qualitative Feedback Themes

-   **Process & Business Metrics**:
    -   Claim Outcomes Accessed (e.g., Letters downloaded)
    -   Call Center Deflection (Correlation of tool usage vs. call volume)
    -   Login Method Adoption[^9]

**Potential opportunities** include implementing synthetic monitoring, using GA4 funnels, and deeper integration of qualitative feedback with session data.

---

## Recommendations for an Enhanced Monitoring & Analytics Strategy

To build on the current foundation, we recommend:

1.  **Unify and Correlate Data Across Tools**: Combine data from Datadog, GA, etc., in Domo or internal reports for holistic insights. Break down data silos.
2.  **Expand Real User Monitoring**: Instrument new UI elements with RUM actions. Track Core Web Vitals and long tasks, especially on mobile/older browsers. Instrument first, then launch.
3.  **Implement Synthetic Monitoring**: Use Datadog Synthetics for critical user flows (login, view claim, download letter) to catch issues during off-hours and enforce SLAs.
4.  **Leverage Advanced Analytics**: Migrate to GA4 for improved event tracking, funnel reporting, and segmentation capabilities.
5.  **Enhance Customer Feedback Integration**: Correlate qualitative feedback (survey comments) with quantitative data (Datadog session traces, GA behavior) to uncover nuanced issues.
6.  **Set Targeted Objectives (SLOs/KPIs)**: Define specific, measurable targets for key metrics (e.g., load time < 2s, CSAT > 90%, error rate < 0.5%). Monitor progress via dashboards. Move from reactive to objective-driven monitoring.
7.  **Continuous Improvement and Cross-Team Sharing**: Regularly review and update monitors. Retire old ones (e.g., for EVSS). Share insights and reports with stakeholders and related teams.

By implementing these recommendations, the VA Claims Status Tool can further mature into a **data-informed, highly observable product**, ensuring a seamless and dependable experience for Veterans.

---

## Conclusion

The VA Claims Status Tool’s monitoring and analytics evolution reflects VA.gov's shift towards modern, veteran-centric digital services.
-   **Datadog** provides granular visibility into system performance and reliability[^3][^4].
-   **Real User Monitoring** brings the client-side experience into focus[^5][^6].
-   **Google Analytics** offers insights into user behavior and engagement[^8].
-   **Domo** distills data into meaningful narratives for decision-makers[^9][^8].

By integrating these platforms, the team has created a feedback loop: deploy -> observe -> learn -> iterate. The public documentation shows a sophisticated approach to observability. Moving forward, the CST can serve as a model, using its multi-faceted observability suite to ensure both operational and product excellence. This ensures Veterans have a trusted, reliable experience when checking their claim status online.

---

## Sources

-   [VA.gov Benefits Management Tools 1 – Transition Hub](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md) - Analytics dashboards and tool links.
-   [VA.gov CST Engineering Docs - Datadog](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/engineering/DataDog) - Datadog queries.
-   [VA.gov CST Engineering Docs - Monitoring Handbook](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md) - Monitoring procedures and alerts.
-   [`vets-website` Code Repository](https://github.com/department-of-veterans-affairs/vets-website) - Datadog RUM integration and custom events.
-   [VA Technical Reference Model – Domo Workbench](https://www.oit.va.gov/Services/TRM/ToolPage.aspx?tid=16329) - Domo description.
-   [`va.gov-team` GitHub Issue #4187](https://github.com/department-of-veterans-affairs/va.gov-team/issues/4187) - Performance Monitoring Tools Evaluation (Datadog RUM features).
-   [`vets-api` Code Repository](https://github.com/department-of-veterans-affairs/vets-api) - Datadog APM integration.

---

## Footnotes

[^1]: [Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=azCs5H,%5BDD%20Letter%27s%20app)
[^2]: [Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=Monitoring%5D%28https%3A%2F%2Fvagov.ddog,b196)
[^3]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=monitors%20below%20,represented%20by%20a%20ticket%20like)
[^4]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=and%20error%20alerts,the%20low%20traffic%20alerts%20around)
[^5]: [Evaluate website performance monitoring tools · Issue #4187](https://github.com/department-of-veterans-affairs/va.gov-team/issues/4187#:~:text=RUM%20%28https%3A%2F%2Fwww)
[^6]: [vets-website@2c4fffe Commit](https://github.com/department-of-veterans-affairs/vets-website/commit/2c4fffe#:~:text=datadogRum)
[^7]: [vets-website@2c4fffe Commit](https://github.com/department-of-veterans-affairs/vets-website/commit/2c4fffe#:~:text=datadogRum)
[^8]: [Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=%2A%2AAnalytics%2A%2A%20,%5BGoogle%20Analytics%5D%28h%20ttps%3A%2F%2Fanalytics.google.com%2Fanalytics%2Fweb%2F%23%2Fanalysis%2Fp419143770%2Fedit%2FbMzsgzMCT6y)
[^9]: [Domo Workbench TRM Page](https://www.oit.va.gov/Services/TRM/ToolPage.aspx?tid=16329#:~:text=Website%3A%20Go%20to%20site%20Description%3A,or%20based%20on%20scheduled%20jobs)
[^10]: [vets-api/Rakefile](https://github.com/department-of-veterans-affairs/vets-api/blob/master/Rakefile#:~:text=Datadog.configure%20do%20)
[^11]: [department-of-veterans-affairs/vets-api GitHub Repo](https://github.com/department-of-veterans-affairs/vets-api#:~:text=GitHub%20github,applications%20that%20live%20on)
[^12]: [DataDog README](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/engineering/DataDog#:~:text=)
[^13]: [DataDog README](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/engineering/DataDog#:~:text=index%20%60env%3Aeks,prod%20resource_name%3A%22V0%3A%3ABenefitsClaimsController%23submit5103%22%60%20Link)
[^14]: [DataDog README](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/engineering/DataDog#:~:text=Description%20Query%20Text%20Link%20index,prod%20resource_name%3A%22V0%3A%3ABenefitsClaimsController%23submit5103%22%60%20Link)
[^15]: [DataDog README](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/engineering/DataDog#:~:text=show%20%60env%3Aeks,prod%20%40http.url_details.path%3A%2Fv0%2Fbenefits_claims%2F%2A%2Fsubmit5103%60%20Link)
[^16]: [DataDog README](https://github.com/department-of-veterans-affairs/va.gov-team/tree/master/products/claim-appeal-status/engineering/DataDog#:~:text=)
[^17]: [Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=status%2FCST,appeal)
[^18]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=as%20a%20check%20against%20the,Usually%2C%20that%20threshold%20is)
[^19]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=%5Bthis%20one%5D%28https%3A%2F%2Fgithub.com%2Fdepartment,are%20looking%20into%20an%20alert)
[^20]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=react%20to%20the%20alert%20with,assigned%20to%20the%20next%20triage)
[^21]: [vets-website/package.json](https://github.com/department-of-veterans-affairs/vets-website/blob/main/package.json#:~:text=%22%40datadog%2Fbrowser)
[^22]: [vets-website@2c4fffe Commit](https://github.com/department-of-veterans-affairs/vets-website/commit/2c4fffe#:~:text=match%20at%20L253%20datadogRum)
[^23]: [vets-website@2c4fffe Commit](https://github.com/department-of-veterans-affairs/vets-website/commit/2c4fffe#:~:text=dataDogActionNames)
[^24]: [Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=Roadmap%20,%28Quarter)
[^25]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=alerts%20are%20a%20bit%20more,number%20of%20errors%20related%20to)
[^26]: [Benefits-Management-Tools-Monitoring-Handbook.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/engineering/Benefits-Management-Tools-Monitoring-Handbook.md#:~:text=statistics%29%20,https%3A%2F%2Fvagov.ddog)
[^27]: [Benefits Management Tools 1 Transition Hub 3.31.2025.md](https://github.com/department-of-veterans-affairs/va.gov-team/raw/refs/heads/master/products/claim-appeal-status/Benefits%20Management%20Tools%201%20Transition%20Hub%203.31.2025.md#:~:text=h_mode%3Dsliding%26from_ts%3D1742736772898%26to_ts%3D1742823172898%26live%3Dtrue%29%20,DD%20BMT%20Silent%20Failure)
```
