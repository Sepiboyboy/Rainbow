# 6. Engineering Case Studies

> **Summary.** These case studies were selected from the production history of the platform. Each one follows the same structure: **symptom → root cause → fix → result**. Several share a lesson: when two parts of a system answer the same question independently, they eventually disagree. Many of the fixes therefore consolidate a rule into a single authoritative function.

[← Back to README](../README.md)

---

## 6.1 Persian digits made a rostered coach unbookable

**Symptom.** A coach was visibly on the rota for the afternoon, but parents could never select her.

**Root cause.** Shift times entered on a Persian keyboard were stored as `"۱۶:۰۰"`. The time parser used `\d`, which in JavaScript matches **ASCII digits only, even with the Unicode flag**:

```
toMinutes("16:00") → 960        toMinutes("۱۶:۰۰") → null
```

The rota screen rendered the string correctly, since rendering requires no parsing. The availability check parsed it to `null`, so the coach was shown as rostered and treated as unbookable at the same time. The same parser sat behind 16 server call sites, including opening hours, automatic till close and quiet-hour discounts.

**Fix.** A shared `toASCIIDigits()` now normalises inside the parser, which repaired all existing data at once. Shift writes validate and store canonical `HH:MM`, and the client's separate normaliser was replaced with the shared one so the two can't drift. An idempotent migration rewrote the stored values.

**Result.** The coach became bookable and the whole class of silent failures was closed. A related rule followed: any value the server compares verbatim, such as a generated password, is **displayed in ASCII digits**.

---

## 6.2 One money model instead of five

**Symptom.** The dashboard, analytics, day archive and till disagreed about the same day's income. In one case a day's headline showed about 54M toman while its own subtitle showed about 126M.

**Root cause.** Each screen had its own revenue calculation. Some summed sessions **delivered** (accrual), others summed money **received** (cash). Wallet top-ups and the later spending of that credit were both counted, so the same toman appeared twice on two different days. One field name, `revenueToday`, meant different things on two endpoints.

**Fix.** A single `moneyFor(from, to)` function now defines every financial figure on a **cash basis**: a top-up is counted when it arrives, and spending the credit later adds nothing. Every financial endpoint reads it. Fields with ambiguous names were removed, not redefined. A regression test walks seven days and asserts that headline = sum of ledger rows = sales + top-ups = the analytics trend.

**Result.** Every screen shows the same figure for the same period, and each figure states its basis.

---

## 6.3 Walk-ins were invisible to the owner's reports

**Symptom.** A coach who worked walk-ins all month showed **0 sessions** and 0% utilisation. The occupancy card showed 1 child on a day when 40 passed through.

**Root cause.** Walk-ins (`Guest`) are the large majority of visits (about 330 against 11 reservations in one 30-day window), but aggregates were written against `Booking`. The shared series did include guests, but it dropped every field except the amount, which pushed each consumer back to bookings only.

**Fix.** The shared `revenueSessions()` series now carries coach, phone, parent name and duration for guests, so each consumer's fix was one line. Customer identity was unified on the **normalised phone number**. Guards prevent double-counting: walk-in spend comes from the visit, never from its payment row. A single `presentCount()` drives every occupancy figure.

**Result.** The same coach now reads **70 sessions, about 28.8M toman, 65% utilisation**. Busy-hour charts now describe all traffic, not about 3% of it.

---

## 6.4 The device clock, UTC and the Jalali calendar

**Symptom.** Past time slots could be booked. The rota showed the literal numbers 7 to 13 instead of dates. The analytics calendar labelled August as *Aban*, about four months off. The first three and a half hours of each day's trading were filed under the previous day.

**Root cause.** Dates were taken from the device clock. Server "midnight" was UTC midnight, which is 03:30 in Tehran. Gregorian month buckets were labelled with Persian month names, but a Jalali month is not a fixed offset from a Gregorian one.

**Fix.** A shared time module provides Tehran-anchored primitives (`tehranDateKey`, `tehranStartOfDay`, `isPastSlot`) and real Jalali arithmetic (`jalaliMonthKey`, `jalaliMonthRange`), used by both client and server. Two details mattered: `hourCycle: "h23"` instead of `hour12: false`, which some ICU builds render as "24:00"; and formatting date-only keys at **Tehran midday**, so devices in positive UTC offsets don't fall back a day.

**Result.** Correct dates, correct buckets and correct day boundaries everywhere. The server rejects past slots even if a client offers them.

---

## 6.5 Persona leak in shared navigation

**Symptom.** A manager tapping "message" on a family's file was taken into the **parent portal** in the middle of a session.

**Root cause.** Components such as the coach profile are mounted by several personas but navigated to a hard-coded parent screen id. A second instance of the same pattern sent supervisors back to a manager-only screen.

**Fix.** Shared components now navigate only through `nav.back()` or a role-derived destination (`messagesScreenFor(role)`). This was recorded as a codebase rule covering every multi-persona component.

---

## 6.6 Every bottom sheet's submit button was hidden on phones

**Symptom.** On mobile, "add child" and "add coach" couldn't be completed because the submit button was not visible.

**Root cause.** The page-transition animation used `animation-fill-mode: both`, which keeps the final keyframe applied and so creates a **permanent stacking context**. Sheets rendered inside it, so their `z-index` resolved within that context, and the tab bar, a sibling higher up, painted over the bottom 84 px.

**Fix.** The `Sheet` primitive now renders through a React portal into the screen root, as a true sibling of the tab bar. One change fixed all nine sheets. Rule: overlays never rely on `z-index` from inside an animated container.

---

## 6.7 A staff action filed under the wrong family

**Symptom.** Bookings created at the desk were invisible to the family, priced against the cashier's loyalty record and payable from the cashier's wallet.

**Root cause.** `POST /bookings` set `family = req.user.id`. That is correct when parents book for themselves, which hid the bug until staff acted on someone else's behalf.

**Fix.** The family is now derived from the **child**. The same audit found walk-in prices accepted from the client. Prices now come from server pricing, and `amount` was removed from the writable fields.

---

## 6.8 Analytics performance

**Symptom.** Owner dashboards slowed as history grew.

**Root cause.** The shared revenue series was **unbounded**: it read every booking and guest ever recorded, and the dashboard called it three times per request.

**Fix.** Callers pass their earliest required instant, and multi-window callers pre-load once and slice in memory. Compound indexes were added to match the query shapes, and lifetime questions moved to `$group` aggregates.

**Result.** Median response times after the change: **dashboard 7 ms, analytics 8 ms, revenue trend 2 ms**.

---

[← Security](05-security.md) · [Next: Quality & Testing →](07-quality-and-testing.md)
