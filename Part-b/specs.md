# IRCTC Feature Specifications — Part B

> All specs reference problems documented in `part-a/PROBLEMS.md`.
> Platform: irctc.co.in · Explored: June 2026 · Author: Dhruv Malviya

---

## Feature Spec 1: Virtual Queue for Tatkal Booking Window

### Problem Statement
Every day at 10:00 AM, the IRCTC server collapses under the simultaneous load of hundreds of thousands of Tatkal booking attempts, as documented in Part A — Problem 1. Users receive no feedback — only frozen screens, opaque error codes, or silent session timeouts — and by the time the system recovers, Tatkal quota is fully exhausted. The consequence is that users who prepared, logged in early, and filled all passenger details are indistinguishable from users who arrived late — the crash punishes effort and rewards luck.

### Current State (from Part A)
As documented in Part A, Steps 7–10 of the Tatkal flow are the failure point. The user fills passenger details, completes CAPTCHA, and clicks "Continue" at 10:00 AM — at which point the seat allocation engine is simultaneously hit by hundreds of thousands of requests. Lock contention causes the database query queue to balloon; the web front-end has no graceful degradation and simply drops or times out requests. Users either see a frozen spinner, an "Error 109," or a "503 Service Unavailable" — with no indication of whether to wait, retry, or give up. By the time they attempt any action, the Tatkal quota shows "REGRET."

### Proposed Solution
When the booking system is at capacity, instead of crashing, IRCTC places incoming users into a visible virtual queue. The user sees their position, an estimated wait time, and a live progress bar — all updating automatically. When it is their turn, the system advances them into the booking flow without any action needed from their side. If they disconnect (phone call, network drop), their position is held for 10 minutes and they receive a notification when they are close to the front. The experience turns a chaotic race into a managed, fair line.

### Proposed User Flow — Step by Step
1. User opens irctc.co.in at 9:50 AM and logs in (CAPTCHA improved per Spec 4).
2. User fills train search form with TATKAL quota and clicks "Search Trains."
3. User clicks "Book Now" on the target train and begins filling passenger details.
4. At 10:00 AM, user clicks "Continue" to proceed to seat selection.
5. If concurrent sessions exceed the system threshold, user is placed in a virtual queue instead of seeing an error.
6. Queue page loads immediately — no spinner, no freeze — showing: position ("You are 12,847 of 198,000"), estimated wait time ("~4 minutes"), and a live progress bar updating every 15 seconds.
7. A notification appears when user enters the top 1,000: "Almost there — get ready."
8. When it is their turn, the system automatically advances them to the seat selection screen — no refresh needed.
9. If the user closed the browser while waiting, they can reopen irctc.co.in within 10 minutes; their session is restored and position held.
10. An SMS/web push notification fires when they re-enter the top 1,000 after reconnecting.
11. User proceeds through seat selection, passenger review, and payment — identical to the current happy-path flow.
12. If all Tatkal seats are exhausted before the user reaches the front, they are shown the Failure Recovery screen (see AI-FEATURE.md) rather than a blank REGRET page.

### Technical Implementation Plan

**System components affected:**
- Booking entry endpoint (highest-risk component — queue sits in front of it)
- Session management service
- Notification service (SMS + web push)
- Front-end booking flow router

**New data requirements:**
- `booking_queue` table: `session_token (PK)`, `joined_at`, `position`, `status (waiting/admitted/expired)`, `reconnect_expires_at`
- Queue admission rate stored as a dynamic config value, updated every 30 seconds from booking service CPU/memory metrics

**API changes:**
- `POST /api/queue/join` — called when concurrent sessions exceed threshold; returns `{position, estimated_wait_seconds, queue_token}`
- `GET /api/queue/status` — polled every 15 seconds via Server-Sent Events (SSE); returns updated `{position, estimated_wait_seconds}`
- `POST /api/queue/reconnect` — accepts `session_token`; restores position if within 10-minute reconnect window
- `POST /api/queue/admit` — internal; called by admission service when user reaches front; advances them to booking flow

