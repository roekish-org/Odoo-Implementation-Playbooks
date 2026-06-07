# UAT Sign-Off Template

> User Acceptance Testing sign-off for an Odoo implementation. Copy this file per project,
> fill in the scope, and have the client business owner sign before go-live.
> The point of UAT is simple: **the people who will live in the system confirm it does their job.**

| Field | Value |
|---|---|
| Project | _e.g. ACME — Standard Inventory + FR Accounting_ |
| Odoo version | _19.0 / 18.0 / 17.0_ |
| Environment tested | _staging URL_ |
| UAT period | _start → end_ |
| Business owner (signs off) | _name, role_ |
| Implementation lead | _name_ |

## How to use

1. One row per business scenario, written in the client's language — not Odoo jargon.
2. The **client** executes each scenario on the staging environment.
3. Status: ✅ Pass · ⚠️ Pass with note · ❌ Fail (must be fixed or formally deferred).
4. Nothing goes live with an open ❌ unless it's explicitly listed under *Accepted deferrals*.

## Test scenarios

| # | Scenario (what the user does) | Expected result | Status | Notes / ticket |
|---|---|---|---|---|
| 1 | Create a sales order for a stocked product and confirm it | SO confirmed, delivery order generated | ☐ | |
| 2 | Validate the delivery; stock decreases | On-hand quantity reduced; move recorded | ☐ | |
| 3 | Create a customer invoice from the SO and post it | Invoice posted, correct FR tax + accounts | ☐ | |
| 4 | Register a customer payment and reconcile it | Invoice marked paid; bank statement reconciled | ☐ | |
| 5 | Run a vendor bill → payment cycle | Bill posted and paid; AP balance correct | ☐ | |
| 6 | Generate the legally required FR accounting export (FEC) | FEC file produced and validates | ☐ | |
| 7 | Each role logs in and sees only its permitted menus | Access rights match the agreed matrix | ☐ | |
| 8 | _Project-specific scenario_ | | ☐ | |

> Replace/extend rows to match the delivered scope. Cover every in-scope business process at least once.

## Accepted deferrals

Items the client agrees to handle after go-live (date + owner):

| Item | Reason for deferral | Owner | Target date |
|---|---|---|---|
| | | | |

## Sign-off

By signing, the business owner confirms that the in-scope scenarios above pass on the
staging environment and authorises promotion to production.

| | Name | Signature | Date |
|---|---|---|---|
| Client business owner | | | |
| Implementation lead | | | |

---

*Part of the [Roekish Odoo Implementation Playbooks](../README.md).*
