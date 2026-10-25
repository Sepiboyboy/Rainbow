# 9. Roadmap

> **Summary.** The core operational platform is complete and in production use. Planned work focuses on three areas: extending the platform to a native mobile client, deepening owner accounting so a professional bookkeeper can use it, and adding operational safeguards for platform administration.

[← Back to README](../README.md)

---

## 9.1 In Progress

| Item | Description |
|---|---|
| **Platform administration role** | A developer-level role for platform maintenance, with a maintenance-mode switch and a guarded data-reset procedure (typed confirmation, password re-verification and an automatic full export). Built and tested locally; staged rollout pending. |

## 9.2 Planned

| Area | Item | Rationale |
|---|---|---|
| Mobile | **Native Android client** connected to the live API (an Expo / React Native prototype exists, currently offline-only) | Parents primarily use phones; push notifications for session start, end and reports |
| Accounting | **CSV / spreadsheet export** of ledger, expenses and P&L | Lets an external bookkeeper work from the system's figures |
| Accounting | **Recurring expenses** (rent, utilities) | Removes monthly manual entry |
| Accounting | **Receivables view** | Track unsettled balances explicitly |
| Operations | **Dated rota exceptions** (absences, one-off cover) | The rota is a weekly template today, which is why hourly pay is labelled an estimate |
| Communication | **Targeted parent messaging** from the parent portal | The current parent entry point opens a general "message the centre" thread |
| Scale | **Multi-site readiness** | Revisit staff-wide visibility of children and reports, which was accepted for a single centre |

## 9.3 Principles for Future Work

The rules that held through the project's history carry forward:

1. **One question, one authority.** New rules go into a single domain module that every caller imports.
2. **The server decides** prices, deadlines, privileges and totals.
3. **Figures state their basis.** A number shown to the owner names what it measures and over what period.
4. **Every fix ships with a test** that reproduces the original defect.

---

[← Deployment & Operations](08-deployment-and-ops.md) · [Back to README](../README.md)
