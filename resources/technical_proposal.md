# 1 | Technical Approach 

The Office of the Chief Technology Officer (OCTO) has set a strategic goal under the Cross Benefits Crew initiative to enhance its digital services' performance, reliability, and usability. A key challenge in this effort is reducing latency across VA.gov applications, particularly in tools like the Claim Status Tool, which provides Veterans with real-time updates on their benefits claims. Latency issues in the Claim Status Tool can lead to frustration, increased call center inquiries, and delays in accessing critical information. Given the complexity of VA's digital ecosystem-comprising multiple APIs, data sources, and front-end dependencies-optimizing performance requires a comprehensive, data-driven approach that improves observability, infrastructure efficiency, and user experience.

To develop our proposed technical approach, Team Agile Six conducted an in-depth analysis of the VA-provided repositories, including VA.gov API documentation, the vets-website repository, and VA design system guidelines. We combined this research with our extensive experience supporting VA digital services to identify key factors contributing to latency in the Claim Status Tool and related applications. Our findings revealed bottlenecks in API response times, front-end rendering inefficiencies, and suboptimal data-fetching strategies-all of which impact the speed and reliability of claim status updates for Veterans.

Our proposed technical approach focuses on reducing latency for the VA.gov Claim Status Tool by at least $50 \%$, aligning with OCTO's North Star goal to decrease the time Veterans spend waiting for an outcome. The approach includes enhanced observability through distributed tracing, improved caching strategies, parallel processing for API calls, and real-time monitoring to detect and address performance bottlenecks proactively. While our primary focus is on optimizing the Claim Status Tool, these strategies are scalable across the entire Cross Benefits Crew PWS, ensuring broader performance improvements for VA.gov and the VA Health \& Benefits mobile app.

## 1.1 | Targeted Optimization Approach

To achieve OCTO's goal of cutting application latency by at least $50 \%$, we plan to (1) establish a comprehensive view of current performance as experienced by an end user, (2) analyze performance metrics and identify the addressable bottlenecks creating latency, (3) run technical spikes to validate performance improvements and mitigate risks, and (4) implement targeted improvements in both our back-end integrations and front-end rendering. To enhance efficiency, our team will use AI-powered Integrated Development Environments (IDEs), such as GitHub Copilot, Cursor, and Cline, to accelerate development, improve code quality, and iterate rapidly—all while preserving upstream data governance.

### 1.1.1 | Building a View of Current Performance with End-to-End Observability

We will start by improving visibility into performance bottlenecks that cause latency and poor Veteran end-user experiences. Building on existing Datadog instrumentation, we will extend distributed tracing across key vets-api endpoints to capture and aggregate request lifecycle metrics from vets-api through data stores and external services. This approach will enable the VA to have a clear picture of response times across system and team boundaries and to pinpoint delays and errors.

Simultaneously, we plan to integrate Datadog Real User Monitoring (RUM) into our distributed tracing framework to extend our analysis to the end users' browsers and mobile experiences. By correlating front-end rendering times with back-end metrics, we can improve Veterans' experiences through faster page loads via front-end optimizations and caching of slow external data. We will use consistent trace IDs throughout the entire request lifecycle to further achieve continuous observability. We will also enrich application logs with metadata such as payload sizes, retry attempts, and request parameters to allow deeper analyses of slow and error-prone transactions. To promptly address performance issues, we will configure Datadog's machine learning anomaly detection on critical endpoints to alert the team if response times are abnormal or external services begin timing out at higher rates.

# 1.1.2 | Technical Spikes to Identify Opportunities and Risks 

To identify and validate the highest-impact performance improvements, we will run single iteration experiments, "technical spikes," to test optimizations and approaches. Technical spikes will also allow us to research system, data, team, and vendor boundaries and better understand associated risks and mitigations. During spikes, we will classify upstream data based on their requirements for authorization, e.g., PHI, and cacheability, e.g., frequently versus infrequently changing statuses. Possible improvements to test include:

- Optimizing the placement of business logic in vets-api, keeping it close to the user when feasible while ensuring alignment with authoritative systems.
- Consolidating request patterns by exploring parallel calls and prefetching strategies to reduce overhead.
- Examining network-level optimizations between vets-api and external services.
- Integrating advanced caching and data pre-computation strategies based on real-world usage analytics.


### 1.1.3 | Optimizing Data Retrieval and Reducing Latency

The VA.gov Claim Status Tool architecture integrates front-end components (vets-website), back-end logic (vets-api), and external services (EVSS, Lighthouse APIs, BGS, VBMS). As depicted in the diagram below, requests flow from a user's browser through vets-website to vets-api before reaching legacy and external systems for claim details. Our approach will prioritize strategies to decrease latency and improve availability to systems downstream from vets-api while ensuring that upstream systems, including BGS and CorpDB, remain the authoritative systems of record.

We propose a Slow Request Cache Layer and Temporary DB Storage within vets-api to reduce response times and improve reliability. Given that BGS, VBMS, and other legacy VA systems remain a primary source of latency, we will first conduct focused technical spikes to assess the effectiveness of this improvement. Modernized interfaces like the Lighthouse Claims API improve availability and ease of integration. However, Lighthouse still ultimately depends on the underlying systems of record.

To enhance system efficiency and reduce latency, we will explore implementing targeted optimizations across data retrieval and processing layers. This includes expanding the use of caching for frequently accessed claim data, implementing ephemeral storage in vets-api DB

tables with automatic expiration, and ensuring enforcement of authorization policies. Given the data ownership complexities and governance beyond OCTO, we will engage key stakeholders early to ensure alignment on security, compliance, and integration feasibility.
![img-0.jpeg](img-0.jpeg)

# 1.1.4 | Resiliency and Circuit Breakers 

To minimize unnecessary repeated calls to failing services and maintain a more responsive and stable user experience, we will implement a circuit breaker mechanism to avoid cascading failures when external services (EVSS, BGS, etc.) are slow or unresponsive. When monitoring detects repeated timeouts or errors from a given service, the circuit breaker will open, halting additional calls to that dependency for a set interval. We can serve cached or partial data during this period, ensuring the Claim Status Tool remains functional while the external system recovers.

### 1.1.5 | Front-End Performance

Our review of performance data may show that vets-website is a source of latency. If so, we will consider implementing or refining the following techniques:

- Asynchronous Data Fetching: Rendering critical UI elements immediately while claim details load in the background.
- Lazy Loading: Deferred loading of non-essential components until the user requires them.
- Asset Optimization: Review and reduce JavaScript bundles, removing unused libraries and employing code-splitting techniques to ensure that large assets do not block the initial render path.


### 1.1.6 | Best Practices: Continuous Testing, Monitoring, and Validation

We will integrate performance checks into the $\mathrm{CI} / \mathrm{CD}$ pipeline to verify that each new deployment maintains or improves upon existing performance baselines. We will conduct $\mathrm{A} / \mathrm{B}$ testing behind feature flags to confirm that our changes lead to meaningful latency reductions in production without introducing instability. Finally, we will enhance performance monitoring to measure our progress, maintain service uptime, and catch performance regressions in real time.
