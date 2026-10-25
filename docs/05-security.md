# 5. Security

> **Summary.** The platform handles children's information, family contact details and money, so security was treated as a product requirement. Controls are layered: transport and header hardening, input sanitisation, authentication, role- and rank-based authorisation, per-object ownership checks, and server-side computation of every value that carries money or privilege. The controls were checked with a **black-box audit against a running server**, not by reading the source alone.

[← Back to README](../README.md)

---

## 5.1 Threat Model

| Asset | Primary threats | Principal controls |
|---|---|---|
| Child and family records | Cross-family data access (IDOR), over-broad staff visibility | Ownership filters on every query; projections per role |
| Money (prices, wallet, till) | Client-side price tampering, self-credit, double-counting | Server-computed prices; top-ups restricted to managers; single settlement path |
| Accounts | Credential stuffing, privilege escalation, account takeover via reset | Rate limiting, bcrypt, `role` ignored at registration, rank-gated reset codes |
| Staff personal data | Leakage of phone numbers, pay or schedules to parents | Parent-facing projections; staff-only rota and swap routes |
| Availability | Request floods, oversized bodies | Account-keyed rate limiting, 3 MB body cap, bounded analytics queries |

## 5.2 Controls

**Transport and headers.** `helmet` with a strict Content Security Policy (`default-src 'self'`, `object-src 'none'`), HSTS for one year including subdomains, `nosniff`, `SAMEORIGIN` framing, and no `X-Powered-By`. TLS is terminated at the CDN.

**Input handling.** `express-mongo-sanitize` strips `$` and `.` operators from body, query and params. Every write passes through a per-resource **field whitelist**. Malformed JSON returns `400` and oversized bodies return `413`.

**Authentication.** Phone and password login, bcrypt hashing, signed JWTs. The server **refuses to start in production** without a real signing secret. Generated credentials use `crypto.randomInt`, not `Math.random`.

**Authorisation.**
- `authorize(...roles)` at the route level, plus a **rank model** for actions on other accounts. A supervisor cannot mint a password-reset code for the owner.
- Object-level ownership checks: a parent can only read or modify their own children and bookings.
- **Time-bound privilege:** cash-desk rights come from the current rota shift and expire with it.
- The user record is reloaded on every request, so revocation takes effect immediately.

**Least-privilege data shaping.** Parents receive a reduced coach projection, with no linked account, phone, pay or utilisation. Plain cashiers receive till responses without takings or ledger amounts.

**Rate limiting.** Authentication has its own strict limiter. General API traffic and the owner analytics API have separate production ceilings. Limits are **keyed by account rather than IP**, because the whole centre shares one connection.

## 5.3 Audit Methodology and Results

The audit was run against a live local server with seeded accounts for every role. Each probe was an actual HTTP request.

| Area | Method | Result |
|---|---|---|
| **Authorisation matrix** | 20 read and 16 write endpoints × 5 identities (anonymous, parent, coach, supervisor, owner) | Correct. Anonymous users reach only public classes, events and landing content. Parents and coaches are refused all money and operations routes. Analytics is owner-only. |
| **IDOR** | Family B attempts to read, modify, delete, pay for and cancel family A's records | All refused with `403`; data intact |
| **Privilege escalation** | `role` supplied at registration; self-granting cashier rights; self top-up | Ignored or `403` |
| **NoSQL injection** | Operator payloads at login and in query filters | Rejected at login. Stripped from queries without bypassing ownership filters. |
| **Information disclosure** | Response shapes per role | Three leaks found and closed: coach records exposed staff phone and pay data to parents, and rota and swap routes (with free-text reasons) were readable by parents. A regression test was added. |
| **Account takeover** | Password-reset code issuance across roles | One finding fixed: reset codes are now rank-gated and require a pending reset request |
| **Dependencies** | `npm audit` | **14 vulnerabilities (7 high) → 0** through lock-file-only transitive upgrades |

## 5.4 Accepted Risks (Product Decisions)

These were documented as deliberate decisions for a single small centre, to revisit if the platform becomes multi-site:

- Staff can see all children and all session reports. Bookings themselves remain scoped, so a coach sees only their own.
- Only the owner can delete a child record.

## 5.5 Operational Security Practices

- Secrets live only in the CI secret store and the hosting environment. They are never committed, and `.env` is ignored by git.
- Production data is never used for development or testing. Seed scripts that create demonstration accounts are local-only by policy.
- A backup is taken before each schema migration or destructive operation.
- This public repository contains no source code, credentials, hostnames or customer data.

---

[← Key Features](04-key-features.md) · [Next: Engineering Case Studies →](06-engineering-case-studies.md)
