# 4. Key Features

> **Summary.** This section describes the platform's main capabilities and the rule that governs each one. A theme runs through all of them: **the server decides**. Prices, availability, deadlines, till access and money totals are computed in one place on the server. The client only displays them.

[← Back to README](../README.md)

---

## 4.1 Booking and Server-Side Pricing

<img src="../assets/screenshots/parent-new-booking.png" width="240" align="right" alt="Booking flow">

A parent books through a four-step flow: **child → time → package → coach**. The system:

1. Builds a six-day date strip from **Tehran's today** and hides slots that have already started. The list refreshes every 30 seconds, so a slot that lapses mid-flow drops out.
2. Offers only slots that fit inside the centre's **opening windows**, which the owner edits.
3. Offers only coaches who are **rostered for the entire session**, not suspended, and not managers.
4. Calls `GET /bookings/quote` for the price. `POST /bookings` recomputes the price with the **same function**, so the quoted figure and the charged figure always agree.

**Discount engine.** `quoteFor(family, duration, slot)` combines:

| Component | Rule |
|---|---|
| Package price | From the owner's price list (`Settings.packages`) |
| Loyalty tier | Derived from the family's lifetime hours; thresholds follow the configured tiers |
| First-booking discount | Optional, off by default. "First" means zero non-cancelled bookings ever. |
| Quiet-hour discounts | Matched on session **start**, optionally per weekday |

