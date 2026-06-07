# Standard Odoo Inventory Setup

> A reproducible playbook to take a French SMB from zero to a working **Odoo Inventory** go-live in **~85 hours** (€5,950 anchor price).

| Item | Value |
|---|---|
| Playbook version | v1 — 2026-06-04 |
| Odoo edition | Community **or** Enterprise (differences flagged inline) |
| Validated versions | Odoo 17.0, 18.0, 19.0 (19.0 = current stable, default for new projects) |
| Region | France / EU |
| Effort anchor | **85 hours** |
| Price anchor | **€5,950** (Roekish standard rate) |
| Audience | Ops lead or in-house IT executing a single-warehouse rollout |

> Price/hours are **anchors based on Roekish devis 2026-003**, not a quote. Your project is scoped during discovery.

---

## Scope (in)

This playbook covers everything to deliver a **single-company, single- or two-warehouse** Odoo Inventory go-live:

1. Inventory app activated and configured
2. Master data: products, product categories, units of measure, vendors
3. Warehouse(s), locations, and stock structure
4. Stock valuation method chosen and applied
5. Routes & rules for receipts, internal transfers, and deliveries
6. Reordering rules (min/max) and/or make-to-order
7. Barcode scanning (optional, +8h)
8. Standard inventory adjustments procedure
9. Reports, dashboards, and KPIs
10. User training (key users + end users)
11. UAT and go-live
12. 5 hours of hypercare after go-live

## Scope (out)

These are **explicitly out** of the 85h anchor. If you need them, we scope separately:

- Multi-company / inter-company transfers
- Manufacturing (MRP, Bills of Materials, work orders)
- Quality module
- Purchase forecasting beyond standard reordering rules
- Custom integrations (WMS, EDI, e-commerce sync, transport portals)
- Data migration from a legacy ERP (see `/migrations/`)
- Custom reports beyond standard Odoo dashboards
- Multi-warehouse beyond 2 sites
- Lot/serial number traceability if not used standard (>4h extra effort)

---

## Prerequisites

Confirm **all** of these **before** you start. Most failed inventory implementations come from gaps here.

### Decisions to make with the customer

- [ ] **Edition**: Community or Enterprise (Enterprise required for barcode app, advanced reports)
- [ ] **Hosting**: Odoo.sh EU region, on-prem FR, or other (see *Hosting & GDPR* below)
- [ ] **Stock valuation method**: Standard Price, Average Cost (AVCO), or FIFO
- [ ] **Costing**: manual or automated (Automated requires Anglo-Saxon accounting in some setups)
- [ ] **UoM**: are you using multiple units of measure (kg/ton, m/m², piece/dozen)? If not, **disable** the feature — fewer bugs.
- [ ] **Lot/Serial tracking**: none, lots, serial — per product? Globally?
- [ ] **Barcode**: yes/no, scanner model, mobile vs desktop
- [ ] **Reordering**: min/max thresholds per product, or make-to-order
- [ ] **Number of warehouses and locations** at go-live

### Data to receive from the customer (before kickoff)

- [ ] Product master list (CSV/XLSX) — at minimum: internal reference (SKU), name, type (storable / consumable / service), UoM, sales price, cost, vendor, barcode (if any)
- [ ] Vendor list (name, email, address, payment terms, default lead time)
- [ ] Customer list (if not already done in Sales)
- [ ] Opening stock per product per location (cutover snapshot)
- [ ] Org chart of warehouse users and roles

### Hosting & GDPR

For French/EU customers we recommend **Odoo.sh EU region**. It satisfies:

- GDPR data residency (servers in EU)
- Daily backups + point-in-time recovery
- Native staging environment for testing
- Automated upgrades

If self-hosting, document:

- Datacenter region (must be EU for GDPR-by-default)
- Backup policy (frequency, retention, restore test cadence)
- Patch / upgrade ownership
- Access control (who can SSH, how is it logged)

---

## Step-by-step setup

### Step 1 — Install Inventory and required apps (1h)

1. **Apps → Update Apps List** (developer mode) if you self-host. Skipped on Odoo.sh.
2. Install **Inventory** (`stock`).
3. Install **Purchase** (`purchase`) if vendors and POs are in scope.
4. (Optional) Install **Barcode** (`stock_barcode`) — Enterprise only.

**Validation:** menu *Inventory* visible in the main app launcher. *Operations / Transfers* opens an empty list.

### Step 2 — Company & warehouse configuration (4h)

