# Standard Odoo Accounting Setup (France)

> A reproducible playbook to take a French SMB from zero to a working **Odoo Accounting** go-live with French chart of accounts, TVA, FEC export, and bank reconciliation in **~36 hours** (€2,520 anchor price).

| Item | Value |
|---|---|
| Playbook version | v1 — 2026-06-04 |
| Odoo edition | **Enterprise recommended** (Accounting app is Enterprise-only). Community has *Invoicing* but not full Accounting. |
| Validated versions | Odoo 17.0, 18.0, 19.0 (19.0 = current stable, default for new projects) |
| Region | France (FR localization + PCG 2014) |
| Effort anchor | **36 hours** |
| Price anchor | **€2,520** (Roekish standard rate) |
| Audience | French SMB CFO / accountant, or in-house IT, executing a fresh Odoo Accounting setup |

> Price/hours are **anchors based on Roekish devis 2026-003**, not a quote. Discovery refines them.

---

## Scope (in)

This playbook covers a **single legal entity, French SMB**, going live on Odoo Accounting:

1. French localization installed (PCG, FR tax templates, FEC export)
2. Company configuration (legal info, VAT, currency, fiscal year)
3. Chart of accounts reviewed and adapted
4. Tax setup (TVA 20 / 10 / 5.5 / 2.1, autoliquidation, intra-EU)
5. Fiscal positions (FR, intra-EU, export, autoliquidation BTP)
6. Journals (sales, purchase, bank, cash, miscellaneous)
7. Bank account configuration + sync (Ponto/Plaid) or CFONB/OFX import
8. Customer & vendor configuration (payment terms, payment methods)
9. Opening balances import (trial balance at cutover date)
10. Standard reports: P&L, Balance Sheet, FEC, déclaration TVA (CA3/CA12)
11. User training (accountant + key users)
12. UAT, go-live, hypercare

## Scope (out)

Not included in the 36h anchor:

- Multi-company / consolidation
- Multi-currency beyond invoicing in foreign currency (no consolidation FX)
- Asset management module setup
- Budget module setup
- Cash basis accounting (régime micro-BNC) — most micro-entrepreneurs don't need Odoo
- Inter-company transactions
- Analytic accounting beyond cost centers (e.g., per-project P&L is +6h)
- Integration with external payroll (Silae, PayFit, Sage Paie) — separate scope
- Chorus Pro (public sector e-invoicing) integration — +8h
- Factoring / dailly bank integrations
- Migration from Sage / Cegid / EBP (see [`/migrations/`](../migrations/))

---

## Prerequisites

### Decisions with the customer

- [ ] **Fiscal year** start/end (most FR SMBs use Jan 1 → Dec 31; some use a *clôture décalée*, confirm)
- [ ] **Cutover date** — the date opening balances are taken (often year-end, sometimes month-end)
- [ ] **TVA regime** — *réel normal* (CA3 monthly/quarterly), *réel simplifié* (CA12 annual), or *franchise* (no TVA collected)
- [ ] **TVA periodicity** — monthly, quarterly, or annual
- [ ] **Bank reconciliation approach** — automated sync (Ponto, Plaid, native PSD2 connector), CFONB/OFX import, or manual
- [ ] **Currency** — EUR primary; any foreign-currency invoicing?
- [ ] **Number of journals needed** beyond defaults (e.g., separate journal for cash vs bank, or per branch)

### Data to receive from the customer

- [ ] **Trial balance at cutover date** — every active account with debit/credit balance
- [ ] **Open AR** — list of unpaid customer invoices (number, date, customer, amount, due date)
- [ ] **Open AP** — list of unpaid supplier invoices (same fields)
- [ ] **Open bank reconciliation items** — items posted in books but not yet on bank statement, and vice versa
- [ ] **K-bis** (RCS extract), **TVA number**, **APE/NAF code**, **SIRET**
- [ ] **Last filed CA3/CA12** for cross-checking opening TVA accounts

### Hosting & GDPR

For French customers we recommend **Odoo.sh EU region**, identical reasoning to the Inventory playbook (data residency, backups, staging, automated upgrades).

For accounting specifically, also confirm:

