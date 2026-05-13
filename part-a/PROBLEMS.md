# IRCTC Problem Discovery — Part A

## Summary
- Total problems documented: 6 (3 given + 3 self-discovered)
- Platform explored: irctc.co.in (live, as of May 13, 2026)
- Devices used: Desktop Chrome (Windows)

---

## Problem 1: Tatkal Booking Crashes at 10:00 AM [Given]

**What is broken:**
The server routinely becomes unresponsive or throws gateway timeouts exactly at 10:00 AM (AC classes) and 11:00 AM (Non-AC classes) when Tatkal quota opens. There is no queue system, no graceful degradation, and no user feedback regarding their place in line or the server status.

**Affected users:**
Millions of daily commuters and emergency travelers trying to secure last-minute reservations.

**Frequency:**
Daily. The crash pattern is consistently observed every day at exactly 10:00 AM and 11:00 AM during peak load.

**Current flow — step by step:**
1. User logs into IRCTC at 9:55 AM.
2. User enters source and destination stations and selects tomorrow's date.
3. User selects "Tatkal" from the quota dropdown.
4. User clicks "Search" and waits on the train list page.
5. At exactly 10:00:00 AM, user clicks "Check Availability & Fare" for 3A class on their preferred train.
6. The page shows a loading spinner for 30-60 seconds.
7. The server drops the connection, logging the user out or displaying a "Service Unavailable (503)" / "Gateway Timeout (504)" error.
8. User is forced to restart from Step 1, by which time Tatkal quota is exhausted.

**Where exactly it breaks:**
Step 6 to 7: The system fails during the availability fetch under extreme concurrent load, resulting in connection timeouts instead of queuing the user's request.

---

## Problem 2: Search Filters Do Not Work Reliably [Given]

**What is broken:**
Applied search filters (e.g., Sleeper class, specific departure times, "Available tickets only") do not reliably persist during navigation or fail to correctly filter the dataset on the client side. Clicking "Back" or modifying a search often clears all previously selected filters without warning.

**Affected users:**
Users looking for specific travel preferences, particularly those comparing multiple trains or dates before booking.

**Frequency:**
High. Occurs on almost every search iteration where users refine results or navigate back from the passenger details page.

**Current flow — step by step:**
1. User searches for trains between NDLS (New Delhi) and BCT (Mumbai Central).
2. The results page loads with 20+ trains.
3. User applies the "Journey Class: Sleeper (SL)" filter on the left sidebar.
4. The page updates to show only trains with Sleeper class.
5. User clicks "Check Availability" on a train and proceeds to "Book Now".
6. User decides to check another train and clicks the browser "Back" button or the on-page "Back" button.
7. User is returned to the search results page, but the "Sleeper (SL)" filter is cleared, showing all classes again.

**Where exactly it breaks:**
Step 7: The client application fails to persist filter state in the URL parameters, local storage, or session state when navigating between views.

---

## Problem 3: Seat Selection Resets [Given]

**What is broken:**
The user's preferred berth selection (e.g., Lower Berth) made during the booking flow is not guaranteed and often resets or is ignored without notifying the user before final payment.

**Affected users:**
Elderly passengers, pregnant women, and users with mobility issues who specifically require a Lower Berth.

**Frequency:**
Moderate to High, especially on mobile devices where session sync issues or page reloads occur more frequently.

**Current flow — step by step:**
1. User selects a train and class, then clicks "Book Now".
2. User is redirected to the Passenger Details page.
3. User enters passenger name, age, gender.
4. User selects "Lower" from the Berth Preference dropdown.
5. User fills in contact details and captcha, then clicks "Continue".
6. User is taken to the Review Journey page.
7. The berth preference is missing or reset to "No Preference" in the summary, and the user only realizes this after payment when the final ticket is generated.

**Where exactly it breaks:**
Step 6 to 7: The selected berth preference fails to accurately pass from the form submission to the review page's state, leading to silent data loss before the payment gateway handoff.

---

## Problem 4: Unnecessary Captcha for Read-Only PNR Status [Self-Discovered]

**How I found it:**
Navigated to the "PNR STATUS" section from the home page menu to check the status of a ticket.

