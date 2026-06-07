# Go-Live Checklist

> The last gate before an Odoo system carries real business. Work top to bottom; do not
> go live with an unchecked **blocker**. Copy per project.

| Field | Value |
|---|---|
| Project | |
| Go-live date/time | _pick a low-traffic window_ |
| Rollback owner | |
| Cutover lead | |

## T-7 days — readiness

- [ ] **UAT signed off** by the business owner ([template](./uat-sign-off.md)) — *blocker*
- [ ] No open ❌ defects (or all formally deferred and accepted)
- [ ] Production environment hardened ([setup guide §5](../setup/self-hosted-docker.md)):
      `list_db = False`, real secrets, TLS, `proxy_mode = True`, port 8069 not public — *blocker*
- [ ] Backups automated **and a restore has been tested** — *blocker*
- [ ] User accounts created; access-rights matrix applied
- [ ] End users trained; quick-reference guide shared
- [ ] Support path agreed (who, how, response time) for the first weeks

## T-1 day — data & config freeze

- [ ] Configuration frozen on staging; no further changes without change control
- [ ] Opening balances / master data validated (chart of accounts, partners, products, stock)
- [ ] Sequences set (invoice numbers, SO/PO) to continue from legacy where required — *blocker*
- [ ] 🇪🇺 FR fiscal: correct fiscal year, taxes, and FEC export verified
- [ ] Email (outgoing/incoming servers) tested from production
- [ ] Final pre-cutover backup taken

## Cutover day

- [ ] Maintenance window communicated to users
- [ ] Final data migration / import run and reconciled against source totals — *blocker*
- [ ] Smoke test in production: SO → delivery → invoice → payment completes
- [ ] Scheduled actions (crons) enabled and verified
- [ ] DNS / reverse proxy points to production; HTTPS valid
- [ ] Go / no-go decision recorded by the cutover lead

## T+1 to T+7 — hypercare

- [ ] Daily check-in with key users; triage issues fast
- [ ] First-day backups confirmed running
- [ ] Month-end / VAT process dry-run scheduled
- [ ] Lessons learned captured; deferrals scheduled

## Rollback plan

If a *blocker* surfaces during cutover:

1. Stop new data entry; communicate to users.
2. Restore the last pre-cutover backup ([restore drill](../setup/self-hosted-docker.md#6-backups-do-this-before-go-live-not-after)).
3. Re-point DNS/proxy to the legacy system if still available.
4. Reschedule; record the root cause before retrying.

---

*Part of the [Roekish Odoo Implementation Playbooks](../README.md).*