- **Backup retention** ≥ 10 years (French accounting record retention requirement: 10 years from end of fiscal year for accounting documents).
- **Immutability** of posted entries — Odoo's *Lock Date* and *Hash Chain* (see Step 11) cover this.

---

## Step-by-step setup

### Step 1 — Install French localization (1h)

1. *Apps → Settings → French localization* — search for `l10n_fr` and install.
2. Confirm the installed modules:
   - `l10n_fr` — PCG 2014 chart of accounts
   - `l10n_fr_account_vat_return` — TVA declaration (CA3) generator (Enterprise)
   - `l10n_fr_fec` — FEC export (mandatory for tax audits)
   - `l10n_fr_facturx` — Factur-X e-invoicing (recommended)
3. **Important:** the chart of accounts is loaded **once** at install. If you install on an existing company, the COA may not load — install on a fresh company or use the *Localization* link in *Settings → Companies*.

**Validation:** *Accounting → Configuration → Chart of Accounts* shows ~250 accounts following the PCG (411xxx Clients, 401xxx Fournisseurs, 445xxx TVA, etc.).

### Step 2 — Company configuration (1h)

*Settings → Companies → Update Info*:

- Legal name (matching K-bis)
- Address (matching SIRET registration)
- VAT number (`FR` + key + SIREN, 13 chars)
- SIRET (14 digits)
- APE/NAF code
- Currency: EUR
- Country: France
- *Accounting → Configuration → Settings*:
  - Fiscal Country: France
  - Currency: EUR
  - Default Sale Tax / Purchase Tax (set in Step 4)
  - Fiscal Year: set start/end day

**Validation:** *Settings → Companies* shows all fields populated. Print a test invoice — header reflects legal info correctly.

### Step 3 — Chart of accounts review (4h)

The PCG ships ~250 accounts. **Do not blindly accept it** — review with the customer's accountant:

1. *Accounting → Configuration → Chart of Accounts*.
2. **Disable** accounts the customer doesn't use (uncheck *Active*). Don't delete — the journal hashes reference account IDs.
3. **Add** accounts the customer's expert-comptable specifies, particularly:
   - `4456xxx` — sub-accounts for TVA déductible / collectée by rate
   - `467xxx` — comptes d'attente if used
   - Analytic accounts if multi-project
4. **Map** each account's *Type* correctly — this drives the P&L and BS layout:
   - 1xx → Capital, Provisions
   - 2xx → Immobilisations
   - 4xxx → Tiers (clients / fournisseurs / Etat)
   - 6xxx → Charges
   - 7xxx → Produits

**Validation:** *Accounting → Reporting → Balance Sheet* and *Profit and Loss* render with French headings. Cross-check against the customer's last annual report.

### Step 4 — Taxes (TVA) (4h)

The French localization ships default taxes:

| Rate | Code | Use case |
|---|---|---|
| 20% | `TVA 20%` | Standard rate |
| 10% | `TVA 10%` | Restaurants, transport, some renovation |
| 5.5% | `TVA 5.5%` | Food, books, energy-saving works |
| 2.1% | `TVA 2.1%` | Medicines reimbursed, certain media |
| 0% | `TVA 0%` | Exports, intra-EU B2B, exempt operations |

1. *Accounting → Configuration → Taxes*. Confirm each tax has:
   - Correct rate
   - Correct grid mapping (CA3 boxes 08, 09, 10, 11, etc.) — *Tax Grid* tab
   - Correct tax accounts (445660, 445710 for collected; 445620 for deductible)
2. **Autoliquidation** (reverse charge): create or confirm taxes for:
   - Intra-EU acquisitions (B2B): purchase auto-liquidée, posts both a debit and credit on 4452xx/4456xx
   - Sub-contracting in BTP (Article 283-2 nonies CGI): purchase auto-liquidée
3. **Set defaults** at the company level: *Settings → Default Sale Tax = TVA 20% Sales*, *Default Purchase Tax = TVA 20% Purchases*.

**Validation:**

- Create a test customer invoice for €100 HT with TVA 20%. Confirm: total TTC = €120, journal entry credits 707xxx €100 and credits 445710 €20, debits 411xxx €120.
- Create a test intra-EU purchase. Confirm autoliquidation posts equal debit and credit to TVA accounts (net cash effect = 0).