**Frontend changes:**
- New `QueuePage` component — shows position, progress bar, estimated wait; subscribes to SSE stream
- Queue router middleware — intercepts the "Continue" click at passenger details; checks if queue is active before proceeding
- Queue status polling replaced with SSE (WebSocket fallback for browsers that don't support SSE)

**Third-party services:**
- Redis (queue storage and atomic position decrement)
- Existing IRCTC SMS gateway (reconnect notification)
- Web Push API (browser notification when near front of queue)

### Success Metrics
- **Server error rate at 10:00–10:10 AM window:** reduce from current ~70% of sessions hitting an error to < 5%
- **Tatkal booking success rate:** users who attempt and complete a Tatkal booking increase from ~35% to ≥ 50% within 6 months
- **User-reported satisfaction (Tatkal flow):** App Store / Play Store rating for Tatkal flow from current 2.1/5 to ≥ 3.5/5
- **Queue abandonment rate:** ≤ 30% of users leave the queue before being admitted (benchmark: comparable rail/ticket queues)

### Edge Cases and Constraints
- **If the queue service itself crashes:** the system must fall back to the current direct-access model (no queue) rather than blocking all bookings. The queue is a progressive enhancement, not a dependency.
- **Bot and agent detection:** position in the queue is tied to the authenticated IRCTC user account, not IP address. One account = one queue position. Simultaneous sessions from the same account are collapsed into one.
- **Quota exhausted before user reaches front:** user is notified immediately and offered the Failure Recovery flow rather than being held in a now-pointless queue.
- **IRCTC-specific constraint:** any change to the booking entry endpoint requires Railways Ministry security sign-off and staging deployment on a production-mirrored environment. Sprint 3 timing accounts for this approval cycle.
- **Graceful degradation:** if Redis goes down, the queue service falls back to a simple rate-limiter (500ms delay between admissions) rather than crashing. The queue page shows "High demand — connecting you shortly" without position data.

---

## Feature Spec 2: Persistent URL-Encoded Filter State for Train Search

### Problem Statement
When users apply filters on the train search results page — class, availability, departure time — those filters are silently discarded the moment they navigate away: changing the date, tapping a train, or pressing the browser back button resets all selections to default, as documented in Part A — Problem 2. This affects every IRCTC user who searches trains between major stations, which is effectively the entire user base. The consequence is repeated manual re-entry of filters every single interaction, which is especially punishing on mobile where the filter panel itself requires multiple taps to open.

### Current State (from Part A)
As documented in Part A, Steps 5–6 and Steps 9–10 are the failure points. Filter state is stored in an in-memory JavaScript variable that is destroyed on any page navigation event. The results page re-renders from scratch via a fresh API call without reading back any prior filter state. Additionally, the "Available only" filter is processed entirely client-side on data that already includes waitlisted trains — meaning it visually hides rows but the underlying data is wrong, and filtered results still occasionally show WL entries. On mobile, navigating to a train's details page and pressing back returns the user to a fully unfiltered 30-train list.

### Proposed Solution
Every filter selection the user makes is immediately reflected in the page URL as a query parameter. The URL becomes the single source of truth for filter state. This means pressing back in the browser restores the exact filtered view automatically — the same way a Google search result is bookmarkable and shareable. Changing the date on the results page reloads availability for the new date while keeping all other filters intact. The "Available only" filter is enforced at the server level, not the client, so waitlisted trains are genuinely excluded from the response. On mobile, active filters are shown as dismissible chips below the search bar so the user always knows what's applied.

### Proposed User Flow — Step by Step
1. User searches NDLS → CSTM for June 10 — results page loads at URL: `irctc.co.in/train-list?from=NDLS&to=CSTM&date=20260610`
2. User opens the filter panel and selects: Class = SL, Availability = Available, Departure = Morning.
3. URL updates instantly to `...&class=SL&avail=confirmed&dept=morning` — no page reload.
4. Results list updates to show only SL-available, morning-departure trains. Count shown: "Showing 6 of 24 trains."
5. User taps a train to view fare details — navigates to the train detail page.
6. User taps the browser back button — returns to results page. URL still contains `&class=SL&avail=confirmed&dept=morning`. Filters are restored exactly. No re-selection needed.
7. User changes travel date to June 11 using the date picker on the results page.
8. Page reloads results for June 11 while preserving all filter parameters in the URL.
9. User wants to clear one filter — taps the "SL ×" chip below the search bar. URL updates, results refresh with only Availability and Departure filters still active.
10. User taps "Clear all filters" — URL reverts to base search params, full unfiltered results shown.

### Technical Implementation Plan

**System components affected:**
- Train search results front-end component
- Train search API (availability filter)
- Mobile filter panel component

**New data requirements:**
- No new database fields required.
- Filter parameter schema defined as URL query string spec: `class`, `avail`, `dept`, `days` — documented in internal API spec.

**API changes:**
- `GET /api/trains/search` — add server-side `availability` parameter. When `avail=confirmed`, the query excludes any train/class combination where current status is WL ≥ 1 or RAC. Previously this was client-side filtering on a full dataset.
- All existing parameters (`from`, `to`, `date`, `quota`) remain unchanged.

**Frontend changes:**
- Results component reads all filter state from URL query params on every mount — replacing the current in-memory store.
- Filter selections write to URL via `history.pushState()` — no page reload on filter change.
- New `FilterChips` component: horizontal strip below search bar showing active filters as dismissible pills. Each pill has a label ("SL") and an "×" to remove that filter.
- Mobile filter panel: replaced with a bottom sheet (`position: fixed; bottom: 0`) triggered on `touchstart` to eliminate the 300ms tap delay that causes the "double tap" issue.
- Results count label added: "Showing N of M trains."

**Third-party services:**
- None.

### Success Metrics
- **Filter persistence rate:** % of back-navigation events where filter state is restored — target 100% (from current 0% on mobile)
- **"Available only" filter accuracy:** % of results shown when filter is active that are genuinely CNF — target 100% (from current ~80%)
- **Mobile filter panel open rate on first tap:** target 100% (from current ~50% — currently requires two taps)
- **Average search-to-select time on mobile:** reduce by ≥ 25% as a proxy for reduced re-filtering friction

### Edge Cases and Constraints
- **URL length:** combining all filter params results in a URL of ~150–180 characters — well within browser limits.
- **Shareable URLs:** a user sharing the URL will share their filtered view. This is intentional and beneficial, but the receiving user may see different results if availability has changed. A timestamp of the search is appended to the URL: `&as_of=20260610T1005` and a banner shown: "Results as of 10:05 AM — refresh for current availability."
- **Browser back on first results page:** pressing back from results when there is no prior page navigates to the home search form — unchanged from current behaviour.
- **IRCTC-specific constraint:** the server-side availability filter adds a WHERE clause to the existing train search query. This must be load-tested to ensure it does not increase query time at peak hours — an index on `(train_id, class, quota, status)` is added as part of this spec.
- **Graceful degradation:** if URL param parsing fails (malformed URL from an external link), the page loads with no filters applied and shows a non-blocking toast: "Filters could not be restored — please re-apply."

---

## Feature Spec 3: Berth Lock with Visual Carry-Through

### Problem Statement
When users select a specific berth on the IRCTC coach seat map and click "Proceed," their selection is silently discarded by the time the passenger details form loads, as documented in Part A — Problem 3. This affects every mobile browser user (where the reset is 100% reproducible) and disproportionately harms senior citizens, persons with disabilities, and parents travelling with infants — all of whom have a medical or safety need for a specific berth type. The consequence is that users complete their booking believing they have a lower berth, only to receive an upper or middle berth on the final ticket.

### Current State (from Part A)
As documented in Part A, Steps 5–6 are the failure point. The berth selection is stored in a JavaScript session object in the browser. When the page transitions to the passenger details form, this object is supposed to be serialized and passed in the request payload. On mobile browsers, the page transition (slower render, earlier JS garbage collection) causes the object to be dropped before it is read. The backend receives the passenger form submission without a berth field, falls back to system auto-assignment, and the selection is permanently lost — the seat map is not revisited in the flow.

### Proposed Solution
Berth selection is confirmed at the moment the user taps a berth — not when they click "Proceed." The moment a berth is tapped, a lock is registered with the backend, holding that berth for 8 minutes under the user's session. Every subsequent screen in the booking flow (passenger details, review, payment) shows a non-editable "Your Selected Berth" card at the top so the user can see their selection is intact. The lock is released if payment is not completed within 8 minutes; the user is told this clearly and returned to the seat map, not to a generic error.

### Proposed User Flow — Step by Step
1. User has selected their train and quota; arrives at the coach seat map.
2. User scrolls the map to Coach S3 and taps Berth 23 — Lower.
3. Berth 23 highlights immediately in the UI. Simultaneously, the app sends a background lock request to the backend: `POST /api/booking/lock-berth` with `{berth_id: "S3-23", session_token}`.
4. Backend confirms the lock within < 500ms. Berth 23 is now reserved for this session for 8 minutes.
5. A "Selected Berth" chip appears below the map: "S3 · Berth 23 · Lower · Locked for 7:58."
6. User clicks "Proceed to Passenger Details."
7. Passenger details form loads. At the top: a green card showing "Your Berth: S3 — Berth 23 — Lower." This is non-editable and persists through the entire form.
8. User fills passenger names, ages, contact details, and clicks "Review Booking."
9. Review page shows the berth card again: "S3 — Berth 23 — Lower" alongside passenger summary.
10. User proceeds to payment, completes payment.
11. Booking confirmation screen and SMS ticket both show: "Coach S3 — Berth 23 — Lower" — matching exactly what was selected in Step 2.
12. If the lock expires (user took > 8 min): a modal appears — "Your berth lock expired. Berth 23 may have been taken. Please re-select." User is returned to the seat map with their previous coach position preserved — not to train search.
13. If the berth is taken by another user in the lock window (extremely rare — race condition): user sees "Berth 23 was just taken. Please select another." — returned to seat map.

### Technical Implementation Plan

**System components affected:**
- Seat map front-end component
- Passenger details form component
- Review and confirmation pages
- Booking service (backend)

**New data requirements:**
- New `berth_lock` table: `lock_id (PK)`, `berth_id`, `session_token`, `locked_at`, `expires_at` (8-minute TTL), `status (active/released/expired)`
- Berth lock token stored in `sessionStorage` on client — not in-memory JS variable (survives page transitions)

**API changes:**
- `POST /api/booking/lock-berth` — body: `{berth_id, session_token}`; response: `{lock_token, expires_at}` or `{error: "berth_unavailable"}`
- `DELETE /api/booking/lock-berth` — releases lock on payment failure or user back-navigation to seat map
- All subsequent booking API calls include `lock_token` as a header; booking service validates lock is still active before processing payment

**Frontend changes:**
- Seat map: berth tap triggers lock API call before updating visual state. Visual highlight only shown after lock confirmation (< 500ms — within acceptable UI feedback time).
- `BerthLockCard` component: persistent across passenger details, review, payment pages. Reads lock details from `sessionStorage` on every page mount — no dependency on page-transition JS state.
- Lock countdown timer shown in `BerthLockCard`: "Locked for 7:43" — updates every second.
- On lock expiry: modal overlay with single action: "Re-select Berth."

**Third-party services:**
- None. Lock is managed entirely within IRCTC's booking service.

### Success Metrics
- **Berth match rate:** % of completed bookings where final ticket berth matches selected berth — target ≥ 98% (from current estimated < 50% on mobile)
- **Mobile berth reset rate:** % of mobile sessions where berth selection is dropped between seat map and passenger details — target 0% (from current ~100%)
- **Lock expiry rate:** % of sessions where the 8-minute lock expires before payment — target < 10% (indicates form-filling is fast enough)
- **Support tickets about wrong berth assignment:** reduce by ≥ 60% within 3 months of launch

### Edge Cases and Constraints
- **User selects a berth but does not proceed:** lock is held for 8 minutes and released. This temporarily reduces available berths for other users. Mitigation: lock is only activated on explicit "Proceed" click, not on hover/tap. Users who tap multiple berths before deciding hold only the last-selected lock.
- **Two users attempt to lock the same berth simultaneously:** backend uses a row-level lock on the `berth_lock` table. First write wins; second receives `{error: "berth_unavailable"}` and sees "just taken" message.
- **User changes passenger count after berth lock:** if a passenger is added, additional berths must be selected. UI prompts: "You've added a passenger — please select a berth for them." Returns to seat map without releasing the existing lock.
- **IRCTC-specific constraint:** Tatkal bookings have an 8-minute total window from seat selection to payment. The 8-minute lock TTL is intentionally matched to this constraint. For General quota bookings (no time pressure), the lock TTL is extended to 15 minutes.
- **Graceful degradation:** if the lock API is unavailable, the feature falls back to the current behaviour (no lock, system auto-assignment). A notice is shown on the seat map: "Berth selection is unavailable right now — the system will assign you a berth automatically." No silent failure.

---

## Feature Spec 4: Smart CAPTCHA Replacement with Invisible Bot Detection

### Problem Statement
IRCTC's image CAPTCHA — required at login and again at booking confirmation — fails to render in approximately 1 in 3 page loads, and when it does render, failure clears the entire form including username and password fields, as documented in Part A — Problem 4. Additionally, Aadhaar OTP (now mandatory for Tatkal as of July 2025) consistently arrives after its 3-minute expiry window during the 10 AM peak, trapping users in a request-expire-retry loop. The consequence is that legitimate users — particularly those attempting time-critical Tatkal bookings — are locked out by the very security mechanism meant to protect them.

### Current State (from Part A)
As documented in Part A, Steps 2–5 (login CAPTCHA) and Steps 8–10 (booking confirmation CAPTCHA + Aadhaar OTP) are the failure points. The CAPTCHA image is served from IRCTC's own servers without CDN caching — under load the image endpoint returns errors, producing broken `<img>` tags. CAPTCHA state is server-side and session-bound: any form re-render generates a new challenge, invalidating whatever the user has typed. The OTP system relies on third-party SMS aggregators that are simultaneously overwhelmed during the Tatkal window when thousands of users request OTPs within seconds of each other.

### Proposed Solution
For users who have logged in successfully within the last 30 days, the image CAPTCHA at login is replaced with invisible bot detection — the system watches normal interaction signals (mouse movement, keystroke timing, scroll behaviour) to confirm the user is human. No action is required from the user. First-time or long-lapsed users see a simple checkbox ("I am not a robot") rather than distorted text. If a session is flagged as suspicious, a CAPTCHA appears — but failure no longer clears any fields other than the CAPTCHA itself. For Aadhaar OTP, the validity window is extended to 5 minutes, and after 60 seconds a visible "Resend OTP" button appears alongside a QR-based Aadhaar verification alternative.

### Proposed User Flow — Step by Step
1. Returning user (last login < 30 days) opens irctc.co.in and types username and password.
2. No visible CAPTCHA appears. Invisible bot detection runs in the background as the user types.
3. User clicks Login — invisible verification completes in < 300ms. Login succeeds.
4. First-time or long-lapsed user: sees a single checkbox "I am not a robot." Checks it. Login proceeds.
5. Suspicious session (bot-like behaviour detected): a standard CAPTCHA challenge appears. User types the characters. If wrong: only the CAPTCHA field clears — username and password remain filled.
6. After 3 failed CAPTCHA attempts: account is temporarily locked for 2 minutes with a clear countdown timer shown ("You can try again in 1:47"). Not a silent session kill.
7. User proceeds to book a Tatkal ticket. At the booking confirmation screen, no second CAPTCHA appears — the session-level invisible verification from login is sufficient.
8. For Tatkal, Aadhaar OTP is requested. A 5-minute timer is shown: "OTP valid for 4:53."
9. After 60 seconds without OTP entry, a "Resend OTP" button becomes active.
10. An "Aadhaar QR alternative" link also appears — user can scan their Aadhaar QR code using UIDAI's offline XML verification instead of waiting for SMS.
11. User enters OTP (or completes QR scan) and proceeds to payment.

### Technical Implementation Plan

**System components affected:**
- Login form (front-end + authentication service)
- Booking confirmation screen (front-end)
- OTP issuance service
- Session verification middleware

**New data requirements:**
- `user_verification_history` table: `user_id`, `last_successful_login`, `risk_score_at_last_login` — used to determine whether invisible or visible CAPTCHA is shown
- No new fields required for OTP changes (TTL is a config value in the OTP service)

**API changes:**
- `POST /api/auth/login` — accepts an invisible CAPTCHA token (from hCaptcha/reCAPTCHA v3) in addition to credentials. Backend validates token server-to-server before issuing session.
- `POST /api/auth/request-otp` — response now includes `expires_in: 300` (5 minutes, up from 180). No other changes.
- `POST /api/auth/verify-aadhaar-qr` — new endpoint accepting UIDAI offline XML payload as alternative to OTP.

**Frontend changes:**
- Login form: remove image CAPTCHA `<img>` and refresh button. Add invisible CAPTCHA SDK script (hCaptcha Invisible or reCAPTCHA v3). Script loads asynchronously — does not block form render.
- CAPTCHA failure handling: only the CAPTCHA input field is cleared on failure (not the whole form). Username and password fields are `autocomplete="on"` and not cleared programmatically.
- Booking confirmation: CAPTCHA removed. Session token from login is passed as a header for verification.
- OTP screen: add countdown timer component, "Resend OTP" button (active after 60s), and "Use Aadhaar QR instead" link.

**Third-party services:**
- hCaptcha Invisible or Google reCAPTCHA v3 (invisible bot detection)
- UIDAI Offline e-KYC API (Aadhaar QR XML verification — already government-operated, no new vendor)

### Success Metrics
- **Login success rate on first attempt:** target ≥ 90% (from current estimated ~65% due to CAPTCHA failures)
- **CAPTCHA-related booking abandonment rate:** reduce by ≥ 50% within 60 days of launch
- **Aadhaar OTP completion rate within the validity window:** target ≥ 85% (from current estimated ~55% during peak hours)
- **Bot/fraudulent booking rate:** must not increase — monitored weekly for 3 months post-launch

### Edge Cases and Constraints
- **Invisible CAPTCHA gives a low confidence score for a legitimate user (false positive):** system falls back to checkbox CAPTCHA, never to distorted image CAPTCHA. User is not blocked.
- **UIDAI Aadhaar QR API is down:** OTP remains the primary method. QR option is hidden from the UI if the UIDAI endpoint health check fails.
- **Users on feature phones or very old browsers:** invisible CAPTCHA SDK may not load. Fallback: show checkbox CAPTCHA. Never show broken image CAPTCHA.
- **IRCTC-specific constraint:** IRCTC operates under IT Act and Railways Ministry guidelines for authentication. Any CAPTCHA replacement requires approval from Railway Board's IT security committee. hCaptcha and reCAPTCHA are both used by other Indian government portals (DigiYatra, GSTN) — precedent exists.
- **Graceful degradation:** if the invisible CAPTCHA vendor's SDK is unreachable (CDN outage), the feature falls back to the existing image CAPTCHA — no login breakage. The SDK is loaded with a timeout; fallback triggers after 2 seconds.

---

## Feature Spec 5: Contextual PNR Status Dashboard with Waitlist Intelligence

### Problem Statement
Checking PNR status on IRCTC requires login despite PNR data being publicly accessible, and the resulting status page shows only a raw waitlist number with no context about what it means or what the user should do next, as documented in Part A — Problem 5. There are no contextual action links — no Cancel, no TDR — from the PNR page, forcing users to navigate a 4-step path through My Transactions to access these options. Approximately 15–20% of all IRCTC tickets are initially waitlisted; every one of these users faces this experience every time they check their ticket status.

### Current State (from Part A)
As documented in Part A, Steps 3–4 (login gate for public data) and Steps 7–10 (missing waitlist intelligence and contextual actions) are the failure points. The PNR status page shows a table with current waitlist position but no trend, no probability, and no comparison to historical data for the same train/class/season. The TDR filing option is buried four navigation levels deep with no link from the PNR page. Users who need to make contingency travel decisions — book an alternative, cancel for a refund, or wait — have none of the information they need to decide.

### Proposed Solution
PNR status lookup is available without login — the user types their 10-digit PNR on the homepage or on a shareable URL and immediately sees their current status. For waitlisted tickets, the page shows: the current WL position, a trend indicator (improving or worsening, with a 7-day sparkline), a confirmation probability percentage derived from historical data for that specific train and class, and an estimated confirmation timeline. Below the status, contextual action buttons appear based on eligibility: Cancel Ticket, File TDR, or Set SMS Alert for status changes. Logged-in users see all their tickets in one dashboard view. No hunting through menus required.

### Proposed User Flow — Step by Step
1. User receives a WL ticket SMS and wants to check current status.
2. User opens irctc.co.in — a PNR search bar is visible on the homepage without logging in.
3. User types their 10-digit PNR and clicks "Check Status."
4. Status page loads immediately — no login redirect.
5. Status card shows: PNR, Train name, Date, Class, Journey route.
6. Below: current status — "WL 8 (was WL 12 at booking)" with a green "▼ Improving" badge.
7. A 7-day sparkline chart shows WL movement: 12 → 11 → 10 → 9 → 8.
8. Confirmation probability shown: "74% likely to confirm — based on historical data for 12952 Mumbai Rajdhani, SL class, June journeys."
9. Timeline estimate: "Tickets at this position typically confirm by D-3. Your journey is in 4 days."
10. Action buttons appear below, with eligibility logic applied:
    - "Cancel & Get Refund" — shown with refund amount calculated: "You'll receive ₹ 840 back."
    - "File TDR" — shown only if journey date has passed and conditions are met, with one-line eligibility reason.
    - "Set SMS Alert" — toggle; sends SMS when WL position changes.
11. User taps "Set SMS Alert" — a confirmation toast appears: "You'll be notified when your WL position changes." No login required for this action.
12. For Cancel and TDR actions: user is prompted to log in at that point — not before. Login is scoped to the action, not to the entire page.

### Technical Implementation Plan

**System components affected:**
- Homepage search component
- PNR status page (front-end + back-end)
- Notification service (SMS)
- TDR eligibility rules engine (new)

**New data requirements:**
- `wl_confirmation_lookup` table: `train_id`, `class`, `month`, `wl_position_at_booking`, `historical_confirmation_rate`, `avg_days_to_confirm` — pre-computed weekly from historical booking outcome data
- `pnr_alerts` table: `pnr`, `phone_number`, `last_wl_position`, `created_at` — for SMS alert subscriptions

**API changes:**
- `GET /api/pnr/:pnr` — no auth required. Proxies to NTES PNR API; augments response with WL probability from `wl_confirmation_lookup` table; returns TDR eligibility flag.
- `POST /api/pnr/alert` — no auth required for set; body: `{pnr, phone_number}`. Creates row in `pnr_alerts`.
- `GET /api/pnr/history/:pnr` — returns 7-day WL movement array for sparkline. Populated by a daily cron job that records WL positions for all active waitlisted tickets.
- Cancellation and TDR endpoints: unchanged. Accessed after login prompt from the PNR page.

**Frontend changes:**
- Homepage: PNR search bar added alongside the existing "Plan My Journey" form.
- PNR status page: redesigned with status card, sparkline chart, confirmation probability meter, action buttons.
- `WLSparkline` component: renders 7-day WL movement using SVG — lightweight, no charting library required.
- `ConfirmationMeter` component: percentage bar with label and confidence range ("68–80%").
- Action buttons: rendered conditionally based on eligibility flags returned by API. Greyed with tooltip if ineligible ("TDR not applicable — journey date has not passed").
- Login prompt: triggered only on Cancel/TDR tap — modal overlay, not full page redirect.

**Third-party services:**
- NTES (National Train Enquiry System) PNR API — already public, used by many third-party apps
- Existing IRCTC SMS gateway for alerts

### Success Metrics
- **PNR status checks without login:** target ≥ 70% of all PNR checks using the no-login path within 3 months
- **Contextual action conversion rate:** % of WL ticket holders who take an action (cancel, alert, TDR) from the PNR page — target ≥ 20% (from current ~0% due to absent action links)
- **WL confirmation probability accuracy:** predicted vs. actual confirmation rate within ±10 percentage points — monitored monthly
- **"Where do I cancel my ticket" support queries:** reduce by ≥ 40% within 3 months (proxy for navigability improvement)

### Edge Cases and Constraints
- **PNR entered for a completed journey:** status page shows final outcome ("Confirmed — journey completed") and only shows TDR option if applicable.
- **PNR not found or invalid:** clear error shown — "PNR not found. Please check the 10-digit number on your ticket." — not a generic 404.
- **Confirmation probability for rare train/class/season combinations:** lookup table may have insufficient historical data. In this case, probability is not shown — replaced with "Insufficient data for this route." Never show a fabricated probability.
- **SMS alert for a PNR that confirms or cancels before the next check:** alert fires once ("Your ticket WL 8 has confirmed — Coach S3, Berth 14") and the alert subscription is auto-deleted.
- **IRCTC-specific constraint:** NTES API has rate limits (~100 req/min for unauthenticated access). IRCTC's institutional access tier has higher limits. Caching: PNR status is cached for 5 minutes; sparkline data is cached for 1 hour.
- **Graceful degradation:** if the NTES API is down, the PNR page shows the last cached status with a timestamp: "Status as of 2 hours ago — NTES is currently unavailable." Never shows an empty or broken page.

---

## Feature Spec 6: Responsive Mobile-First Redesign of Core Booking Flow

### Problem Statement
Accessing irctc.co.in on a mobile browser produces a functionally broken experience across every critical booking screen — the search results table overflows the viewport, the filter panel requires two taps to open, and the seat map renders at desktop scale with touch targets too small to tap accurately — as documented in Part A — Problem 6. This affects an estimated 60–70% of IRCTC's total web traffic, which is mobile-browser-based. The consequence is that the majority of users are trying to complete one of the most time-critical actions in Indian consumer internet — Tatkal booking — on an interface that is physically broken for their device.

### Current State (from Part A)
As documented in Part A, Steps 3–5 (results table overflow and filter panel double-tap) and Steps 7–10 (seat map scale and touch target failure) are the failure points. The results table uses a fixed-width CSS layout with no responsive container — class availability columns are off-screen on any viewport under 768px with no horizontal scroll indicator. The filter panel trigger has no minimum tap target size and uses a `click` event listener that fires after the 300ms mobile tap delay, causing the first tap to appear to do nothing. The seat map SVG renders at fixed desktop coordinates with individual berths approximately 18–22px wide — far below the 44×48px minimum for reliable touch interaction.

### Proposed Solution
The core booking flow is redesigned as a mobile-first experience. Search results are displayed as vertical cards — one train per card — where all key information is visible without scrolling or zooming. Active filters appear as dismissible chips at the top. The filter panel opens on the first tap via a bottom sheet. The seat map is replaced with a touch-friendly berth picker where each berth is a large, clearly labelled button. All changes use progressive enhancement — the desktop layout is completely unchanged. A mobile user and a desktop user interact with the same underlying system but see interfaces built for their context.

### Proposed User Flow — Step by Step
1. User opens irctc.co.in on mobile Chrome and fills in the search form — identical to current.
2. User taps "Find Trains." Results load as vertical cards. Each card shows: Train name + number, Departure time + station, Arrival time + station, Duration, and class availability pills (SL ✓ | 3A ✓ | 2A WL | 1A —) in a single horizontal row within the card.
3. All key information is visible within the card without horizontal scrolling.
4. User taps the "Filters" button at the top right — a bottom sheet slides up from the bottom of the screen on the first tap.
5. User selects "SL" and "Available Only" within the bottom sheet. Taps "Apply." Sheet closes; results update; filter chips appear: "SL × | Available ×."
6. User taps a train card — navigates to fare details.
7. User taps back — returns to filtered results with chips intact (URL-based persistence, Spec 2).
8. User taps "Book Now" and proceeds through passenger details — unchanged from current desktop flow.
9. User reaches seat selection. Instead of the full coach SVG map, a "Berth Picker" view is shown on mobile:
   - Coach selector at top (S1 / S2 / S3 — horizontally scrollable chips)
   - Below: berths listed as a 3-column grid of large buttons (48×48dp minimum) labelled by number and type (L / M / U / SL / SU)
   - Available berths shown in white; booked in grey; selected in green
10. User taps Berth 23 — Lower. Button highlights green. Lock is registered (Spec 3). A "Selected: S3 · Berth 23 · Lower" bar appears at the bottom.
11. User taps "Confirm Selection" — proceeds to passenger details with berth card visible at top.
12. A persistent bottom bar shows throughout the booking flow: current step (Step 2 of 4 · Passenger Details), selected train summary, and a "Continue" CTA — always visible without scrolling.

### Technical Implementation Plan

**System components affected:**
- Train search results component
- Filter panel component
- Seat map / berth selection component
- Booking flow layout wrapper

**New data requirements:**
- None. All changes are presentational — no new data is required.

**API changes:**
- None. The mobile card layout consumes the same train search API response as the desktop table.

**Frontend changes:**
- CSS breakpoint at `max-width: 767px`. Below this breakpoint:
  - Results table replaced by card grid (`display: flex; flex-direction: column; gap: 12px`)
  - Each card component: `TrainResultCard` — receives same train object as the table row, renders as a card
  - Class pills inside card: `display: flex; overflow-x: auto; gap: 8px` — horizontal scroll within the card only, not the whole page
- Filter panel: replaced with `BottomSheet` component at mobile breakpoint. Trigger button uses `touchstart` event listener (not `click`) to eliminate 300ms delay. Bottom sheet uses `position: fixed; bottom: 0; width: 100%` with slide-up animation.
- `FilterChips` component: horizontal strip below search bar. Each chip: `min-height: 44px; padding: 0 12px; border-radius: 20px`. Tap to remove.
- Seat map at mobile: existing SVG hidden via `display: none` at mobile breakpoint. New `BerthPicker` component shown: CSS grid of `<button>` elements, each `min-width: 48px; min-height: 48px; font-size: 13px`.
- `BookingProgressBar`: `position: fixed; bottom: 0` — shows step count, train summary, Continue CTA. Hides on scroll-up (IntersectionObserver) to not obstruct content.
- All desktop styles untouched. All new mobile styles in a separate `mobile.css` file imported conditionally.

**Third-party services:**
- None.

### Success Metrics
- **Mobile booking completion rate:** % of mobile sessions that reach payment confirmation — target increase of ≥ 30% from baseline within 90 days
- **Mobile results page horizontal scroll events:** target 0 (from current ~100% of mobile sessions)
- **Filter panel first-tap open rate on mobile:** target 100% (from current ~50%)
- **Berth selection accuracy on mobile:** % of taps that select the intended berth — target ≥ 98% (from current ~60–70% due to small touch targets)
- **Mobile Lighthouse performance score:** target ≥ 70 on mobile (First Contentful Paint < 2.5s on 4G)

### Edge Cases and Constraints
- **Trains with many classes (6+ availability columns):** class pills inside the card use horizontal scroll within the card — overflow is contained and expected. A "scroll hint" indicator (gradient fade on right edge) is shown on first load.
- **User switches from mobile to desktop mid-session:** the URL-encoded filter state (Spec 2) means switching devices restores the same search context. Layout adapts automatically via CSS breakpoint.
- **Low-bandwidth connections:** card layout renders with a skeleton loader per card rather than waiting for all results — progressive loading. Each card appears as data arrives.
- **Right-to-left language support:** IRCTC currently serves English and Hindi. The card and bottom sheet layout must support RTL text rendering if Hindi UI mode is enabled — flex direction is reversed via `[dir="rtl"]` selector.
- **IRCTC-specific constraint:** IRCTC's front-end codebase is a legacy Angular application. `BottomSheet` and `BerthPicker` must be implemented as Angular components, not React or Vue. CSS changes must not conflict with the existing Bootstrap grid used on desktop.
- **Graceful degradation:** if JavaScript fails to load (low-memory device, script error), the existing desktop table is shown with a banner: "For the best experience on mobile, try the IRCTC Rail Connect app." The mobile-first redesign is a JS-dependent enhancement — the baseline HTML table remains in the DOM and is the fallback.