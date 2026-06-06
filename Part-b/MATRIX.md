# Product Prioritisation: 2×2 Impact vs Effort Matrix

## The Matrix

|                   | Low Effort (Fast to Build) | High Effort (Complex Architectures) |
|-------------------|----------------------------|-------------------------------------|
| **High Impact** | **QUICK WINS** | **MAJOR PROJECTS** |
|                   | • Spec 4: Sticky Search    | • Spec 1: High-Concurrency Queue    |
|                   | • Spec 2: Form Persistence | • Spec 5: Inline Biometric Payment   |
|                   | • Spec 6: Offline Ticket   | • Spec 3: Multi-Channel Alerts      |
|                   |                            | • AI-Feature: WL Predictor          |
|-------------------|----------------------------|-------------------------------------|
| **Low Impact** | **FILL-INS** | **TIME SINKS** |
|                   | None                       | None                                |

---

## Pre-Placement Scoring Table

| Feature Spec Reference | Target User Base Impact (1-5) | Engineering Complexity (1-5) | Assigned Strategic Quadrant |
|:---|:---:|:---:|:---|
| **Spec 1:** Tatkal Token Queueing | 5 | 5 | Major Project |
| **Spec 2:** Client Form State Persistence | 4 | 2 | Quick Win |
| **Spec 3:** Distributed PNR Push Alerts | 4 | 4 | Major Project |
| **Spec 4:** Sticky Global Search State | 4 | 1 | Quick Win |
| **Spec 5:** Native Biometric Payment Intent | 5 | 4 | Major Project |
| **Spec 6:** Offline PNR Ticket Hydration | 4 | 2 | Quick Win |
| **AI-Feature:** Waitlist Probability Engine | 5 | 5 | Major Project |

---

## Placement Justifications

### Spec 1: High-Concurrency Virtual Token Queueing — Major Project
* **Impact:** Massive. This feature addresses peak Tatkal traffic spikes, directly benefiting millions of users encountering infrastructure errors every morning.
* **Effort:** High. Implementing this requires re-engineering reverse proxy entry nodes and deploying optimized Redis clusters to manage real-time queues.
* **Prioritisation Meaning:** A high-priority operational requirement that must be properly planned, budgeted, and load-tested before deployment.

### Spec 2: Client-Side Form State Persistence — Quick Win
* **Impact:** High. This completely eliminates the frustration of losing passenger data during sudden application unmounts or crashes.
* **Effort:** Low. The system uses local device storage APIs (IndexedDB) without requiring any changes to backend APIs or databases.
* **Prioritisation Meaning:** An immediate implementation candidate that delivers significant value with minimal development overhead.

### Spec 3: Distributed Multi-Channel Waitlist Push Alerts — Major Project
* **Impact:** High. This saves millions of users from having to repeatedly open the app to manually verify their PNR travel updates.
* **Effort:** High. This requires setting up asynchronous background message brokers (Kafka) and integrating with external enterprise WhatsApp APIs.
* **Prioritisation Meaning:** Build this immediately after stabilizing the core transactional engine to significantly reduce inbound lookup traffic.

### Spec 4: Sticky Global Search Parameter State Caching — Quick Win
* **Impact:** High. This streamlines navigation across the app, ensuring travel criteria are retained when users move between views.
* **Effort:** Minimal. This is a client-side state adjustment that takes a front-end developer only a few hours to configure.
* **Prioritisation Meaning:** Ship this immediately in the next minor version release to provide an instant user experience improvement.

### Spec 5: Automated Inline Biometric Payment Intent Flow — Major Project
* **Impact:** Critical. This cuts down booking time by eliminating slow banking redirects and OTP copying steps during time-sensitive windows.
* **Effort:** High. This requires setting up secure handshakes with native device hardware systems and payment gateways.
* **Prioritisation Meaning:** A highly impactful project that should be co-developed alongside partner banking institutions.

### Spec 6: Intelligent Offline PNR Data Hydration Engine — Quick Win
* **Impact:** High. This ensures passengers can always pull up their travel confirmation passes on the train, even in zero-signal areas.
* **Effort:** Low. This is achieved by utilizing standard client-side service workers and structured storage cache frameworks.
* **Prioritisation Meaning:** A reliable, low-effort feature that noticeably enhances trust in the application during journeys.

### AI-Feature: Predictive Intelligence Engine for Waitlists — Major Project
* **Impact:** Very High. This introduces clear, data-driven transparency, helping users confidently plan alternate arrangements when waitlists are unlikely to clear.
* **Effort:** High. This involves clean dataset preparations, training custom XGBoost models, and setting up dedicated inference servers.
* **Prioritisation Meaning:** A strategic long-term capability that should be introduced as a beta feature on high-traffic routes first.

---

## Recommended Sprint Execution Order

1.  **Phase 1 (Sprint 1 - Quick Wins Delivery):** Implement **Spec 4 (Sticky Search Caching)** and **Spec 2 (Form Persistence)** immediately to instantly improve user workflow reliability.
2.  **Phase 2 (Sprint 2 - Offline Readiness):** Ship **Spec 6 (Offline Ticket Viewer)** to ensure reliable access to boarding passes mid-journey.
3.  **Phase 3 (Sprint 3 - Core Scale Infrastructure):** Build and deploy **Spec 1 (High-Concurrency Virtual Queue)** to handle heavy Tatkal booking traffic.
4.  **Phase 4 (Sprint 4 - Payment & Alerts Enhancement):** Launch **Spec 5 (Biometric Pay)** and **Spec 3 (Multi-Channel Notifications)** to simplify the checkout and post-booking experience.
5.  **Phase 5 (Sprint 5 - Predictive Intelligence):** Deploy the **AI Waitlist Probability Predictor** once the data pipelines and inference clusters are fully optimized.