### Step 5 — Fiscal positions (2h)

Fiscal positions auto-remap accounts and taxes based on partner profile.

1. *Accounting → Configuration → Fiscal Positions*. Defaults:
   - *Régime National* — domestic France
   - *Régime Intracommunautaire* — intra-EU B2B
   - *Régime Extracommunautaire* — exports outside EU
2. Verify auto-detect rules: *Detect Automatically* on the position, with country group = EU (for intra-EU) or EXCLUDING EU (for export). *Foreign Tax ID Required* = yes for intra-EU.
3. Assign on each customer/vendor under *Sales & Purchase tab → Fiscal Position* — leave on "Automatic" if the auto-detect rules are correct.

**Validation:** create a German B2B customer with VAT number `DE123456789`. Place an order — invoice generates with TVA 0% and footer text noting *Autoliquidation Art. 283 CGI*.

### Step 6 — Journals (3h)

Default journals from `l10n_fr` cover most needs. Audit and adjust:

| Journal | Type | Code | Common adjustments |
|---|---|---|---|
| Customer Invoices | Sale | INV | rename to "Ventes" |
| Vendor Bills | Purchase | BILL | rename to "Achats" |
| Bank | Bank | BNK1 | rename to your bank's short code |
| Cash | Cash | CSH1 | only if cash is used |
| Miscellaneous Operations | Misc | MISC | always present |

Per journal, configure:

- *Short Code* — prefix on every entry number; max 5 chars
- *Default Income/Expense Account* — drives default lines on quick entries
- *Bank Account* (bank journal only) — link to the bank account record
- *Sequence* — starting number; format `INV/2026/00001` is the Odoo default and is fine

For French legal compliance (Article L102 B CGI), every posted journal entry must be **hashed and immutable**. Odoo Enterprise does this automatically on posted entries. Make sure *Lock Posted Entries with Hash* is **on** per journal (Step 11).

**Validation:** post one entry in each journal, then attempt to delete it — Odoo must refuse (button greyed out for posted entries).

### Step 7 — Bank account setup & synchronization (4h)

#### Option A — Native PSD2 / Ponto / Plaid synchronization (recommended)

1. *Accounting → Configuration → Add a Bank Account*.
2. Search for the customer's bank. Odoo Enterprise integrates with most French banks via PSD2 connectors (CIC, BNP, LCL, Crédit Agricole, La Banque Postale, Crédit Mutuel, Société Générale, etc.).
3. Authenticate (the customer must do this — credentials are not stored by Odoo, just the consent token). Many banks require an SCA every 90 days under PSD2.
4. Statements sync daily.

#### Option B — CFONB / OFX / CSV import

If sync isn't available:

1. *Accounting → Configuration → Journals → [Bank Journal] → Bank Statement → Create*.
2. Upload the CFONB 120/240 file from the bank's web portal.
3. Define an import template if recurring.

**Validation:** import or sync the last 30 days of statements. Run *Accounting → Bank → Reconcile*. Confirm at least 80% of lines auto-match against open invoices.

### Step 8 — Customers, vendors, payment terms & methods (2h)

1. **Payment terms**: *Accounting → Configuration → Payment Terms*. Standard French defaults:
   - *Immediate Payment*
   - *30 days end of month*
   - *45 days end of month + 10*
   - *60 days net*
2. Confirm late-payment **indemnités forfaitaires** notice on the invoice template (mandatory in France: €40 minimum recovery cost).
3. **Payment methods**: *Accounting → Configuration → Payment Methods*. Enable SEPA Direct Debit and SEPA Credit Transfer if used.
4. **Customers / Vendors** — import via *Contacts → Favorites → Import* with at minimum:
   - `name`, `street`, `city`, `zip`, `country_id`, `vat`, `email`
   - `property_payment_term_id` (sales payment terms)
   - `property_supplier_payment_term_id` (vendor payment terms)
   - `property_account_receivable_id` / `property_account_payable_id` (override only if customer has special AR/AP)

**Validation:** sample customer record has VAT auto-validated (*VIES check* green for intra-EU), fiscal position auto-detected, payment terms set.