1. *Settings → Companies → Update Info*. Confirm legal name, address, VAT number, currency (EUR), country (France).
2. *Inventory → Configuration → Warehouses*. The default warehouse is `WH`. Rename it to a meaningful short code (e.g., `PAR` for Paris).
3. For each additional warehouse: **Create** with code + address. Keep short codes — they prefix every operation reference.
4. *Inventory → Configuration → Settings*:
   - **Storage Locations**: on (required for multi-location)
   - **Multi-Step Routes**: on if you need separate input / quality / stock sub-areas, off otherwise
   - **Units of Measure**: only if used
   - **Lots & Serial Numbers**: only if tracked
   - **Consignment**: off unless vendor-managed stock
   - **Storage Categories**: on if you have put-away constraints (e.g., temperature zones)
5. Save and **reload the page** — the menu structure changes after enabling features.

**Validation:** *Inventory → Configuration → Locations* shows expected hierarchy per warehouse (Stock, Input if multi-step, Output if multi-step, Quality Control if enabled).

### Step 3 — Locations & sublocations (4h)

For each warehouse:

1. *Inventory → Configuration → Locations → Create*.
2. Define location hierarchy. Recommended starting structure for a single warehouse:
   ```
   WH/Stock
     WH/Stock/A    (zone A)
       WH/Stock/A/01   (rack 01)
         WH/Stock/A/01/01   (shelf 01)
   ```
3. Use *Removal Strategy* per parent location:
   - **FIFO**: oldest stock leaves first (default for FIFO valuation)
   - **FEFO**: First Expired, First Out — required if expiry dates tracked
   - **LIFO**: rare, regulated industries
4. Mark *Is a Scrap Location* on the dedicated scrap location.

**Validation:** *Inventory → Reporting → Locations* shows the hierarchy. Create one test product, run an internal transfer between two sublocations, confirm it succeeds.

### Step 4 — Units of measure (1h, if enabled)

1. *Inventory → Configuration → Units of Measure → UoM Categories*. Default categories cover most cases.
2. Add new categories only when you have a UoM family Odoo doesn't ship (uncommon).
3. Per UoM, set **Type**:
   - *Reference* — the unit conversions are based on (e.g., kg in the "Weight" category)
   - *Bigger than reference* — e.g., ton (ratio 1000)
   - *Smaller than reference* — e.g., gram (ratio 0.001)

**Validation:** create a test product with sales UoM = "kg" and purchase UoM = "ton". Place a PO for 1 ton, confirm the receipt stocks 1000 kg.

### Step 5 — Product categories & master data (10h)

1. *Inventory → Configuration → Product Categories*. Plan the tree **before** loading products. Categories control:
   - Inventory valuation account (links to Accounting)
   - Costing method (Standard / AVCO / FIFO)
   - Stock valuation account
   - Default routes (Buy / Manufacture / MTO)
2. Set Costing Method and Inventory Valuation (Manual / Automated) per category. Be **consistent** across the whole catalog unless you have a clear reason not to.
3. Import products. Use *Inventory → Products → Favorites → Import*. Template fields to map:
   - `default_code` — internal SKU
   - `name` — product name
   - `type` — `product` (storable), `consu` (consumable), `service`
   - `categ_id` — product category
   - `uom_id` / `uom_po_id` — sales / purchase UoM
   - `list_price` — sales price
   - `standard_price` — cost
   - `barcode`
   - `seller_ids/partner_id` — default vendor
   - `seller_ids/delay` — vendor lead time in days
   - `tracking` — `none` / `lot` / `serial`

**Gotchas:**

- **Storable** products are tracked in stock. **Consumable** products are not (use for fittings, small parts).
- Don't mix tracking modes in the same category if you can avoid it.
- Internal reference (SKU) must be unique if you enable the constraint.

**Validation:** import a known-good subset of 10 products. Verify each appears in *Inventory → Products* with all fields. Then run the full import.

### Step 6 — Vendors and lead times (3h)

1. *Purchase → Orders → Vendors → Create* or import.
2. On each product, *Purchase tab → Vendors*, set the primary vendor, vendor product code, lead time, and minimum quantity.

**Validation:** for a sample product, run *Inventory → Operations → Replenish*. The proposed RFQ should pre-populate vendor, price, and quantity.

### Step 7 — Routes, rules, and replenishment (8h)

Odoo's routes drive what happens automatically.

