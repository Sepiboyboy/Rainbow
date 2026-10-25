# 7. Quality and Testing

> **Summary.** Quality is backed by an end-to-end API integration suite of 101 tests, which runs over HTTP against a live server and database. It also rests on two practices: rehearsing migrations before they touch production, and verifying fixes in a real browser and against live data rather than inferring them from source. Production defects are fixed together with a regression test that reproduces them.

[← Back to README](../README.md)

---

## 7.1 Test Strategy

| Layer | Tooling | Scope |
|---|---|---|
| **API integration** | Node.js built-in test runner + native `fetch` | 101 tests over HTTP: authorisation, pricing, booking lifecycle, settlement, cash desk, rota rules, analytics consistency, accounting |
| **Client static analysis** | Babel parser + traverse | Fails if any client identifier has no binding in scope. The bundler treats such identifiers as globals and builds successfully, so this check catches what the build cannot. It needs no server, so it can gate deploys. |
| **Load benchmark** | Custom read-only benchmark | Concurrent simulated users against heavy endpoints. It issues GET requests only, so it is safe to run against production. |
| **Browser verification** | Headless Chrome against seeded data | Visual and behavioural confirmation of UI fixes at phone and desktop widths |

The suite deliberately tests through the HTTP boundary, not by calling functions directly, so middleware, authorisation, validation and serialisation are all exercised the way a client exercises them.

### Representative test themes

- **Contract invariants:** "the room card counts walk-ins, like every other screen"; "Tehran is the clock, not the device"; "a slot that has already started cannot be booked or staffed".
- **Money consistency:** across seven days, headline = sum of ledger rows = sales + top-ups = analytics trend. `sum(topCustomers) ≤ collected`.
- **Security regressions:** parents can't read staff projections; reset codes are rank-gated; a client-sent `dueAt` or price is ignored.
- **Idempotency:** paying twice, approving payroll twice (`409`), re-running migrations.

When behaviour changes intentionally, the corresponding tests are **rewritten to the new contract, not deleted**, so the invariant underneath is kept.

## 7.2 Growth of the Suite

The suite grew with the product: 43 → 57 → 74 → 84 → 101 tests across successive releases. Each release added the tests that pinned down its own fixes.

## 7.3 Performance Engineering

| Change | Effect |
|---|---|
| Time-bounded revenue series (callers pass their earliest instant) | Removed full-collection scans on every dashboard request |
| Load once, slice many windows (`moneyFor(from, to, preloaded)`) | Dashboard computes today, yesterday and month from one read |
| Compound indexes: `Booking {status, createdAt}`, `{createdAt}`; `Payment {createdAt}`, `{kind, createdAt}`; `Guest {checkInAt}` | Index-backed range queries |
| Lifetime metrics via `$group` aggregation | Constant-size responses for repeat-rate questions |
| Batched enrichment of ledger rows | No N+1 queries when filling display fields |

Measured medians after optimisation: **dashboard 7 ms · analytics 8 ms · revenue trend 2 ms**. The production client bundle is about 400 kB.

## 7.4 Migration Discipline

- Migrations live in a single **idempotent** script. Re-running it reports zero changes.
- Each migration is **rehearsed on a database dump** prepared to look like production before it is used for real.
- Fields removed from schemas are dropped through the raw collection, avoiding Mongoose strict-mode stripping.
- Semantics are not inverted carelessly. When a presence toggle was replaced by suspension, the old `available: false` was **not** carried over as `suspended: true`. Production data showed that would have suspended the busiest coach on the first day.
- New settings fields use code-level fallbacks, because schema defaults do not reach documents that already exist.

## 7.5 Verification Practices

- Fixes are confirmed against a running system: HTTP probes, browser checks at phone and desktop widths, and production health checks after each deploy.
- Figures shown to the owner are cross-checked against each other on screen (for example, two plus 60 equals 62 children, matching the room list).
- Order-dependent tests are made self-contained, for example by topping up a wallet before testing idempotent payment.

---

[← Engineering Case Studies](06-engineering-case-studies.md) · [Next: Deployment & Operations →](08-deployment-and-ops.md)
