# 10. Architecture Decision Record

> **Summary.** This log records the decisions that shaped the platform. Each entry gives the context it was made in, the option chosen, the alternatives considered, and the consequences the team accepted. Most of them come back to one constraint: a small business with one engineer needs a system that is **correct about money and time** and **cheap to operate**, more than one that is maximally scalable.

[← Back to README](../README.md)

---

## Decision Index

| # | Decision | Status |
|:---:|---|---|
| ADR-01 | [Modular monolith: SPA and API in one process](#adr-01--modular-monolith-spa-and-api-in-one-process) | Accepted |
| ADR-02 | [MongoDB with invariants enforced in the domain layer](#adr-02--mongodb-with-invariants-enforced-in-the-domain-layer) | Accepted |
| ADR-03 | [The server is the only authority on price](#adr-03--the-server-is-the-only-authority-on-price) | Accepted |
| ADR-04 | [A single cash-basis money model](#adr-04--a-single-cash-basis-money-model) | Accepted, supersedes an accrual-based design |
| ADR-05 | [In-process scheduler for end-of-day work](#adr-05--in-process-scheduler-for-end-of-day-work) | Accepted |
| ADR-06 | [Till access derived from the rota, not from a role](#adr-06--till-access-derived-from-the-rota-not-from-a-role) | Accepted, supersedes a permanent flag |
| ADR-07 | [Stateless JWT, with the user re-read on every request](#adr-07--stateless-jwt-with-the-user-re-read-on-every-request) | Accepted |
| ADR-08 | [One shared time module on Tehran time and the Jalali calendar](#adr-08--one-shared-time-module-on-tehran-time-and-the-jalali-calendar) | Accepted |
| ADR-09 | [Business parameters are data, not code](#adr-09--business-parameters-are-data-not-code) | Accepted |
| ADR-10 | [Managed PaaS in the domestic region, with CI-built artefacts](#adr-10--managed-paas-in-the-domestic-region-with-ci-built-artefacts) | Accepted |

---

## ADR-01 · Modular monolith: SPA and API in one process

**Context.** The platform has one customer site, one engineer and a limited hosting budget. Delivery speed and operational simplicity matter more than independent scaling of components.

**Decision.** One Node.js process serves both the compiled React SPA and the `/api/*` REST API. Inside that process, business rules are split into small single-purpose domain modules (`pricing`, `schedule`, `cashdesk`, `roles`, `suspension`, `nightly`), and every route imports them.

**Alternatives considered.** Separate frontend hosting with a standalone API; microservices per domain.

**Consequences.**
- ✅ One deployable unit and one container, with no CORS or cross-site cookie configuration.
- ✅ The module boundaries give most of the clarity of separate services without the network hops.
- ⚠️ Frontend and API scale together. That is acceptable at single-site load, and the module boundaries leave a clean seam if the platform is ever split.

---

## ADR-02 · MongoDB with invariants enforced in the domain layer

**Context.** The domain changed weekly during the first months: rooms were removed, walk-ins were promoted to first-class records, and ratings moved from per-coach to per-session. The schema had to evolve without a migration framework slowing every change.

**Decision.** MongoDB through Mongoose. Integrity rules that a relational schema would express as constraints live in the domain layer instead, backed by targeted indexes (for example a sparse unique index so each session can be rated only once). Writes go through per-resource field whitelists.

**Alternatives considered.** PostgreSQL with a migration tool.

**Consequences.**
- ✅ Fast schema evolution, with documents shaped like the screens that read them.
- ⚠️ No foreign keys, so cascades (`cascade.js`) and ownership checks are explicit code, and each is covered by tests.
- ⚠️ Migrations are idempotent scripts, rehearsed on a copy of the data first. Mongoose strict mode silently drops `$unset` on removed fields, so cleanup migrations go through the raw collection.

---

## ADR-03 · The server is the only authority on price

**Context.** Packages, loyalty tiers, first-booking discounts and quiet-hour discounts combine into one price, and the owner can change any of them at any time.

**Decision.** One function, `quoteFor(family, duration, slot)`, computes every price. The quote endpoint and the booking endpoint both call it. Prices sent by the client are ignored, and walk-in amounts are not writable fields at all.

**Alternatives considered.** Pricing on the client with server-side validation.

**Consequences.**
- ✅ The quoted figure always equals the charged figure, and a modified client can't change what it pays.
- ✅ Discount reasons come back with the quote, so the payment screen explains the price.
- ⚠️ Every pricing screen needs a round trip. A 30-second Settings cache with explicit invalidation on save keeps that cheap.

---

## ADR-04 · A single cash-basis money model

**Context.** At one point five screens computed revenue independently. Some counted sessions delivered (accrual), others counted money received, and wallet top-ups were counted twice: once on arrival and again when the credit was spent. See [Case Study 6.2](06-engineering-case-studies.md#62-one-money-model-instead-of-five).

**Decision.** One function, `moneyFor(from, to)`, defines every financial figure on a **cash basis**. Money counts when it arrives, so spending existing wallet credit adds nothing. The dashboard, analytics, day archive, till and P&L all read it.

**Alternatives considered.** Accrual accounting (revenue when the session is delivered), with prepaid credit tracked as a liability.

**Consequences.**
- ✅ One number per period across the whole product, and it matches the physical till.
- ✅ The owner, who is not an accountant, gets figures that match the cash in the drawer.
- ⚠️ Prepaid credit not yet used is reported separately as *unused credit* so the cash basis doesn't hide it.

---

## ADR-05 · In-process scheduler for end-of-day work

**Context.** The till must be sealed at closing time, the next day's register opened with the carried float, and stale attendance closed. The hosting platform offers no cron.

**Decision.** An in-process scheduler (`nightly.js`) polls every 60 seconds and runs idempotent jobs. As a backstop, the register also rolls over lazily on first read if a night was missed.

**Alternatives considered.** A single `setTimeout` to midnight; an external cron service calling an endpoint.

**Consequences.**
- ✅ It corrects itself across restarts, container sleeps and clock drift, with no extra infrastructure.
- ✅ Jobs never take money on their own. They only close and roll over.
- ⚠️ It assumes a single application instance. Running more than one would need a lock or a leader.

---

## ADR-06 · Till access derived from the rota, not from a role

**Context.** Cash-desk access was originally a permanent profile flag. It was granted and never revoked, so access grew silently over time.

**Decision.** A supervisor marks a **shift** as the till shift. Access exists only during that shift window, on that weekday, in Tehran time, and then ends by itself. `cashdesk.js` is the single authority, and every money route asks it.

**Alternatives considered.** A cashier role; a manually toggled flag.

**Consequences.**
- ✅ Least privilege by default: no standing access to money.
- ✅ The person shown as "on the till" comes from the rota, not from whoever opened the drawer.
- ⚠️ Guards that check access hit the database, so they are async. The UI reads till status at sign-in, so a shift that starts mid-session needs a reload to show the tab.

---

## ADR-07 · Stateless JWT, with the user re-read on every request

**Context.** Staff are promoted, suspended or have access revoked during the working day, and those changes must apply at once.

**Decision.** Authentication uses JWTs (HttpOnly cookie, with a bearer fallback), bcrypt password hashing and a refresh flow. Authorisation does **not** trust claims inside the token: the `protect` middleware reloads the user on every request, and role and rank guards run on that fresh record.

**Alternatives considered.** Role claims embedded in the token, valid until expiry.

**Consequences.**
- ✅ A suspension or demotion takes effect on the very next API call.
- ⚠️ Each request costs one indexed primary-key read, which is negligible at this scale.

---

## ADR-08 · One shared time module on Tehran time and the Jalali calendar

**Context.** The business runs on Iranian civil time and the Jalali calendar. The server runs on UTC, and devices carry arbitrary clocks. See [Case Study 6.4](06-engineering-case-studies.md#64-the-device-clock-utc-and-the-jalali-calendar).

**Decision.** `schedule.js` is imported by **both** client and server. It provides Tehran-anchored day keys and minutes, slot generation, past-slot detection and real Jalali month arithmetic, all built on `Intl` with an explicit `Asia/Tehran` time zone.

**Alternatives considered.** A date library on each side; trusting the device clock.

**Consequences.**
- ✅ If the screen offers a slot, the server accepts it, and the reverse holds too.
- ✅ Reports bucket by real Jalali months, not by Gregorian months with Persian labels.
- ⚠️ ICU differences between runtimes need defensive handling (`hourCycle: "h23"`, stripping era suffixes from years).

---

## ADR-09 · Business parameters are data, not code

**Context.** Prices, packages, loyalty tiers, programmes, opening hours, discounts and capacity started as code constants, so every commercial change needed a deploy.

**Decision.** All of them live in one `Settings` document, which the owner edits in the app. Writes go through an explicit key whitelist, and the client applies settings once at boot.

**Consequences.**
- ✅ The owner changes the commercial model without an engineer.
- ⚠️ A new setting must be added to the write whitelist and the cache-invalidation list, or it is silently dropped. This is documented as a rule and covered by tests.

---

## ADR-10 · Managed PaaS in the domestic region, with CI-built artefacts

**Context.** The users and the business are in Iran. There is no operations team, and a build step on the platform is one more thing that can time out or fail mid-deploy.

**Decision.** Host on a managed PaaS in the Iran region, behind a CDN with managed TLS. GitHub Actions builds the client on the CI runner and uploads a ready artefact. Deployment is path-filtered to the application directory. Separate manually triggered workflows handle logs, backups and migrations, so production needs no shell access.

**Alternatives considered.** A self-managed VPS; building on the platform.

**Consequences.**
- ✅ Low latency for local users, no servers to patch, and repeatable deploys.
- ✅ Operational tasks run as audited workflows, not ad-hoc shell sessions.
- ⚠️ Some platform behaviour (for example, environment changes applying only on redeploy) had to be learned and documented. See [Deployment & Operations](08-deployment-and-ops.md).

---

[← Roadmap](09-roadmap.md) · [Back to README](../README.md)