1. *Inventory → Configuration → Routes*. Default routes shipped: *Buy*, *Manufacture* (if MRP installed), *Make to Order*, *Replenish on Order*.
2. Configure default routes per product category for hands-off behaviour.
3. **Reordering rules** (min/max):
   - *Inventory → Operations → Replenishment*. Add a rule per (product, location) pair.
   - Set *Min Quantity*, *Max Quantity*, *Multiple Quantity* (must be multiple of pack size).
   - Schedule the auto-replenishment cron: *Settings → Technical → Scheduled Actions → Run Scheduler* (set frequency to daily for most ops).
4. **Make-to-Order** (MTO): enable for products where stock is never held (custom, low-volume). Activate the *Make To Order* route on the product.

**Validation:**

- Create a sales order for a product set to Buy + Reordering rule. Confirm the SO. Verify the PO is created automatically at threshold breach.
- Receive the PO. Verify on-hand quantity updates.

### Step 8 — Stock valuation (4h)

> **Decision is per category.** Changing valuation method after go-live triggers complex value adjustments — get it right now.

| Method | Use when | Caveats |
|---|---|---|
| Standard Price | Cost is reviewed periodically by Finance | Variance posts to a Price Difference account |
| Average Cost (AVCO) | Costs change frequently, simple ledger | Average recalculates on every IN move |
| FIFO | Required for regulated industries, exact lot costing | Heaviest computation; requires landed costs for accuracy |

1. *Inventory → Configuration → Product Categories → [Category] → Costing Method*.
2. For **Automated valuation**:
   - Set *Inventory Valuation = Automated*.
   - Map *Stock Valuation Account*, *Stock Input Account*, *Stock Output Account*. Confirm with Accounting before saving — this drives journal entries.
3. For **Manual valuation**: stock moves don't post; periodic JE done by Accounting.

**Validation:** run a test purchase + receipt. Verify the journal entry generated (Automated) or absence (Manual). Check value in *Inventory → Reporting → Valuation*.

### Step 9 — Barcode setup (8h, optional)

Enterprise only.

1. Install **Barcode** app.
2. *Inventory → Configuration → Settings → Barcode → Barcode Scanner: on*.
3. Print location barcodes: *Inventory → Configuration → Locations → Print → Locations Barcode*.
4. Print product barcodes from *Inventory → Products → Print → Products Barcodes*.
5. Define barcode patterns under *Inventory → Configuration → Barcode Nomenclatures* if you use weight-embedded barcodes (EAN-13 with embedded price/weight common in food retail).
6. Test on the scanner model the customer owns (USB-HID acts like a keyboard; Bluetooth scanners need pairing; ruggedized handhelds with the Odoo mobile app behave differently — test the exact device).

**Validation:** receive a 10-line PO using only the scanner. Confirm zero clicks needed beyond the barcode scans.

### Step 10 — Standard procedures (6h)

Document and train on:

1. **Receiving a PO** — Purchase → Orders → Receipts → Validate
2. **Internal transfer** — Inventory → Operations → Transfers → Create
3. **Delivery to customer** — driven by Sales (out of scope here) or manual
4. **Inventory adjustment** — Inventory → Operations → Physical Inventory → Start Inventory → Apply
5. **Scrap** — Inventory → Operations → Scrap → Create
6. **Returns** — from the source receipt/delivery, click *Return*

Put each procedure in a one-page SOP. We ship a template in `/templates/sop-template.md` (TODO in v2).

### Step 11 — Reports & KPIs (4h)

Enable and bookmark for the customer:

- *Inventory → Reporting → Stock Valuation*
- *Inventory → Reporting → Inventory History*
- *Inventory → Reporting → Forecasted Inventory*
- *Inventory → Reporting → Inventory Aging* (Enterprise)
- *Inventory → Reporting → Locations*

Add a dashboard to the user's home screen with the 3 most-used reports.

### Step 12 — User training (6h)

Split into two sessions:

- **Key user (admin) training** — 3h, covers everything in this playbook, including troubleshooting and inventory adjustment.
- **End user training** — 3h, narrower: receiving, transfers, deliveries, scanner operation, and what to do when something looks wrong.

Record both sessions if the customer is OK with it. Hand the recording over with the project files.

### Step 13 — UAT (8h)

Run the customer through **at least these test cases** in a staging environment:

| # | Test case | Expected outcome |
|---|---|---|
| 1 | Receive a PO | Stock increases; valuation journal posts (if Automated) |
| 2 | Sell to a customer | DO confirms; stock decreases on validate |
| 3 | Internal transfer A → B | Quantities move between locations |
| 4 | Inventory adjustment | Difference posts; valuation reflects new value |
| 5 | Reordering rule trigger | Scheduler creates RFQ on next run |
| 6 | Return from customer | Stock returns to chosen location |
| 7 | Scrap | Stock leaves to scrap location; value posts to scrap account |
| 8 | Backorder | Partial receipt → backorder created → second receipt closes it |
| 9 | Lot/Serial (if used) | Trace from sale back to receipt |
| 10 | Barcode (if used) | Full receipt flow without keyboard |