### Step 9 — Opening balances (6h)

This is where most go-lives go wrong. **Do not skip the reconciliation step.**

1. Build the opening trial balance in a spreadsheet:
   - One row per account (including all 4xxx, 1xxx, 2xxx, 5xxx, 6xxx, 7xxx with non-zero balances)
   - Columns: account code, debit, credit
   - Totals must balance (debits = credits)
2. **Open AR** must be imported **per invoice**, not as a lump sum:
   - Create a journal entry: `411xxx (sub-ledger = customer) D / 411-opening C` for each invoice
   - This way, every invoice is open and reconcilable against future payments
3. **Open AP** — same pattern with `401xxx`.
4. **Open bank** balance = closing balance per the bank statement on cutover date.
5. **All other accounts** — single journal entry posting their opening balance, against `890000` (compte d'attente, then offset by the AR/AP detail).
6. Post in a **dedicated** "Opening" journal entry, dated the cutover date.

**Validation:**

- Trial balance in Odoo matches the customer's source trial balance to the cent.
- Open AR total = sum of all unpaid invoices.
- Open AP total = sum of all unpaid bills.
- First bank reconciliation matches the bank statement closing balance.

**Save the reconciliation report.** Customer's expert-comptable should sign off in writing.

### Step 10 — Reports & FEC export (3h)

1. *Accounting → Reporting*:
   - **Balance Sheet**
   - **Profit and Loss** (CR — *Compte de Résultat*)
   - **General Ledger**
   - **Journal**
   - **Partner Ledger** (lettrage)
   - **Aged Receivable** / **Aged Payable**
2. **FEC export** — *Accounting → Reporting → Audit Reports → FEC*. Select fiscal year, format = *FEC*. Output is a `.txt` (UTF-8, tab-separated) per the DGFiP spec.
3. **CA3 (TVA monthly/quarterly)** — *Accounting → Reporting → Tax Report*. Generate, review, then mark as **Closed** for the period.
4. Export to PDF and Excel and walk the customer's accountant through each.

**Validation:**

- FEC opens in `LibreOffice Calc` / Excel without parser errors.
- FEC line count matches Odoo's journal entry count for the period.
- CA3 boxes 08, 09, 10, 11 match the manual sum from the tax grids on entries.

### Step 11 — Lock dates & journal hashing (1h)

For French legal compliance:

1. *Accounting → Settings → Lock Posted Entries with Hash = on* per journal (already done in Step 6).
2. *Accounting → Actions → Lock Dates*:
   - **Tax Lock Date** — once TVA is filed for a period, set this to the last day of that period. Blocks any change that would affect that TVA filing.
   - **Lock Date** — set at the end of each closed fiscal year. Blocks any change to that year's books.
3. **Hash chain** — *Accounting → Reporting → Audit Reports → Inalterability Check*. Run this monthly. Green = compliant.

### Step 12 — Training (2h)

Two short sessions:

- **Accountant / CFO** — 1.5h: chart of accounts, journals, posting workflow, FEC export, lock dates, bank reconciliation.
- **AR / AP user** — 0.5h: how to enter a vendor bill, send a customer invoice, register a payment.

### Step 13 — UAT (3h)

Test cases:

| # | Test case | Expected outcome |
|---|---|---|
| 1 | Customer invoice 20% TVA | Posts to 411 / 707 / 44571; TVA grid box 08 hit |
| 2 | Vendor bill 20% TVA | Posts to 401 / 607 / 44566; TVA grid box 20 hit |
| 3 | Intra-EU B2B customer invoice | Autoliquidation footer; TVA 0%; box 06 |
| 4 | Intra-EU B2B vendor bill | Autoliquidation; net cash effect = 0 |
| 5 | Customer payment via bank | Bank statement line reconciles against invoice |
| 6 | Partial payment | Invoice stays partially open, lettrage correct |
| 7 | Customer refund | Credit note + reconcile |
| 8 | FEC export | File parses, totals match GL |
| 9 | CA3 generation | All boxes accurate, ready for télédéclaration |
| 10 | Year-end close | Lock date set, hash chain intact |

Sign-off in writing.

### Step 14 — Go-live (1h)

Once UAT is signed:

1. Set opening lock date to cutover date.
2. Switch bank journal from "test mode" to live (if any test config).
3. Send the go-live email with login URL and SOPs.
4. Hand the customer the FEC export procedure and the URL of the télédéclaration TVA portal.

---

## Validation checklist

- [ ] French localization installed, COA = PCG 2014
- [ ] Company legal info complete and matching K-bis
- [ ] All TVA rates configured with correct tax grid mappings
- [ ] Autoliquidation taxes (intra-EU, BTP if applicable) configured
- [ ] Fiscal positions auto-detect on EU customers/vendors
- [ ] Journals configured with hashing on
- [ ] Bank account synced or import process documented
- [ ] Opening trial balance imported and matches source to the cent
- [ ] Open AR and AP imported invoice-by-invoice (not lump sum)
- [ ] First bank reconciliation passes
- [ ] FEC export produces a valid file
- [ ] CA3 generation tested against a known period
- [ ] Lock dates set; hash inalterability check green
- [ ] Training delivered
- [ ] All 10 UAT cases signed off
- [ ] Backup retention configured for 10+ years

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| FEC export missing lines | Some entries still in draft | Post all entries; re-run export |
| CA3 boxes empty | Tax grid not mapped on the tax | *Accounting → Configuration → Taxes → [Tax] → Tax Grid* |
| Bank sync stopped working | PSD2 90-day SCA expired | Customer re-authenticates via the bank's portal |
| Customer invoice rejected by FEC | Sequence has a hole (deleted draft) | Audit the journal sequence; if a draft was hard-deleted via dev mode, restore order with a `noupdate` SQL fix (escalate) |
| Hash chain broken | Manual DB edit happened | Identify the entry; ask expert-comptable for guidance; in last resort, lock from a new starting date and document |
| Lock date prevents legitimate entry | Tax/audit lock set too aggressively | Temporarily roll back lock date (admin only); re-set after the entry |
| Intra-EU autoliquidation doesn't appear | Customer's VAT number invalid in VIES | Validate the VAT number; ensure fiscal position is *Intracommunautaire* |

---

## Hours estimate (line by line)

| Step | Description | Hours |
|---|---|---|
| 0 | Kickoff + discovery review | 2 |
| 1 | French localization install | 1 |
| 2 | Company configuration | 1 |
| 3 | Chart of accounts review | 4 |
| 4 | TVA rates + autoliquidation | 4 |
| 5 | Fiscal positions | 2 |
| 6 | Journals | 3 |
| 7 | Bank account sync / import | 4 |
| 8 | Customers, vendors, payment terms | 2 |
| 9 | Opening balances + reconciliation | 6 |
| 10 | Reports + FEC + CA3 | 3 |
| 11 | Lock dates + hash chain | 1 |
| 12 | Training | 2 |
| 13 | UAT | 3 |
| 14 | Go-live | 1 |
| | Buffer / coordination | (rolled into above) |
| | **Total** | **36** |

> Most overrun comes from **Step 9 (opening balances)** when the customer's source data is incomplete. We gate go-live on a clean reconciliation — no exceptions.

---

## What we'll do for you

We are [Roekish](https://roekish.com). We deliver this exact playbook for **€2,520** (36h × our standard rate, FR-based delivery).

What you get:

- A French-compliant Odoo Accounting go-live in a target ~4-week window
- PCG 2014 chart of accounts, TVA fully configured (with FEC + CA3)
- Bank sync set up where supported, or CFONB/OFX import process if not
- Opening balances reconciled to the cent with your expert-comptable
- Hosted on Odoo.sh EU region or your existing instance
- Training for your accounting team
- 10-year backup retention configured

**Reach out: hello@roekish.com** — first call is free, scoping document delivered within a week.

---

## Changelog

- **2026-06-04 — v1**: initial publication. Validated against Odoo 17.0 and 18.0 with `l10n_fr` modules.
- **2026-06-07 — v1.1**: added Odoo 19.0 to validated versions. 19.0 is the current stable and our default for new French engagements — it carries the most current `l10n_fr` / Factur-X e-invoicing foundations for the French reform. Menu paths and fields below are unchanged on 19.0; version-specific differences are flagged inline.
