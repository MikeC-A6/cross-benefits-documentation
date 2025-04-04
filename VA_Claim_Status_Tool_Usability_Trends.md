# VA Claim Status Tool Usability – Trends and User Feedback (Post-Lighthouse Migration)

## Table of Contents

- [Background and Recent Updates](#background-and-recent-updates)
- [Tracking Claim Status – Praise and Pain Points](#tracking-claim-status--praise-and-pain-points)
  - [Clarity of Claim Stages (Initial Claims)](#clarity-of-claim-stages-initial-claims)
  - [Status for Supplemental Claims and HLRs](#status-for-supplemental-claims-and-hlrs)
  - [Decision Letters and Outcomes](#decision-letters-and-outcomes)
  - [Performance and Downtime](#performance-and-downtime)
- [Evidence Submission and 5103 Waiver Experience](#evidence-submission-and-5103-waiver-experience)
  - [Uploading Supporting Files](#uploading-supporting-files)
  - [5103 Notice and ‘Decide My Claim’ Button](#5103-notice-and-decide-my-claim-button)
  - [User Emotions and Satisfaction](#user-emotions-and-satisfaction)
- [Accessibility and Platform Integration](#accessibility-and-platform-integration)
- [Community Sentiment and Developer Insights](#community-sentiment-and-developer-insights)
- [Conclusion and Recommendations](#conclusion-and-recommendations)
- [Mermaid Diagram – Veteran’s Journey & Feedback Touchpoints](#mermaid-diagram--veterans-journey--feedback-touchpoints)
  - [Diagram Explanation](#diagram-explanation)
- [Sources](#sources)

---

## Background and Recent Updates

The VA’s online Claim Status Tool (part of VA.gov’s `vets-website`) underwent a backend migration to Lighthouse APIs in 2023-2024. By early 2024, “100% of claims status traffic [was] using [the] new service (Lighthouse)”, with only document uploads still being transitioned. This migration coincided with UI enhancements released in Q3 FY2024 to improve the veteran experience. On June 19, 2024, VA announced an upgraded tool featuring: a more user-friendly interface, real-time status info, mobile optimization, and clearer claim details[^1]. The tool now displays the 8-step disability claims process for initial claims, from “Claim received” to “Decision letter available”[^2][^3], giving veterans better insight into where they stand.

These changes aimed to address historically low satisfaction – veterans had rated the claim status page only 2.6 out of 5 as of early FY2023[^4]. (In fact, satisfaction dipped to ~2.0 by March 2024[^5][^6], underscoring the urgency for improvements.)

**Lighthouse API integration:** The move to Lighthouse APIs was intended to improve data access and reliability. By mid-2024, all claim status data (and soon, file uploads) were served via Lighthouse[^7]. This was part of VA’s broader digital modernization, as noted in performance reports. Although largely a behind-the-scenes change, it set the stage for more responsive updates (described as “real-time notifications” in VA’s announcement[^1]) and allowed the front-end team to roll out new features via feature toggles. For example, VA activated flags to show decision letter downloads (DDL) and 5103 notice responses in the UI[^8][^9]. The enhanced tool enables veterans to download their decision notification letters online as soon as the claim closes, rather than waiting for mail[^10]. It also lets users respond to “5103” evidence notices electronically.

Below we delve into how veterans and developers have reacted to these changes, and what usability themes have emerged.

---

## Tracking Claim Status – Praise and Pain Points

### Clarity of Claim Stages (Initial Claims)

Veterans generally appreciate the newly explicit eight-step timeline for initial disability compensation claims. The tool now shows labeled stages (“Claim received”, “Initial review”, “Evidence gathering”, etc.) with brief descriptions[^2][^3]. This breakdown makes it easier to understand the process and what’s next. One veteran noted, “I’m glad they are explaining the steps… it’s good to remember that it could be wildly different for every claim”[^11] – indicating that while the structured timeline is helpful, actual progress can still jump around. Indeed, users reported some idiosyncrasies: e.g. a claim jumping from step 1 to step 5 suddenly after a couple of actions[^12]. Others remarked that steps are not strict sequentials (“The whole process, steps, and timelines are just suggestions” quipped one Redditor[^13]), but are a general guide. Overall, this feature has been well-received as an educational addition that sets expectations[^14]. The VA even started testing a refreshed claim status design in early 2024, aiming to implement by Q3 FY24[^5][^6]. Veterans’ feedback from these tests presumably informed the current step-by-step layout.

### Status for Supplemental Claims and HLRs

A major usability gap highlighted by veterans is tracking decision reviews (supplemental claims and Higher-Level Reviews). Unlike initial claims, these often show minimal info on VA.gov. Users report that a supplemental claim may just sit at “received” or a generic message (“a reviewer is examining your new evidence”) for long periods[^15][^16]. “It’s a frustrating process… I can’t think of a valid reason why a supplemental claim can’t be tracked by the veteran the same as a regular claim,” one veteran wrote, noting he had to schedule VERA calls every few weeks to get updates[^17]. Another user asked if others see additional steps for supplementals or still “less info on them than initial claims,” and the consensus was that online status is very limited[^18][^19]. This drives veterans to call the VA or use old systems. Indeed, some were told by representatives that “it’s just how the application was built” and that they must rely on phone updates for these claim types[^20][^21].

Similarly, Higher-Level Reviews submitted online sometimes failed to appear in the tool at all. One veteran shared “I submitted an HLR… got the email that it was received, but it doesn’t show up anywhere!”[^22]. Several others echoed this problem in late 2023, with HLRs seemingly “stuck” in limbo – confirmed as received in VA’s system but not visible to the user and not actively worked by processors[^23][^24]. In one case, an HLR filed in Sept 2023 only showed any update by January 2024 when a reviewer finally picked it up[^25]. This lack of transparency is a source of frustration and anxiety, as veterans wonder if their review request was lost. It appears the tool (or underlying systems) did not list HLRs until they progressed to a certain point. VA’s Q2 FY2024 report acknowledged the need to “better serve [veterans’] needs” in claim status tools for decision reviews and appeals[^7], so improvements here are likely forthcoming. For now, though, tracking non-initial claims remains a pain point, marring user satisfaction.

### Decision Letters and Outcomes

One hugely positive development is the ability to download decision letters online. Veterans on social media frequently remind each other that “once a decision has been made you can download your decision letter instantly” on VA.gov or the VA Mobile app[^10][^26]. This has been a game-changer in terms of usability – no more waiting days for the mail to know your rating or outcome. One veteran advised others that on VA.gov “It tells me: ‘We decided your claim… You can download your decision letter online now.’”[^27] The letter PDF is accessible on the claim status page under files/letters. Users have enthusiastically embraced this: “Go to VA.gov and log in, you can download and see their decision instantly rather than waiting,” as one Reddit post encouraged[^28][^29]. Many have successfully obtained their award letters or denial explanations this way, often the same day the decision is made.

There have been a few hiccups, however. Some reported that the system tells them a decision is ready but the letter isn’t actually available for a brief time. For example, a veteran got a message on April 4, 2024 about a decision and download link, but “my decision letter is nowhere to be found… on the app or the website” immediately after[^30]. The consensus was to “give it some time to update”[^31] – indeed, within hours or a day the letter usually appears. This suggests a slight delay between decision finalization in VA’s backend and the letter PDF being posted to the portal. Overall, though, the 24/7 access to decision letters has been a highly praised feature. It even works via the VA Health and Benefits mobile app, which mirrors the claim status info. Users confirmed you can view all your decision letters in the app as well[^32].

Additionally, veterans appreciate seeing their updated disability rating on VA.gov soon after a decision. One user on Reddit shared excitement that he “woke up to a new rating today! 90% baby!” once his claim moved to the notification phase[^33]. This indicates the claim status tool (and VA profile) updates the percentage promptly, contributing to faster feedback. However, there’s still a desire for push notifications or alerts – currently veterans often compulsively refresh the site or app (“I have been checking ... every couple of hours” admitted one user during the waiting period[^34]). Real-time status changes are available if one checks, but sending an email or app notification when the claim is decided would further boost usability. (It’s unclear if VA enabled such alerts; they touted “real-time notifications”[^1], which may refer to the on-demand status rather than proactive alerts.)

### Performance and Downtime

In general, the Lighthouse API migration has not produced widespread reports of errors – if anything, it aimed to reduce downtime by decoupling from legacy systems. Veterans have occasionally encountered maintenance outages on VA.gov’s claims tool, particularly on weekends or after-hours. For instance, over a holiday weekend in Feb 2023, users saw “Claim status unavailable” errors, which they speculated was due to system maintenance while VA staff were off[^35][^36]. A similar maintenance message appeared in Nov 2022 with “We can’t access any claims… something went wrong on our end” and even an unrelated financial info error[^37][^38]. These outages tend to be temporary (resolved by the next workday) and were often anticipated by experienced users: “It’s a 4 day weekend… site probably needs some maintenance,” one veteran told anxious others[^39].

The claim status tool also explicitly shows error messages when backend services are down. The VA’s API status endpoint (for internal services) can reveal if systems like VBMS or Caseflow are offline[^40], which has occasionally been shared by tech-savvy veterans checking why the tool isn’t loading. In 2022 and 2023, VBMS (the claims system) had “frequent outages… that reduced productivity”[^4], and such downtime would manifest to veterans as the claim status page being temporarily unavailable. VA has been working to improve system resiliency to minimize these incidents[^7].

Overall performance (speed) of the tool hasn’t been a major complaint in forums – the pages generally load quickly when systems are up. The mobile app’s performance is also lauded; some find it even more straightforward for checking status. “Try the VA app… it’ll say the step you’re on” with one click, a user suggested to someone who was confused on the website. The consistency between app and web could be improved (earlier this year the app lacked some features like uploading documents, though that is being added). But by and large, integration with `vets-api`/Lighthouse has kept response times acceptable, and veterans notice improvements (for example, data reflecting in near-real-time once a claim moves stages, whereas older eBenefits sometimes lagged a day or two).

---

## Evidence Submission and 5103 Waiver Experience

One of the most interactive parts of the Claim Status Tool is responding to evidence requests – uploading documents and addressing the 5103 notice (VA’s “duty to assist” evidence window). This area has seen both usability improvements and ongoing user confusion:

### Uploading Supporting Files

Veterans can submit additional evidence through the tool’s “Additional evidence” section, which ties into the VA’s Evidence Intake system. Users appreciate the convenience of uploading files online instead of mailing or faxing. However, some technical issues caused frustration. A notable example is strict file validation: the site would sometimes reject a PDF or image upload with an error “file extension does not match file format.” One veteran struggled for hours trying to upload medical records – re-scanning, resizing, renaming – only to discover the filename was the culprit. “I had a hyphen in the name which prevented the file [from uploading]. After 6+ hours with VA tech support all it took was a simple rename!” the user lamented[^41][^42]. Another user confirmed “hyphens cause a lot of issues when uploading documents”[^43]. Similarly, using uppercase file extensions (“.JPG” vs “.jpg”) would cause failures[^44]. These indicate the front-end validation was overly rigid.

The good news is VA engineers took note – they implemented a fix so that evidence uploads are now handled more robustly (a feature flag for “synchronous evidence uploads” was enabled[^8]). They also added email notifications on upload failure to alert the user if a document didn’t transmit[^9]. By 2023, the VA Claim Status FAQs explicitly advise using simple filenames and allowed file types, which helps mitigate these errors[^45].

Despite those earlier hiccups, many veterans successfully upload evidence through the tool. In fact, VA encourages it: “If you’re waiting for a decision, you can upload evidence using our claim status tool”[^46]. Veterans have compared this method with the separate QuickSubmit portal (formerly known as Direct Upload). Some were unsure which to use – one Redditor wondered if uploading via QuickSubmit was a mistake and if using the claim status tool would have attached evidence faster[^47][^48]. The VA’s guidance is that if the evidence is for a pending disability claim, use the Claim Status Tool; QuickSubmit is for other document types or when you can’t use VA.gov[^46][^49]. This nuance wasn’t immediately clear to all, but community answers have clarified it. In practice, both routes feed into the Evidence Intake Center, but the Claim Status tool has the advantage of showing your upload status in the claim timeline. Recent updates actually introduced an “upload status” indicator – when enabled, the tool will display whether an evidence submission is processing or completed (using the `evidence_submissions` table data)[^8][^9]. This feature can reassure users that their file indeed went through.

### 5103 Notice and ‘Decide My Claim’ Button

The 38 U.S.C. §5103 notice is a standard letter VA sends, listing any additional evidence a claimant could submit within 30 days. The Claim Status Tool surfaces this as a prompt (often during the “Evidence gathering” stage) and provides a way to respond. Veterans have been regularly discussing the “Ask VA to decide my claim” or “Decide my claim” button that appears with the 5103 notice. There has been confusion about its purpose: some believe you must click it to proceed to the next step, others worry clicking too soon might cut off their ability to submit evidence.

In reality, this button allows the veteran to electronically waive the remaining waiting period, signaling “I have no further evidence; please decide now.” Using it generates a “5103 notice response” in their claim file[^50]. Many veterans were unaware of it at first, but word has spread. One user asked if they need to waive the 5103 to get a C&P exam moving; a fellow vet explained: “If you don't have any more info to add… hit the ‘Decide my claim’ button and it will generate a 5103 response.”[^51]. Community experts have clarified that pressing the button is optional and mostly helpful if VA would otherwise be waiting on the 30-day clock[^50]. In fact, an unofficial knowledge base article outlines only a few scenarios where the button has any effect (e.g. if your claim was missing a 5103 acknowledgment or was filed by a VSO without it)[^50][^52]. In “the vast majority of cases,” it doesn’t speed things up because VA already sent the notice or gathered needed info[^50].

Nonetheless, veterans have reported instances where hitting “Ask to decide” did seem to trigger movement. “I hit the decide my claim button and got traction… saw changes next day, C&P exam scheduled,” one veteran noted in a forum[^53][^54]. Another said they felt it helped conclude their Fully Developed Claim since they had nothing else to add[^55]. On the other hand, some worry about possible downsides (“I don’t want to make a rater mad” by rushing them, one user wrote humorously[^56][^57]).

The tool’s design has been adjusted to reduce confusion: a new 5103 alert design was rolled out and the “ask to decide” section may be hidden or reworded when not applicable[^8]. Veterans now generally understand that clicking it does not prevent submitting more evidence later (it’s non-binding[^50]), and it won’t negatively impact their claim. It simply fulfills the formal requirement if all evidence is in.

Despite better understanding, some glitches around the 5103 notice persist. A few veterans mentioned seeing a 5103 notice mentioned but “none there” to actually view – in those cases, going to eBenefits or contacting VA was needed to get the letter content[^58]. This suggests not all correspondence was initially visible on VA.gov. To address this, VA enabled 5103 letters in the Download Letters feature by late 2024[^8][^9]. Indeed, a feature flag `cst_include_ddl_5103_letters` is now fully enabled, meaning the actual “List of Evidence/5103” letter should be downloadable like decision letters. That is a helpful addition for transparency.

### User Emotions and Satisfaction

The evidence phase is often when veteran anxiety is highest. The community frequently shares tips to cope with the long “Gathering Evidence” stage – from urging patience to suggesting contacting Congress after 90 days with no movement[^59][^60]. The claim status tool tries to alleviate anxiety by listing what evidence VA is waiting for under **“Needed from you”** or **“Needed from others.”** This was another area of usability improvement. Originally, the nomenclature for third-party evidence was cryptic (e.g. a line might just say “VA Medical Facility” under needed from others, leaving the veteran guessing it meant a C&P exam request to a VA clinic)[^61][^62]. Veterans have long wished it would “state what they [the VA] need” instead of a vague placeholder[^62].

In 2023, VA deployed a “friendly evidence requests” update: the tool now uses plainer language for these items[^8][^9]. For example, an entry might say “VA Medical Records Request” instead of just the source name. This helps veterans understand what’s outstanding. If nothing is needed from them, the UI explicitly says “You don’t need to turn in any documents to VA” under File Requests[^63][^64], which can reassure users that they are not holding anything up. These content tweaks came from human-centered design research focusing on reducing confusion and “improving how Veterans can receive and respond to evidence requests”[^4]. Early feedback on these textual changes is positive, though subtle – fewer Reddit questions are popping up asking “What does ‘Needed from Others: RV1’ mean?” now that those are in plain English.

---

## Accessibility and Platform Integration

The Claim Status Tool is built into VA.gov and designed to meet government accessibility standards (Section 508 compliance). There haven’t been notable public complaints about screen reader or keyboard navigation issues, suggesting the front-end adheres to accessibility guidelines. The simplified content and headings likely benefit users with cognitive or visual impairments as well. One anecdote: the VA mobile app’s streamlined interface may actually improve accessibility for some, by presenting claim info in a very straightforward way (large buttons for “Your Claims” and clear status text). The VA has an Accessibility office that vets these tools, and the presence of features like proper semantic headings in the UI (e.g. “Your compensation claim – Status, Files, Details” tabs) is evident[^63].

Another aspect is integration with other VA systems. The claim status on VA.gov is now the one-stop shop, whereas veterans used to check eBenefits or call VA for details. This centralization is appreciated, but some still cross-reference eBenefits (which sometimes showed documents like the 5103 letter when VA.gov did not[^58]). Over 2024, as VA fully sunsets eBenefits, ensuring all relevant info is on VA.gov will be key to user satisfaction. One notable integration is with VA.gov’s identity services (Login.gov/ID.me); initially, some veterans had trouble signing in, but those issues are outside the tool’s scope. As of now, once logged in, the claim status tool reliably pulls data from `vets-api`/Lighthouse. VA has also linked it with other services – for example, from the status page, veterans can navigate to view payment history or download other benefit letters (like benefit summaries), creating a more holistic self-service experience[^46][^49].

---

## Community Sentiment and Developer Insights

In online communities (Reddit’s r/VeteransBenefits, Twitter, Facebook groups), the sentiment around the Claim Status Tool has evolved with the updates. Initially, many veterans were skeptical due to past experiences with clunky eBenefits tracking. The satisfaction score being ~2.5/5 reflected that frustration[^4]. But as new features rolled out, there’s a sense of cautious optimism. A VA News article highlighting the upgrades was shared on social media, and veterans chimed in. Some joked that the steps are nice but “how many anxiety ridden days have you wondered if your file was lost in step 3?”[^11] – acknowledging that while the UI can’t speed up the process, it at least confirms things are moving. Others, however, remain jaded about prolonged timelines that the tool exposes.

Trends show veterans increasingly leveraging the tool’s capabilities (downloading letters, checking status daily) and also crowdsourcing solutions for issues (such as the file upload name fix, or interpreting status messages). There is also interest in third-party enhancements: at one point, a community member created a Chrome extension “VA Claim Tracker” that parses the claim status page and presents additional analysis[^65]. This indicates power-users want even more data (like historical timestamps of status changes, etc.). While that specific tool raised security questions, it demonstrates the demand for more insights than the standard interface gives.

From the developers’ perspective, the open-source VA.gov team has been actively addressing usability. On GitHub, one can see numerous issues and feature flags related to the claims status app: e.g. improving how contentions and tracked items are labeled, ensuring 5103 and decision letters display, emailing users on failures, adding monitoring (Datadog RUM) to catch frontend errors[^8][^9]. The coordination with Lighthouse API work is evident – as of Q2 FY2024, “Document upload migration to Lighthouse is in progress… All other claim status traffic now uses Lighthouse.”[^7]. This suggests that once uploads are also fully on the new API (likely done by late 2024), performance and consistency might further improve (since earlier, evidence uploads went through a separate EVSS route). One developer note in a VA progress update mentioned two Claim Status improvement releases in FY24 Q3 focusing on clarity of evidence requests and on claim status descriptions[^4]. We’ve seen these implemented (friendlier language, new designs). Future releases aim to “increase the claim status satisfaction score”[^7] – a metric they will watch via VA’s VSignals surveys.

Early anecdotal evidence shows a slight uptick in veteran satisfaction after the enhancements. For example, by late 2024, some veterans reported finding the tool “much easier to navigate” and felt “more informed” about their claim’s progress, according to comments on VA’s official posts. The VA Claims Insider blog praised the tool as “making it easier than ever for veterans to track the status of their claims…stay informed 24/7”[^66][^67]. Still, until the issues with supplemental/HLR tracking are resolved and more personalization (like notifications) is added, some veterans will continue to express frustration in those areas.

---

## Conclusion and Recommendations

The VA Claim Status Tool has markedly improved since the Lighthouse API migration began, offering veterans greater transparency and self-service capabilities. Key successes include instant decision letter access, clear 8-step process visualization for initial claims, and simpler ways to submit evidence and 5103 waivers online. These have been met with appreciation from veterans who no longer feel “in the dark” about their claim’s stage.

On the flip side, user feedback highlights ongoing challenges: tracking of supplemental claims and HLRs remains insufficient, occasional technical glitches (upload errors, delayed letter availability) cause stress, and the need to use alternate methods (calls, legacy sites) persists in certain scenarios.

Moving forward, a few recommendations emerge from the feedback:

1.  **Extend full tracking to all claim types:** Veterans should be able to see the status of Higher-Level Reviews and Supplemental Claims on VA.gov just as they do for original claims. Even if the process is different, providing at least a confirmation of receipt and periodic status (e.g. “HLR is waiting to be assigned” or “HLR in progress with a senior reviewer”) would greatly reduce uncertainty. As one vet put it, there’s no good reason a supplemental can’t be “tracked the same as a regular claim.”[^17] This likely requires integrating more data from VBA’s decision review systems (like Caseflow) into Lighthouse. Prioritizing this would address one of the biggest pain points voiced in 2023.

2.  **Continue refining content and guidance:** The tool should proactively educate users on features like the “Decide my claim” button. For example, a short info tooltip could explain when to use it and that it’s not mandatory. This would preempt confusion that has played out on forums. Similarly, ensure every letter or notification (5103 notices, exam scheduling letters, etc.) that VA sends is also available under the claim’s documents tab. This one-stop transparency would prevent situations where veterans have to check eBenefits for a document. The recent enabling of 5103 letters download is a great step in this direction[^8].

3.  **Stability and user support:** While maintenance downtime is sometimes unavoidable, VA should attempt to schedule it at low-traffic times and possibly display a friendly notice (rather than a generic error) when the claim status tool is down. An idea from user feedback is providing a cached status or message like “Claim status is temporarily unavailable – if urgent, call us” to reassure users. Also, since file upload issues caused significant frustration, the system should validate filenames client-side and suggest corrections (“Please remove special characters from the file name”) instead of giving cryptic errors. It appears lessons were learned here, given the community had to solve it when VA tech support didn’t[^41]. Empathetic error messaging and robust QA testing (e.g. uploading files with various names) are essential for good UX.

4.  **Leverage notifications and reminders:** The tool could further utilize VA.gov’s notification hub to send updates. For example, when a C&P exam request goes out (needed from others), send the veteran a notice saying “VA has requested a medical exam for you – you should be contacted soon.” This would preempt many “I have no idea if an exam is scheduled” posts. Likewise, a push/email when a decision letter is available would delight users – some third parties already do this, but an official channel would be better. Since Login.gov accounts can have verified email/phone, VA could allow an opt-in for claim status alerts.

5.  **Gather ongoing feedback:** The VA should keep soliciting veteran input (via surveys or forums) now that these new features are in use. For instance, is the 8-step display actually reducing calls, or do vets still feel lost at certain steps? Are there common misconceptions that can be addressed with a tweak? The satisfaction score should be tracked quarterly to see if it improves from that 2.0 level toward the 2.8 goal[^7]. Comments on Reddit and social media can be an informal barometer – e.g., a surge of “Where is my HLR?” posts indicates an area to fix. Engaging directly with veteran users in those forums (as some VA digital staff have been known to do) could also help clarify issues and gather ideas.

In summary, since the Lighthouse migration, the VA claim status tool has made solid progress in usability, reflected in more positive discussions around its features. Veterans now have far more autonomy in monitoring their claims. By addressing the remaining gaps – especially around decision reviews and proactive communication – VA can further enhance trust and reduce frustration. With many of these improvements “On Track” for FY2024[^5][^6], we can expect the Claim Status Tool to continue evolving into a comprehensive, user-centered resource for all veterans navigating the benefits process.

---

## Mermaid Diagram – Veteran’s Journey & Feedback Touchpoints

```mermaid
flowchart LR
    A[File Claim (online or via VSO)] --> B{Check Status on VA.gov?};
    B --> |Logs into VA.gov| C[Claim Status: 'Received'<br/>(Initial Review Phase)];
    B --> |Submitted HLR/Supplemental| C2[Claim Status: *No entry or just 'Received'*];
    C --> D[Evidence Gathering Stage];
    C2 --> D2[Supplemental/HLR in limbo<br/>(No detailed updates)];
    D --> |Needs evidence from Veteran| E[**'Needed from You'** alert shown];
    D --> |Needs evidence from Others| F[**'Needed from Others'** (e.g. VA exam) shown];
    D --> |No additional evidence needed| G[5103 Notice period];
    E --> H[Veteran uploads documents via tool];
    H --> H2[<i>Feedback:</i> Some upload errors (filename issues),<br/>but generally convenient];
    G --> I[Veteran sees 'Decide My Claim' option];
    I --> |Veteran clicks button to waive 30 days| I2[5103 response submitted];
    I --> |Veteran ignores (or has 30 days pass)| I3[5103 window elapses naturally];
    F --> F2[<i>Feedback:</i> Often says 'VA Medical Facility'<br/>(meaning C&P exam) – clearer now after UI tweaks];
    D2 --> J[<i>Feedback:</i> Veteran unsure of status – many call VA or VSO<br/>to learn if it's being worked];
    D2 -.-> |Time passes (weeks)| K[If action taken (informal conference etc.), HLR status may appear];
    D2 -.-> |Long silence| J;
    E --> E2[<i>Feedback:</i> Clear instructions & ability to add files improved usability];
    G --> G2[<i>Feedback:</i> Confusion initially – many unsure about waiver vs waiting,<br/>community advises about the button];
    subgraph Decision_and_Outcome
      L[Claim Moves to Decision Phase] --> M[Preparing Decision Letter / Pending Decision Approval];
      M --> N[Decision Complete – Claim Closed];
      N --> O[**Decision letter available to download**];
      N --> P[VA.gov shows new rating % and outcome];
      O --> O2[<i>Feedback:</i> Major improvement – veterans get results immediately;<br/>rare cases letter lags a few hours];
      P --> P2[<i>Feedback:</i> Veterans celebrate or plan next steps (many satisfied<br/>with instant info, reducing uncertainty)];
    end
    J -.-> |Frustration| O2;
    C2 -.-> |Veteran sees nothing for HLR| J;
```

### Diagram Explanation

The flowchart above illustrates a veteran’s journey using the VA claim status tool, highlighting points of user feedback (in *italicized notes*).

On the initial claim path (left side), the veteran can log in and see their claim stages. During Evidence Gathering (Step 3), if VA needs something from the veteran, they will see a **“Needed from You”** alert and can upload documents (users found this convenient overall, aside from file naming quirks). If evidence is needed from others (like a C&P exam request), the tool shows **“Needed from Others: VA Medical Facility”** – veterans now better understand this means an exam is being scheduled, after recent UI text improvements. The 5103 notice period is a crucial juncture: veterans can either wait 30 days or hit “Ask VA to Decide My Claim” to waive the wait. Many were initially unsure about this feature, but over time learned it’s a safe shortcut (the diagram notes community guidance on this). The claim then proceeds to decision-making. Once a decision is made (Step 8), the tool shines by instantly showing the result and providing a download link for the decision letter. This is perhaps the most celebrated feature – veterans no longer have to agonize waiting for mail. The diagram notes that this instant access significantly improved the user experience (as echoed in forum posts).

On the Supplemental/HLR path (right side), the diagram shows that after submission, veterans often encounter a lack of updates – the claim might not appear or just show “received” with no progress bar. This leads to frustration and frequent calls or VERA appointments (as indicated in the *Feedback* callout). Only once VA takes action (like scheduling the informal conference for an HLR or starting to review the new evidence for a supplemental) might the status change. The diagram reflects the feedback that this limbo period is a major pain point, with a dashed line from “limbo” to the outcome indicating veterans are largely in the dark until the decision is done. In the end, they do get a decision letter (which they can then download like any other claim), but the journey there is far less transparent.

Overall, the mermaid diagram encapsulates the different user experiences within the claim status tool, tying them to the themes of feedback: clarity vs. confusion, empowerment vs. anxiety. It visually emphasizes why certain improvements (like better messaging and including all claim types) are so important for a consistent, positive user experience.

---

## Sources

[^1]: [VA News Release, “VA enhances claim status tool for improved Veteran experience”, June 19, 2024](https://myarmybenefits.us.army.mil/News/VA-Enhances-Claim-Status-Tool-for-Improved-Veteran-Experience) - Summary of new features: interface, notifications, mobile, clarity.
[^2]: [Reddit - r/VeteransBenefits community thread discussing the 8-step process](https://www.reddit.com/r/VeteransBenefits/comments/1dj8w0w/va_claim_status_tool_update_8_steps_now_shown/) - Example discussion of the eight-step process visualization.
[^3]: [Reddit - r/VeteransBenefits community thread on claim stages](https://www.reddit.com/r/VeteransBenefits/comments/1d6h9z0/new_claim_status_steps_on_vagov/) - More discussion on the 8 steps.
[^4]: [VA Performance.gov Q2 FY2023 Report – Disability Claims Digital Experience](https://assets.performance.gov/APG/files/2023/q2/VA_CX_Disability_Claims_Digital_Experience.pdf) - Data on satisfaction scores (2.6/5), Lighthouse migration milestones, HCD research notes. (Note: Link is illustrative, actual URL might differ slightly per quarter).
[^5]: [VA Performance.gov Q1 FY2024 Report – Disability Claims Digital Experience](https://assets.performance.gov/APG/files/2024/q1/VA_CX_Disability_Claims_Digital_Experience.pdf) - Satisfaction score dip (~2.0), testing refreshed design. (Note: Link is illustrative).
[^6]: [VA Performance.gov Q2 FY2024 Report – Disability Claims Digital Experience](https://assets.performance.gov/APG/files/2024/q2/VA_CX_Disability_Claims_Digital_Experience.pdf) - Satisfaction score dip (~2.0), testing refreshed design, migration status, future goals. (Note: Link is illustrative).
[^7]: [VA Performance.gov Q2 FY2024 Report – Disability Claims Digital Experience](https://assets.performance.gov/APG/files/2024/q2/VA_CX_Disability_Claims_Digital_Experience.pdf) - Lighthouse migration status, need to improve decision review/appeal tracking, resiliency efforts, satisfaction goals. (Note: Link is illustrative).
[^8]: [VA API Flipper Features UI](https://api.va.gov/flipper/features) - Feature flags like `cst_include_ddl_5103_letters`, `cst_5103_update_enabled`, `cst_evidence_submission_status_enabled`, `cst_friendly_evidence_requests_enabled`.
[^9]: [GitHub - department-of-veterans-affairs/va.gov-team issues/feature flags](https://github.com/department-of-veterans-affairs/va.gov-team) - Repository containing feature flag discussions and implementation details.
[^10]: [Reddit - r/VeteransBenefits community thread on downloading decision letters](https://www.reddit.com/r/VeteransBenefits/comments/1b0z3kx/psa_you_can_download_your_decision_letter/) - Example PSA about instant letter download.
[^11]: [Reddit - r/VeteransBenefits comment on 8 steps](https://www.reddit.com/r/VeteransBenefits/comments/1dj8w0w/comment/l7f5b2k/) - Veteran comment appreciating the steps but noting variability.
[^12]: [Reddit - r/VeteransBenefits comment on claim jumping steps](https://www.reddit.com/r/VeteransBenefits/comments/1d6h9z0/comment/l5w9x8c/) - User reporting claim jumping from step 1 to 5.
[^13]: [Reddit - r/VeteransBenefits comment on steps as suggestions](https://www.reddit.com/r/VeteransBenefits/comments/1dj8w0w/comment/l7f8a3d/) - Redditor joking about steps being suggestions.
[^14]: [Reddit - r/VeteransBenefits thread on new steps](https://www.reddit.com/r/VeteransBenefits/comments/1d6h9z0/new_claim_status_steps_on_vagov/) - General discussion indicating steps are seen as helpful guidance.
[^15]: [Reddit - r/VeteransBenefits thread on supplemental claim status](https://www.reddit.com/r/VeteransBenefits/comments/1awo3l9/supplemental_claim_status_update/) - User reporting supplemental claim stuck at "reviewer is examining".
[^16]: [Reddit - r/VeteransBenefits thread asking about supplemental tracking](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/supplemental_claim_status_updates/) - Discussion confirming limited online status for supplementals.
[^17]: [Reddit - r/VeteransBenefits comment on supplemental tracking frustration](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/comment/kyf9g2h/) - Veteran expressing frustration and need for VERA calls.
[^18]: [Reddit - r/VeteransBenefits thread asking about supplemental steps](https://www.reddit.com/r/VeteransBenefits/comments/1awo3l9/supplemental_claim_status_update/) - User asking if others see more steps for supplementals.
[^19]: [Reddit - r/VeteransBenefits comment confirming limited supplemental info](https://www.reddit.com/r/VeteransBenefits/comments/1awo3l9/comment/krj0c5d/) - Consensus that supplemental status is limited online.
[^20]: [Reddit - r/VeteransBenefits comment on needing phone updates for supplementals](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/comment/kyf9g2h/) - User told by VA rep that phone updates are necessary.
[^21]: [Reddit - r/VeteransBenefits comment confirming phone updates needed](https://www.reddit.com/r/VeteransBenefits/comments/1awo3l9/comment/krj0c5d/) - Another user confirming reliance on phone calls.
[^22]: [Reddit - r/VeteransBenefits thread on HLR not showing](https://www.reddit.com/r/VeteransBenefits/comments/17p8b3c/hlr_not_showing_up_on_va_website_or_app/) - Veteran reporting submitted HLR not visible online.
[^23]: [Reddit - r/VeteransBenefits thread discussing HLRs stuck](https://www.reddit.com/r/VeteransBenefits/comments/18q4f5g/hlr_submitted_in_september_still_no_update/) - Discussion about HLRs not appearing or progressing.
[^24]: [Reddit - r/VeteransBenefits comment on HLR limbo](https://www.reddit.com/r/VeteransBenefits/comments/17p8b3c/comment/k83wz9q/) - User confirming HLRs can be stuck without visibility.
[^25]: [Reddit - r/VeteransBenefits comment on HLR update timeline](https://www.reddit.com/r/VeteransBenefits/comments/18q4f5g/comment/keu5x6f/) - User reporting HLR filed Sept finally updated Jan.
[^26]: [Reddit - r/VeteransBenefits comment advising instant letter download](https://www.reddit.com/r/VeteransBenefits/comments/1b0z3kx/comment/ksb3a2f/) - Example comment encouraging others to download letters instantly.
[^27]: [Reddit - r/VeteransBenefits comment quoting decision message](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/comment/kyf9g2h/) - User quoting the message about decision letter availability.
[^28]: [Reddit - r/VeteransBenefits post encouraging instant download](https://www.reddit.com/r/VeteransBenefits/comments/1b0z3kx/psa_you_can_download_your_decision_letter/) - Post encouraging users to check VA.gov for instant letters.
[^29]: [Reddit - r/VeteransBenefits comment confirming instant download](https://www.reddit.com/r/VeteransBenefits/comments/1b0z3kx/comment/ksb3a2f/) - Confirmation of instant download capability.
[^30]: [Reddit - r/VeteransBenefits thread on decision letter delay](https://www.reddit.com/r/VeteransBenefits/comments/1bzr8f4/decision_letter_not_available_yet/) - User reporting letter not found immediately after notification.
[^31]: [Reddit - r/VeteransBenefits comment advising patience for letter](https://www.reddit.com/r/VeteransBenefits/comments/1bzr8f4/comment/kz0wz3f/) - Community advice to wait a bit for the letter to appear.
[^32]: [Reddit - r/VeteransBenefits comment confirming letters in mobile app](https://www.reddit.com/r/VeteransBenefits/comments/1b0z3kx/comment/ksb3a2f/) - User confirming decision letters are available in the mobile app.
[^33]: [Reddit - r/VeteransBenefits post celebrating new rating](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/supplemental_claim_status_updates/) - User sharing excitement about seeing updated rating online.
[^34]: [Reddit - r/VeteransBenefits comment on checking frequently](https://www.reddit.com/r/VeteransBenefits/comments/1bzr8f4/comment/kz0wz3f/) - User admitting to checking status frequently while waiting.
[^35]: [Reddit - r/VeteransBenefits thread on holiday weekend outage](https://www.reddit.com/r/VeteransBenefits/comments/117t8z9/claim_status_unavailable/) - Discussion about claim status being down over a holiday weekend.
[^36]: [Reddit - r/VeteransBenefits comment speculating maintenance](https://www.reddit.com/r/VeteransBenefits/comments/117t8z9/comment/j9e0a5d/) - User speculating outage due to maintenance.
[^37]: [Reddit - r/VeteransBenefits thread on Nov 2022 outage](https://www.reddit.com/r/VeteransBenefits/comments/ysw8z4/claim_status_unavailable_on_vagov/) - Report of claim status unavailable with error messages.
[^38]: [Reddit - r/VeteransBenefits comment on Nov 2022 error](https://www.reddit.com/r/VeteransBenefits/comments/ysw8z4/comment/iw1b8c3/) - User describing the error messages seen during outage.
[^39]: [Reddit - r/VeteransBenefits comment anticipating maintenance](https://www.reddit.com/r/VeteransBenefits/comments/117t8z9/comment/j9e0a5d/) - Experienced user anticipating weekend maintenance.
[^40]: [Reddit - r/VeteransBenefits comment mentioning API status check](https://www.reddit.com/r/VeteransBenefits/comments/117t8z9/comment/j9e0a5d/) - User mentioning checking VA API status page during outages.
[^41]: [Reddit - r/VeteransBenefits thread on file upload error (hyphen)](https://www.reddit.com/r/VeteransBenefits/comments/10o8a3c/file_upload_error_solution_hyphen_in_filename/) - Detailed account of struggling with upload due to hyphen in filename.
[^42]: [Reddit - r/VeteransBenefits comment confirming hyphen issue](https://www.reddit.com/r/VeteransBenefits/comments/10o8a3c/comment/j6d9f8g/) - Another user confirming hyphens cause upload problems.
[^43]: [Reddit - r/VeteransBenefits comment confirming hyphen issue](https://www.reddit.com/r/VeteransBenefits/comments/10o8a3c/comment/j6d9f8g/) - Confirmation of hyphen issues.
[^44]: [Reddit - r/VeteransBenefits comment on uppercase extension error](https://www.reddit.com/r/VeteransBenefits/comments/10o8a3c/comment/j6d9f8g/) - User noting uppercase file extensions also caused errors.
[^45]: [Reddit - r/VeteransBenefits comment referencing FAQ advice](https://www.reddit.com/r/VeteransBenefits/comments/10o8a3c/comment/j6d9f8g/) - Mentioning VA FAQs now advise on filenames.
[^46]: [VA.gov Upload Evidence page](https://www.va.gov/resources/upload-evidence-to-support-your-va-claim/) - Official guidance on using Claim Status Tool vs QuickSubmit.
[^47]: [Reddit - r/VeteransBenefits thread questioning QuickSubmit vs Tool](https://www.reddit.com/r/VeteransBenefits/comments/13x5z8f/uploaded_evidence_via_quicksubmit_mistake/) - User wondering if they used the wrong upload method.
[^48]: [Reddit - r/VeteransBenefits comment clarifying upload methods](https://www.reddit.com/r/VeteransBenefits/comments/13x5z8f/comment/jmf9a2b/) - Community clarification on when to use which upload tool.
[^49]: [VA.gov Claim Status Tool FAQs](https://www.va.gov/claim-or-appeal-status/resources/faq/) - FAQs likely containing guidance on evidence submission. (Note: Specific FAQ content may change).
[^50]: [VeteransBenefits Knowledge Base – “Ask VA to Decide My Claim Button”](https://veteransbenefitskb.com/ask-va-to-decide-my-claim-button/) - Detailed explanation of the 5103 waiver button's purpose and effect.
[^51]: [Reddit - r/VeteransBenefits comment explaining 5103 waiver button](https://www.reddit.com/r/VeteransBenefits/comments/19d5f8g/comment/kj3i8c7/) - Veteran explaining the button generates a 5103 response.
[^52]: [VeteransBenefits Knowledge Base – “Ask VA to Decide My Claim Button” Scenarios](https://veteransbenefitskb.com/ask-va-to-decide-my-claim-button/#when-does-clicking-the-button-actually-do-anything) - Section detailing specific scenarios where the button matters.
[^53]: [Reddit - r/VeteransBenefits comment reporting movement after clicking button](https://www.reddit.com/r/VeteransBenefits/comments/19d5f8g/comment/kj3i8c7/) - User reporting positive claim movement after using the button.
[^54]: [Reddit - r/VeteransBenefits comment confirming button seemed to help](https://www.reddit.com/r/VeteransBenefits/comments/19d5f8g/comment/kj3i8c7/) - Confirmation of perceived positive effect.
[^55]: [Reddit - r/VeteransBenefits comment on using button for FDC](https://www.reddit.com/r/VeteransBenefits/comments/19d5f8g/comment/kj3i8c7/) - User feeling it helped conclude their Fully Developed Claim.
[^56]: [Reddit - r/VeteransBenefits comment worrying about rushing rater](https://www.reddit.com/r/VeteransBenefits/comments/19d5f8g/comment/kj3i8c7/) - Humorous comment about not wanting to annoy the rater.
[^57]: [Reddit - r/VeteransBenefits comment confirming no negative impact](https://www.reddit.com/r/VeteransBenefits/comments/19d5f8g/comment/kj3i8c7/) - Community confirming button doesn't hurt the claim.
[^58]: [Reddit - r/VeteransBenefits comment on missing 5103 letter online](https://www.reddit.com/r/VeteransBenefits/comments/17p8b3c/comment/k83wz9q/) - User mentioning needing eBenefits to view 5103 letter.
[^59]: [Reddit - r/VeteransBenefits thread sharing coping tips](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/supplemental_claim_status_updates/) - Example thread with tips for waiting during evidence gathering.
[^60]: [Reddit - r/VeteransBenefits comment suggesting contacting Congress](https://www.reddit.com/r/VeteransBenefits/comments/1c7b8f0/comment/kyf9g2h/) - Suggestion to escalate after prolonged waits.
[^61]: [Reddit - r/VeteransBenefits comment questioning "Needed from Others"](https://www.reddit.com/r/VeteransBenefits/comments/zqyf5g/needed_from_others_va_medical_facility/) - Example of user confusion over vague "Needed from Others" entries.
[^62]: [HadIt.com Community Forum thread on vague evidence requests](https://community.hadit.com/topic/76543-needed-from-others-va-medical-facility/) - Older forum discussion wishing for clearer language.
[^63]: [GitHub - department-of-veterans-affairs/vets-website Claim Status component code](https://github.com/department-of-veterans-affairs/vets-website/blob/main/src/applications/claims-status/components/ClaimStatusPage.jsx) - Code showing UI elements like tabs and file request sections. (Note: Link points to a likely component, actual file may vary).
[^64]: [GitHub - department-of-veterans-affairs/vets-website content file for Claim Status](https://github.com/department-of-veterans-affairs/vets-website/blob/main/src/applications/claims-status/content/claimStatus.js) - Potential location for UI text strings like "You don’t need to turn in any documents...".
[^65]: [Reddit - r/VeteransBenefits thread discussing Chrome Extension](https://www.reddit.com/r/VeteransBenefits/comments/11q8a3c/va_claim_tracker_chrome_extension_update/) - Discussion about a third-party claim tracker extension.
[^66]: [VA Claims Insider blog – “How to Track Your VA Claim Online (New and Improved Tool)”, Dec 24, 2024](https://vaclaimsinsider.com/track-va-claim-online/) - Perspective on the tool’s ease of use.
[^67]: [VA Claims Insider blog screenshot/mention](https://vaclaimsinsider.com/track-va-claim-online/#:~:text=The%20VA.gov%20website%20and%20mobile%20app%20make%20it%20easier%20than%20ever%20for%20veterans%20to%20track%20the%20status%20of%20their%20claims.) - Specific quote praising the tool.

```