**Screenshot or description:**
See `assets/screenshots/irctc_pnr.png`. The page requires entering the PNR number, clicking submit, and then forces the user to solve a complex arithmetic captcha just to view the read-only status of a ticket.

**What is broken:**
High-friction access to public/read-only information. The PNR status check does not expose sensitive PII (only masking names), yet it requires solving a math captcha, which is an unnecessary hurdle for an informational query.

**Affected users:**
All users, especially those checking waitlisted tickets frequently (which can be multiple times a day). It disproportionately affects users with low numerical literacy or visual impairments.

**Frequency:**
Every single time a user checks a PNR status.

**Current flow — step by step:**
1. User clicks on "PNR STATUS" from the main navigation.
2. A new tab opens with the Enquiry portal.
3. User enters their 10-digit PNR number in the input field.
4. User clicks "Submit".
5. A modal appears asking the user to solve an arithmetic captcha (e.g., "What is 45 + 12?").
6. User calculates the sum, types it in, and clicks "Submit" again.
7. The PNR status is finally displayed.

**Where exactly it breaks:**
Step 5: The introduction of a friction-heavy arithmetic captcha for a non-mutating, low-risk GET request.

---

## Problem 5: Confusing "Vikalp" Opt-in Checkbox [Self-Discovered]

**How I found it:**
Proceeding through the passenger details form during the booking flow for a waitlisted ticket.

**Screenshot or description:**
The Passenger Details page contains a checkbox labeled "Consider for Auto Upgradation" and another section for "Vikalp" scheme which is poorly explained, leading users to believe it increases their chances on the *current* train rather than shifting them to a completely different train.

**What is broken:**
The UI/UX for opting into the Vikalp (Alternate Train Accommodation) scheme lacks clear explanations and consequences. Users check it hoping to get a confirmed ticket on their selected train, but instead get re-routed to a different train at a different time, with no easy way to undo it without cancelling the whole ticket.

**Affected users:**
Users booking waitlisted tickets who are unfamiliar with the specific administrative rules of the Vikalp scheme.

**Frequency:**
High for waitlist bookings.

**Current flow — step by step:**
1. User selects a train with "Waitlist (WL)" status and clicks "Book Now".
2. User fills out passenger details.
3. User sees an option to opt into the "Vikalp" scheme with minimal tooltip explanation.
4. User checks the box, assuming it acts like "Auto Upgradation" to improve chances on this specific train.
5. User completes payment and receives a waitlisted ticket.
6. The ticket chart is prepared, and the user is automatically shifted to an alternate train departing 12 hours later.
7. User realizes the mistake but cannot "opt-out" of Vikalp post-charting without cancelling the ticket and incurring fees.

**Where exactly it breaks:**
Step 3 to 4: The microcopy and information architecture completely fail to communicate the destructive/altering nature of the Vikalp scheme at the point of decision.

---

## Problem 6: Unforgiving Session Timeout Deletes All Entered Data [Self-Discovered]

**How I found it:**
While adding passenger details, I took a few minutes to find the Aadhaar numbers/details for 4 passengers. When I clicked continue, I was thrown out.

**Screenshot or description:**
After taking ~3 minutes on the passenger details page, clicking "Continue" results in a sudden redirect to the login page with an obscure "Session Expired" alert.

**What is broken:**
The session timeout on the passenger details page is extremely short, has no visible countdown or warning, and does not auto-save form state. If the session expires, the user loses all entered passenger data and must start over from the very beginning (train search).

**Affected users:**
Families booking tickets for multiple passengers, users with slower typing speeds, and users who need to look up identification details during booking.

**Frequency:**
Moderate to High for group bookings (4-6 passengers).

**Current flow — step by step:**
1. User selects a train and proceeds to the passenger details form.
2. User begins filling out details for Passenger 1.
3. User switches to another app or physical documents to find age/ID details for Passengers 2, 3, and 4.
4. User spends approximately 3-4 minutes entering all the details.
5. User clicks "Continue" to proceed to the review page.
6. The system abruptly shows a "Session Expired" or "Timeout" popup.
7. User is redirected to the IRCTC home page. All passenger details and selected train state are lost.

**Where exactly it breaks:**
Step 5 to 6: The application fails to provide a proactive timeout warning and completely fails to persist form data in local storage as a fallback.