Discounts **stack** (the owner's decision) and are clamped at 100%, so a price can't go negative. The quote returns named reasons, and the payment screen lists each discount applied.

**Last call.** The final opening window of the day may be entered up to *N* minutes before closing (default 15), so the last hour can be sold. The roster check is capped at closing time so that such a slot is not offered and then refused.

<br clear="right">

## 4.2 Check-in, Live Sessions and Checkout

<img src="../assets/screenshots/parent-qr.png" width="240" align="right" alt="QR check-in">

- A paid booking carries a **QR token**. The coach or desk scans it to check the child in, which starts the session timer.
- Coaches see **today's class**: booked children and walk-ins merged in one list, each with a live timer and a "guest" chip for walk-ins.
- An **upcoming** list shows today's reservations with time, payment state and (for managers) coach. Checking in from the row starts the timer.
- **Extensions** add 30 minutes at the configured extension price. The client may request the length, but the server sets the price, in whole blocks.
- **Checkout settles.** `settleBooking()` computes what is still owed from the payments already recorded for the booking, not from a status flag. As a result an extension is always charged, and a prepaid booking is never charged twice.

<br clear="right">

## 4.3 Walk-in Desk Check-in

<img src="../assets/screenshots/supervisor-active.png" width="240" align="right" alt="Children present">

The desk flow mirrors parent booking but asks a different question: **who is in the building right now?** It doesn't ask who is bookable for a future slot.

- Coaches are taken from `GET /shifts/on-duty`. Slots come from that day's **shifts**, and packages are checked against the **shift's end**.
- A new visitor becomes a `Guest` (cash or card). A registered family becomes a real `Booking`, which earns loyalty hours and may use wallet credit.
- Opening-hours and past-slot rules apply to **self-service only**. The roster rule applies to everyone, so the desk can't assign a coach who is not present.

<br clear="right">

## 4.4 Cash Desk and the Till

<img src="../assets/screenshots/supervisor-cashier.png" width="240" align="right" alt="Cash desk">

- **Access comes from the rota.** A supervisor marks a shift as the till shift. That coach gains cash-desk access for exactly that window on that weekday, and it ends by itself. `cashdesk.js` is the single authority, evaluated on the Tehran clock.
- **Least privilege for cashiers.** A cashier who is not a manager sees the amount of the sale in progress but no takings, totals or ledger history. This is enforced in the API response, not only hidden in the UI.
- **Wallet top-ups** are restricted to supervisors and the owner. Cashiers take payment for sessions; they do not credit accounts.
- **The till closes itself.** See §4.6.

<br clear="right">

## 4.5 Rota, Tasks and Coach Management

| Shift rota | Coach management |
|:---:|:---:|
| <img src="../assets/screenshots/coach-shifts.png" width="220"> | <img src="../assets/screenshots/owner-coaches.png" width="220"> |


- Weekly rota with Jalali dates, shift-swap requests and approval.
- **Tasks are bound to shifts.** An assignee must be on duty at the due time, and the deadline is resolved from the rota by the server. A coach may update only a task's status. Overdue tasks are flagged.
- **Coach suspension** (owner only) is refused while a child is in the coach's care. It removes future shifts but deliberately **does not** cancel open bookings, which are promises to families. The count is returned so the owner can reassign them.
- **Per-session ratings** feed a per-coach performance view: lifetime and recent averages, a six-month trend and the star distribution.

## 4.6 Automated End of Day

An in-process scheduler (`nightly.js`) polls every 60 seconds. Polling was chosen over a single midnight timeout because it corrects itself across restarts and container sleeps, and the hosting platform offers no cron.

| Job | Behaviour |
|---|---|
| `rollRegister()` | Seals a stale register at the end of **its own** day and opens a new one carrying the float, timestamped at the start of the Tehran day. |
| `closeForNight()` | Closes the till at the configured close time within a 10-minute firing window, so a manager's deliberate late reopen is respected. |
| `closeAttendance()` | Closes visits left open from previous days. Walk-ins close at the end of the window they bought and are marked `autoClosed`. A booking that still **owes money is left open**, because the sweep never takes payment. |

While the till is closed, the four money-taking paths refuse new payments, except a checkout that owes nothing, so a late checkout is never stranded.

## 4.7 Owner Analytics and Day Archive

<img src="../assets/screenshots/owner-analytics.png" width="240" align="right" alt="Analytics">

- **One money model.** `moneyFor(from, to)` defines `collected` (money actually received, with top-ups shown as their own line) and the derived figures. The dashboard, analytics, day archive and till all read it. Wallet money is **cash-basis**: counted once, when it arrives.
- **Walk-ins count everywhere.** A shared `revenueSessions()` series merges bookings and guests, so walk-ins appear in revenue by coach, coach utilisation, top customers and busy-hour charts.
- **Busy-hour and busy-weekday charts** are built from when visits actually **ran**. The divisor is real trading days, so closed days show as "closed", not as zero.
- **Every figure states its basis and period**: this Jalali month, the last 30 days, or lifetime.
- **Day archive:** browse any past Tehran day for its sessions held (by scheduled time, not booking time), walk-ins with real attendance times, and a ledger whose rows add up to the headline figure.
- **Satisfaction** is aggregated directly from individual ratings. It is not an average of per-coach averages, and it reads "no score" when there are no ratings.

<br clear="right">

## 4.8 Accounting and Payroll

<img src="../assets/screenshots/owner-accounting.png" width="240" align="right" alt="Accounting">

An owner-only accounting workspace with three panels: **Summary**, **Expenses** and **Payroll**.

- **Expenses** by category, with an edit history.
- **Staff pay profiles** (monthly or hourly), attached to the coach record. The coach may not have a login.
- **Monthly payroll runs** on the Jalali calendar: draft, then approve. Approval writes the salary expenses. A draft counts for nothing, and approving twice returns `409`.
- **Profit and loss** per Jalali month. It states its basis on screen (income is money received), and shows **unused prepaid credit** beside net profit, not inside it, because that is service already paid for but not yet delivered.
- Hourly pay is labelled an **estimate**, because the rota is a weekly template and can't represent one-off absences.

<br clear="right">

## 4.9 Centre Settings

<img src="../assets/screenshots/owner-settings.png" width="240" align="right" alt="Settings">

Everything the business may want to change is editable by the owner without a deployment: package durations and prices, loyalty tiers, programmes, opening windows, last-call minutes, automatic till-close time, space capacity, first-booking and quiet-hour discounts, wallet top-up bonus, landing-page content, and classes and events with images.

Price changes reach the server-side pricing engine within its 30-second cache window, and the cache is invalidated explicitly on save.

<br clear="right">

## 4.10 Parent Experience

| Bookings | Daily reports | Wallet |
|:---:|:---:|:---:|
| <img src="../assets/screenshots/parent-bookings.png" width="220"> | <img src="../assets/screenshots/parent-reports.png" width="220"> | <img src="../assets/screenshots/parent-wallet.png" width="220"> |

- Booking history with status filters, payment and the check-in code.
- Daily session reports written by the coach.
- A post-session review prompt for the most recent unrated session. It reappears on the next app open if postponed, and never interrupts a flow in progress.
- Wallet balance and statement, with top-up bonuses configured by the owner.
- Messaging with the centre.

---

[← Domain Model](03-domain-model.md) · [Next: Security →](05-security.md)
