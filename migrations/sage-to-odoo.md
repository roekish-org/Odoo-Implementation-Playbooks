# Sage → Odoo Migration Checklist

> A reproducible **migration runbook** for moving a French SMB from Sage (100, 50, 50cloud, or Compta) to Odoo. **v1 outline** — battle-tested phases and reconciliation discipline. Per-Sage-version mapping detail will land in v2.

| Item | Value |
|---|---|
| Playbook version | v1 — 2026-06-04 (outline) |
| Source systems supported | Sage 100, Sage 50, Sage 50cloud, Sage Compta (others by analogy) |
| Target | Odoo 19.0 (current stable, recommended) — also 18.0 / 17.0; Enterprise recommended for FEC + audit trail |
| Region | France / EU |
| Effort range | 40–120 hours depending on data volume, ledger complexity, and number of years migrated |
| Audience | Roekish team or in-house IT/finance running a Sage → Odoo cutover |

## Why this checklist exists

Migrations fail in the same places every time:

1. **Chart of accounts mapping** — assumed straightforward, isn't
2. **Open AR / AP** loaded as lump sums, breaking lettrage
3. **No reconciliation report** — nobody can prove the new books match the old
4. **Cutover happens before the dry-run is clean**
5. **Historical data** scope balloons mid-project

This playbook gates each phase on **explicit acceptance criteria**, with a mandatory **dry-run + reconciliation report** before cutover.

---

## Scope (in)

- Extract financial and master data from Sage
- Map Sage chart of accounts (PCG) to Odoo PCG
- Migrate partners (customers, vendors), products, and open invoices
- Migrate opening trial balance at the cutover date
- Migrate **N years of historical journal entries** (N agreed in scope)
- Dry-run import to a staging Odoo
- Reconciliation report signed off by customer's expert-comptable
- Final cutover and post-migration validation

## Scope (out)

