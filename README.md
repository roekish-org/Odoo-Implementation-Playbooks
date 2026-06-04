# Odoo Implementation Playbooks

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Odoo 17 + 18](https://img.shields.io/badge/Odoo-17%20%7C%2018-714B67)](#)
[![Region: EU / France](https://img.shields.io/badge/Region-EU%20%2F%20France-blue)](#)

> Open-source implementation playbooks for **Odoo Community & Enterprise**, written by [Roekish](https://roekish.com) — a France/EU-based Odoo implementation agency.

## What this repo is

A set of **production-grade playbooks** that take a reader end-to-end through standard Odoo implementations, with:

- Scope (in / out) so a buyer knows exactly what they get
- Step-by-step menu paths, fields, and validation checks
- Hours estimates and price anchors based on real projects
- GDPR / EU hosting notes for French and EU customers

The playbooks are written so that **a new engineer or a curious prospect can follow them end-to-end against a fresh Odoo demo environment** and reach a working configuration.

## Why this repo exists

Most Odoo implementation knowledge lives inside agencies. We're making ours public because:

1. **Buyers deserve to see what they're buying.** A clear scope is the first sign of a serious agency.
2. **Engineers learn faster from honest, scoped playbooks** than from marketing-heavy guides.
3. **Reproducibility is the test.** If our public playbook doesn't reproduce, neither does our paid work.

## Who this is for

- **Operations / finance leads** evaluating Odoo or moving off legacy ERPs (Sage, Cegid, EBP, …).
- **In-house IT teams** owning a standard Odoo rollout.
- **Other agencies and freelancers** — fork, adapt, contribute back.

## Playbooks

| # | Playbook | Hours | Price anchor (€) | Status |
|---|---|---|---|---|
| 1 | [Standard Inventory Setup](./inventory/README.md) | 85 | 5,950 | v1 |
| 2 | [Standard Accounting Setup (FR)](./accounting/README.md) | 36 | 2,520 | v1 |
| 3 | [Sage → Odoo Migration Checklist](./migrations/sage-to-odoo.md) | varies | varies | v1 outline |

Hours and prices reflect typical French-SMB scope. They are **anchors, not quotes** — your project is scoped during discovery.

## Odoo version & region

| Item | Value |
|---|---|
| Odoo Community / Enterprise | Both supported; differences flagged inline |
| Validated versions | Odoo 17.0 and Odoo 18.0 |
| Region | France / EU (GDPR-compliant) |
| Hosting recommendation | Odoo.sh EU region, or self-hosted in an EU datacenter |
| Last reviewed | 2026-06-04 |

## Repo layout

```
inventory/         # Standard Inventory Setup playbook
accounting/        # Standard Accounting Setup (FR) playbook
migrations/        # Migration runbooks (Sage → Odoo, more to follow)
templates/         # Reusable templates (discovery questionnaire, UAT, etc.)
```

## How to use

1. Read the playbook end-to-end **before** you start configuring.
2. Capture the answers to the "Prerequisites" section — most failures happen there.
3. Follow each step in order; do not skip the **Validation checklist** at the end of each section.
4. If a step doesn't match your Odoo version, open an issue with the version, screenshot, and what you saw.

## Want us to do it for you?

We are [Roekish](https://roekish.com) — a France/EU Odoo implementation agency. We turn these playbooks into delivered projects.

- Standard inventory go-live: ~85 hours, €5,950 (anchor)
- Standard accounting go-live (FR): ~36 hours, €2,520 (anchor)
- Sage → Odoo migration: scoped per data volume and ledger complexity

Reach out: **hello@roekish.com**

## Contributing

Issues, corrections, and PRs welcome. Please keep contributions:

- **Honest** — no marketing language inside playbooks
- **Reproducible** — every snippet runs against a stated Odoo version
- **Region-aware** — flag FR/EU-specific behaviour explicitly

## License

[MIT](./LICENSE) — use, fork, adapt freely. Attribution to Roekish appreciated.