Sign-off the UAT with the customer **in writing** before scheduling go-live.

### Step 14 — Go-live (4h) + Hypercare (5h)

1. **Snapshot of opening stock** — generate from staging or import via CSV.
2. **Cutover window** — typically end-of-day Friday for weekend cutover.
3. **Smoke tests in prod** — repeat the top 3 UAT cases against real prod data.
4. **Comms** — send the go-live email to all users with the SOP links.
5. **Hypercare** — 5 hours of named support over the first 5 business days. Set up a dedicated channel (Slack / email / phone).

---

## Validation checklist

At the end of the implementation, **every box must be checked**:

- [ ] Inventory app installed, configured, and connected to Accounting (if Automated valuation)
- [ ] All warehouses and locations created and validated
- [ ] All products imported with correct categories, UoM, vendors
- [ ] At least one full purchase → receipt → put-away flow tested
- [ ] At least one full sale → pick → deliver flow tested (if Sales in scope)
- [ ] Reordering rules set up for top SKUs
- [ ] Stock valuation method documented and signed off by Finance
- [ ] Barcode flow tested on the customer's actual device (if applicable)
- [ ] All 10 UAT test cases passed and signed off
- [ ] User training delivered (key users + end users)
- [ ] SOPs handed over (PDF or wiki)
- [ ] Hypercare channel live
- [ ] Backup tested in the hosting environment

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Stock looks wrong after receipt | Multi-step routes enabled but receipt validated at Input only | Validate the *Input → Stock* internal transfer next |
| "No stock available" but quantity > 0 | Reservation locked by a draft transfer | *Inventory → Operations → Transfers*, filter on draft/waiting, unreserve |
| Reordering rule not triggering | Scheduler not running | *Settings → Technical → Scheduled Actions*, run *Run Scheduler* manually; confirm it's enabled |
| Auto valuation: missing journal entry | Stock accounts not configured on the product category | *Inventory → Configuration → Product Categories*, set Stock Valuation / Input / Output accounts |
| Barcode scanner types prefix character | Scanner programmed with a prefix | Scanner config — remove the prefix or set Odoo's nomenclature to skip it |
| Lot number duplicates rejected | Tracking = serial (must be unique globally) | Switch to *lot* if duplication across products is intended |
| FEFO not picking the right lot | Expiry date not set on the lot | Edit lot, set expiry date, re-reserve |

---

## Hours estimate (line by line)

| Step | Description | Hours |
|---|---|---|
| 0 | Kickoff + discovery review | 4 |
| 1 | App install | 1 |
| 2 | Company & warehouse configuration | 4 |
| 3 | Locations & sublocations | 4 |
| 4 | UoM setup | 1 |
| 5 | Product categories & master data import | 10 |
| 6 | Vendors and lead times | 3 |
| 7 | Routes, rules, replenishment | 8 |
| 8 | Stock valuation | 4 |
| 9 | Barcode setup (optional) | 8 |
| 10 | Standard procedures + SOPs | 6 |
| 11 | Reports & KPIs | 4 |
| 12 | User training | 6 |
| 13 | UAT | 8 |
| 14 | Go-live + hypercare | 9 |
| | Buffer / coordination | 5 |
| | **Total** | **85** |

Remove **Step 9** if barcode is not in scope → 77 hours. We keep the anchor at 85h because most French SMBs adopt barcoding within 6 months and it's cheaper to set up once.

---

## What we'll do for you

We are [Roekish](https://roekish.com). We deliver this exact playbook for **€5,950** (85h × our standard rate, FR-based delivery).

What you get:

- A working Odoo Inventory go-live in a target ~6-week window
- Hosted on Odoo.sh EU region (GDPR-compliant) or your existing instance
- Training for key users and end users
- All SOPs and the post-go-live hypercare window
- A clear list of what's not included so there are no surprises

**Reach out: hello@roekish.com** — first call is free, scoping document delivered within a week.

---

## Changelog

- **2026-06-04 — v1**: initial publication. Validated against Odoo 17.0 and 18.0 demo environments.
- **2026-06-07 — v1.1**: added Odoo 19.0 (current stable) to validated versions — now the default for new projects. Menu paths and fields are unchanged on 19.0; version-specific differences are flagged inline.
