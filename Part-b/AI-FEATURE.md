# AI Feature Specification: Predictive Intelligence Engine for Waitlist Confirmation Probabilities

## Problem It Solves
When preferred trains sell out, users are left with ambiguous waitlist markers (e.g., `WL 32 / WL 14`). Without any context, passengers have no way of knowing if their ticket will actually clear. This uncertainty leads to high anxiety, booking abandonment, or double-bookings that lock up inventory. This feature directly addresses **Problem 3 (Waitlist Tracking Friction)** by turning guesswork into clear, actionable probabilities.

## Proposed Feature — User Perspective
When browsing trains or reviewing a waitlisted ticket, an intelligent inline badge automatically appears next to the waitlist number (e.g., **"94% Confirmation Chance"**). 

The system categorizes these chances with simple, color-coded indicators:
* 🟢 **High ( > 80%)**
* 🟡 **Medium (50% – 80%)**
* 🔴 **Low ( < 50%)**

Tapping this indicator opens an explanatory panel with clear, helpful context: *"Tickets on this route typically clear up to WL 45 during summer months due to business travel adjustments. We recommend proceeding with your booking."*

WAITLIST AI PREDICTION COMPONENT — UI Layout
─────────────────────────────────────────────────
TRAIN DETAILS: 12622 TAMIL NADU EXPRESS
📅 Date: 16 June 2026 | Quota: General

AVAILABLE SEATING OPTIONS:
┌───────────────────────────────────────────┐
│ SLEEPER CLASS (SL)                        │
│ Status: WL 24                             │
│ 🟢 92% CONFIRMATION CHANCE                │
│ (AI: High probability of clearing)        │
└───────────────────────────────────────────┘
👉 Why this estimation?
Our predictive model analyzed 4 years of seasonal
trends for Train 12622. Cancellations spike by
~15% in the final 48 hours before departure.
─────────────────────────────────────────────────


## Model or API Choice
We will deploy a custom-trained **XGBoost Classifier Model** instead of generic LLM wrappers like OpenAI GPT-4. 
* **Why?** LLMs are poorly suited for high-volume, exact numerical and tabular time-series forecasts, and they incur high latency and API token costs. 
* An XGBoost model trained on structured history delivers highly accurate classification boundaries, operates with sub-15ms execution latencies, and can scale cost-effectively across millions of daily user requests.

## Training or Input Data
The model runs on historical, anonymized booking and cancellation datasets owned by Indian Railways:
1.  **Historical Trend Data:** Final confirmation status logs of waitlisted tickets from the past 5 years.
2.  **Temporal Inputs:** Day of the week, holiday proximity indicators, seasonal factors, and weather disruptions.
3.  **Route Context:** Train number, vehicle class code, origin/destination pairs, and current days left until departure.

This data is already captured inside internal relational databases and can be safely extracted into training pipelines.

## How Output Is Shown to the User
The estimation output renders as a lightweight, reactive UI badge nested inside the train availability layout card:
* **CSS Class Architecture:** Utilizes clean, text-based conditional formatting to ensure optimal performance on low-end smartphones or poor 2G internet connections.
* **Data Layout:** Avoids heavy layout reflows or blocking elements, integrating cleanly into standard accessibility screen-readers.

## Confidence Threshold and Fallback
To ensure maximum reliability, we enforce strict confidence boundaries:
* **Verification Boundary:** If the model's confidence rating drops below **70%**, the prediction badge gracefully hides itself entirely. The interface scales back to the standard status text (`WL 24`) without confusing the user.
* **System Failure Recovery:** If the inference API service drops or experiences an error, the UI skips the prediction block completely and logs a silent telemetry report.

## Success Metrics
* Uncertainty-driven user booking abandonments drop by 38%.
* Alternate route search conversions increase by 24% when users see low-probability indicators early on.

## Limitations and Risks
* **Unexpected Real-World Fluctuation:** Sudden external events (like unexpected wedding season rushes, sudden festival dates, or unexpected weather changes) can break historical cancellation trends.
* **User Reliance Risk:** A passenger might buy a low-chance ticket if the model estimates a high probability, and still end up stuck if an unusual drop in cancellations occurs. 
* **Mitigation Policy:** The interface explicitly includes a clear, supportive legal disclaimer text block: *"This score is an intelligent historical estimate to help plan your trip; it does not guarantee final board-ready seat allocation."*