- Migration of non-financial Sage modules (Sage Gestion Commerciale stock, Sage Paie payroll) — handled by the relevant Odoo module playbook (Inventory / Payroll-connector)
- Migration of attached documents from Sage GED — separate scope
- Migration of analytic accounting if Sage uses a non-PCG structure — needs custom mapping (+10h)
- Real-time bi-directional sync during a parallel-run period (we don't recommend; do a hard cutover)

---

## Prerequisites

### Decisions

- [ ] **Cutover date** (often year-end Dec 31 for French companies; sometimes month-end)
- [ ] **History scope** — how many fiscal years are imported into Odoo (recommended: 2 full closed years + current YTD)
- [ ] **Lot-level vs aggregate** for open AR/AP — always lot-level (per invoice)
- [ ] **Bank reconciliation handoff** — do we re-reconcile in Odoo or leave Sage as the source of truth for pre-cutover periods
- [ ] **Sage decommissioning** — kept read-only for X years (recommended: keep Sage in read-only mode for full 10-year statutory retention; alternatively archive the Sage DB + a PDF of every report)

### Data to receive (Sage exports)

- [ ] **FEC** for each fiscal year in scope (Sage 100/50 produce native FEC since 2014)
- [ ] **Balance générale** (trial balance) at cutover date
- [ ] **Grand livre** (general ledger) for the years in scope
- [ ] **Échéancier clients** (open AR) at cutover
- [ ] **Échéancier fournisseurs** (open AP) at cutover
- [ ] **Bank reconciliation statement** at cutover date
- [ ] **Fichier des écritures comptables** (could be FEC or proprietary CSV depending on Sage version)
- [ ] Master data: customers (`F_COMPTET` type C), vendors (`F_COMPTET` type F), products (if Sage Gescom is integrated)
- [ ] **Comptes de TVA** balances — to validate intra-period TVA reconciliation

### Tools

- A staging Odoo instance (Odoo.sh staging or local Docker) — never dry-run against prod
- Spreadsheet tool (LibreOffice Calc or Excel) for inspection
- Python + pandas (or your ETL of choice) for transformation
- A controlled storage bucket for the raw Sage exports — **never commit customer data to git**

---

## The 10 phases

### Phase 1 — Discovery & inventory (4–8h)

1. Confirm Sage version and edition (100, 50, 50cloud, Compta, hosted vs on-prem).
2. Confirm which Sage modules are in use (Compta, Gestion Commerciale, Paie, Immo, Moyens de paiement).
3. Document the customer's COA depth and any custom sub-accounts.
4. Identify analytic accounting usage (axes, sections, codes).
5. Identify any **non-standard journal types** (e.g., journal de bourse, journal de paie if Paie integrated).
6. Document the bank account count and bank journal layout.
7. **Deliverable**: Discovery memo signed by customer.

**Gate:** Discovery memo signed → proceed to Phase 2.

### Phase 2 — Mapping (6–16h)

Produce a written mapping document covering:

1. **Account mapping** — Sage account → Odoo account. Most are 1:1 within the PCG, but the customer's sub-accounts need explicit decisions:
   - Sage `411000VIDEO` → Odoo `411000` with a **sub-ledger** by partner (Odoo's pattern)
   - Sage analytic axes → Odoo analytic plans
2. **Journal mapping** — Sage journal codes → Odoo journals (Sales, Purchase, Bank, Misc, …).
3. **Partner mapping** — Sage compte tiers → Odoo res.partner. Decision: merge duplicates yes/no.
4. **Product mapping** — only if Sage Gescom in scope.
5. **TVA mapping** — Sage TVA codes → Odoo taxes (mind grid mapping; see [Accounting playbook](../accounting/README.md) Step 4).
6. **Currency mapping** — usually all-EUR; if not, declare the rates used.
7. **Date handling** — cutover date, historical period boundaries.
8. **Deliverable**: mapping spreadsheet, signed by customer's expert-comptable.

**Gate:** Mapping signed → proceed to Phase 3.

### Phase 3 — Extraction (4–8h)

1. Pull all Sage exports listed in *Prerequisites → Data to receive*.
2. Validate file integrity:
   - FEC parses without errors (one row per movement, debits = credits at journal level)
   - Trial balance balances (Σ debits = Σ credits)
   - Open AR sum == receivable balance per the trial balance
   - Open AP sum == payable balance per the trial balance
3. Stash raw files in customer's secure share. **Do not commit to git, do not paste into chat tools.**

**Gate:** Files validated and stashed → proceed to Phase 4.

### Phase 4 — Cleansing & transformation (8–16h)

Transform Sage data to Odoo's import format. Common operations:

1. **Account code padding** — Odoo requires consistent code length within a chart; pad with zeros if Sage codes are inconsistent.
2. **Partner deduplication** — fuzzy match on SIREN, then on name, then on email; produce a duplicates report for human review.
3. **Date normalization** — Sage's DDMMYYYY → ISO YYYY-MM-DD.
4. **Currency** — confirm sign convention (Sage uses positive for debit, negative for credit on some exports; Odoo expects `debit` and `credit` columns both positive).
5. **VAT validation** — for each customer/vendor, validate VAT number against the **VIES** API for intra-EU partners. Flag invalid ones for customer review (they will fail Odoo's fiscal-position auto-detect).
6. **TVA grid mapping** — for historical entries, map Sage's TVA code field to the Odoo tax that has the correct CA3 grid. Critical for prior-year CA3 audits.
7. **Partner aliases** — load Sage's reference as `ref` on the Odoo partner so the customer's team can still search by their familiar codes.

Output: import-ready CSVs (or XML/JSON if you use Odoo's external ID approach with `noupdate=False`).

**Tooling tip**: a thin Python pipeline with pandas, deterministic, version-controlled (separate private repo, **never** the public playbook repo), is more debuggable than visual ETL tools.

**Gate:** Cleansed files produced + cleansing report → proceed to Phase 5.

### Phase 5 — Dry-run import to staging (8–16h)

1. Spin up a fresh staging Odoo with [Accounting playbook](../accounting/README.md) baseline applied (l10n_fr, COA, taxes, fiscal positions).
2. Import in this order (dependencies matter):
   1. Partners (customers + vendors)
   2. Products (if Sage Gescom in scope)
   3. Account-level opening balances (single JE)
   4. Open AR — one JE per invoice, so lettrage works against future payments
   5. Open AP — same pattern
   6. Historical journal entries (year by year, oldest first)
3. Lock every imported period as it's validated.
4. **Time the import**. The full run becomes the basis for the cutover window estimate.

**Gate:** Staging import completes without errors → proceed to Phase 6.

### Phase 6 — Reconciliation report (4–10h)

**This is the gate everyone wants to skip. Don't.**

Produce a written report comparing Sage and staging Odoo:

| Check | Sage value | Odoo value | Diff | Acceptable? |
|---|---|---|---|---|
| Trial balance: total debit | … | … | … | Must be 0.00 |
| Trial balance: total credit | … | … | … | Must be 0.00 |
| Open AR total | … | … | … | Must be 0.00 |
| Open AR count | … | … | … | Must be 0 |
| Open AP total | … | … | … | Must be 0.00 |
| Open AP count | … | … | … | Must be 0 |
| Bank balance at cutover | … | … | … | Must be 0.00 |
| TVA collectée — last filed period | … | … | … | Must be 0.00 |
| TVA déductible — last filed period | … | … | … | Must be 0.00 |
| P&L: total revenue, last closed FY | … | … | … | ≤ ±0.01 per account |
| P&L: total expenses, last closed FY | … | … | … | ≤ ±0.01 per account |
| BS: equity at cutover | … | … | … | Must be 0.00 |
| Sample 20 invoices: same amount in both | 20/20 | 20/20 | 0 | Must be 20/20 |
| Sample 20 payments: same reconciliation pattern | 20/20 | 20/20 | 0 | Must be 20/20 |

**Diff > 0 on any "must be 0.00" row → stop, root-cause, fix mapping or transformation, redo dry-run.**

Sign-off: customer's expert-comptable signs the reconciliation report **in writing**.

**Gate:** Reconciliation report signed → proceed to Phase 7.

### Phase 7 — Cutover plan (2–4h)

Write a written cutover plan with:

1. **Cutover window** — typically 3 days minimum (Friday evening start, full Saturday/Sunday, Monday morning go-live).
2. **Freeze in Sage** — set Sage to read-only at the cutover timestamp; document who has access and that no new entries are allowed.
3. **Delta extract** — if there's a gap between the dry-run extract date and the cutover date, plan a delta extract (entries posted between those dates).
4. **Communications** — who tells which user when, and what they can/can't do during cutover.
5. **Rollback plan** — if go-live fails, how do we revert (most cases: customer keeps using Sage; Odoo is rebuilt from a fresh backup).
6. **Sign-off matrix** — who signs off cutover (typically: customer's CFO + Roekish lead consultant).

**Gate:** Plan signed → proceed to Phase 8.

### Phase 8 — Production import (4–8h)

Cutover day:

1. Freeze Sage (read-only).
2. Take the final extracts (delta from Phase 7 + final trial balance).
3. Backup the staging Odoo to confirm the import procedure works end-to-end.
4. Run the import on production Odoo using the **same procedure** as the dry-run.
5. **Lock dates** — set the cutover date as the lock date for all imported periods.
6. **Hash chain check** — run *Accounting → Reporting → Audit Reports → Inalterability Check*. Must be green.

### Phase 9 — Post-migration validation (4–6h)

Re-run the **reconciliation report** from Phase 6, now on production:

| Check | Sage (frozen) | Odoo (prod) | Diff | Status |
|---|---|---|---|---|
| All 14 checks from Phase 6 | … | … | … | … |
| Plus: First post-cutover transaction posts cleanly | — | OK | — | ✓ |
| Plus: First post-cutover bank reconciliation works | — | OK | — | ✓ |
| Plus: User logins (key user + 1 random user) succeed | — | OK | — | ✓ |
| Plus: Standard reports (P&L, BS, GL) render | — | OK | — | ✓ |
| Plus: FEC export for last closed year valid | — | OK | — | ✓ |
| Plus: Lock dates set; hash inalterability green | — | OK | — | ✓ |

Sign-off the cutover acceptance.

### Phase 10 — Hypercare (4–8h)

- First **5 business days** post go-live: dedicated channel, daily check-in with the customer's finance team.
- Next **15 business days**: less intensive, every-other-day check-in.
- **End-of-month** post go-live: walk the customer through their first full close in Odoo.
- **Sage decommissioning** — written confirmation 30 days post go-live that Sage is read-only and the customer's team has finished any historical lookups.

---

## Validation checklist

- [ ] Discovery memo signed
- [ ] Mapping document signed by expert-comptable
- [ ] Raw Sage exports validated and securely stashed
- [ ] Cleansing report produced; duplicates reviewed by customer
- [ ] VIES validation done on intra-EU partners
- [ ] Dry-run import completes on staging without errors
- [ ] **Reconciliation report signed by expert-comptable**
- [ ] Cutover plan signed by customer CFO
- [ ] Sage frozen at cutover
- [ ] Production import succeeds
- [ ] Lock dates + hash chain set/green
- [ ] Post-migration reconciliation matches
- [ ] Hypercare channel live, daily check-ins booked for first 5 days
- [ ] Sage decommissioning written confirmation at D+30

---

## Common failure modes (and how this checklist prevents them)

| Failure mode | What this checklist enforces |
|---|---|
| Open AR loaded as lump sum, lettrage breaks | Phase 5 step 2.4: "one JE per invoice" |
| Sub-accounts in Sage flattened, audit trail lost | Phase 2 step 1: explicit decision per sub-account |
| TVA grids wrong on historical entries | Phase 4 step 6: per-historical-entry TVA grid mapping |
| Cutover before reconciliation clean | Phase 6 gate: report must be signed |
| User can't find a customer they remember | Phase 4 step 7: load Sage reference as `ref` |
| Hash chain broken by post-import manual edit | Phase 8 step 6: hash check immediately after import |

---

## Hours estimate (range, line by line)

| Phase | Description | Hours (low → high) |
|---|---|---|
| 1 | Discovery & inventory | 4 → 8 |
| 2 | Mapping | 6 → 16 |
| 3 | Extraction | 4 → 8 |
| 4 | Cleansing & transformation | 8 → 16 |
| 5 | Dry-run import | 8 → 16 |
| 6 | Reconciliation report | 4 → 10 |
| 7 | Cutover plan | 2 → 4 |
| 8 | Production import | 4 → 8 |
| 9 | Post-migration validation | 4 → 6 |
| 10 | Hypercare | 4 → 8 |
| | **Total range** | **48 → 100** |

> Low end = simple Sage Compta, 1 FY of history, ~500 partners, <5k journal lines. High end = Sage 100 with Gescom integrated, 3 FY of history, >5k partners, 50k+ lines. Anything beyond that is a separate scoping exercise.

> **Why this range, not a single anchor?** Sage scope varies more than a standard greenfield setup. Quoting a single anchor would be dishonest. We give you ranges; we tighten in discovery.

---

## What we'll do for you

We are [Roekish](https://roekish.com). We deliver Sage → Odoo migrations against this exact checklist, with a **reconciliation report you can show your expert-comptable** and a **signed cutover acceptance**.

What you get:

- A typed extract + mapping + dry-run on a staging Odoo
- A reconciliation report signed off **before** any production cutover
- A documented cutover plan with rollback
- The first month-end close in Odoo, walked through with your team
- Sage decommissioning confirmation at D+30

**Reach out: hello@roekish.com** — we run a free discovery call to estimate hours and confirm whether this is a "standard" migration or needs extra scoping.

---

## Roadmap

- **v2** — concrete per-Sage-version export commands (Sage 100, Sage 50, Sage Compta), screenshots of each export menu, a sample Python pandas pipeline (anonymized) for the cleansing step.
- **v3** — same checklist adapted for Cegid, EBP, and ISACOMPTA.

PRs welcome.

## Changelog

- **2026-06-04 — v1**: initial outline. 10 phases, gates, reconciliation discipline, hours range.
