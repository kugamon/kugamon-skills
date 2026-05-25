# Changelog

All notable changes to this repository are documented here. This project loosely follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [Released]

## [0.2.4] — 2026-05-24

### Fixed

- `README.md` — Usage section examples used invented record-name prefixes that contradicted the naming conventions documented in Appendix E of `SKILL.md` (added in 0.2.3):
  - `Q-0042` (Quote) → `SQ-260519-0020461` (correct `SQ-{YYMMDD}-{0000000}` format).
  - `O-0099` (Order) → `SO-260521-0113570` (correct `SO-{YYMMDD}-{0000000}` format).
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.3` to `0.2.4` — done in the same commit as the README and CHANGELOG updates per the lesson learned in 0.2.3 (prior releases claimed frontmatter bumps that never made it into the file).
- Bumped the README skills table row from `0.2.3` to `0.2.4` to match.

### Notes

- Docs-only change. No behavior change.

## [0.2.3] — 2026-05-24

### Added

- `kugamon-full-qtc-submgmt` skill — new **Appendix E: Name Field & Sample Data Conventions** in `SKILL.md`. Documents the `Name`-field behavior of every key transactional and master-data Kugamon object so that sample data is generated with the correct convention. Organized into four groups:
  - **Group A** — AutoNumber with date-stamped prefix: `kugo2p__SalesQuote__c` (`SQ-{YYMMDD}-{0000000}`), `kugo2p__SalesOrder__c` (`SO-{YYMMDD}-{0000000}`), `kugo2p__KugamonInvoice__c` (`INV-{YYMMDD}-{0000000}`). DO NOT set `Name`.
  - **Group B** — AutoNumber, sequence-only (7-digit zero-padded). Covers all line/junction objects (`SalesQuoteProductLine`, `SalesQuoteServiceLine`, `SalesQuoteAdditionalChargeCredit`, `SalesQuoteOptionalLine`, `SalesOrderProductLine`, `SalesOrderServiceLine`, `SalesOrderAdditionalChargeCredit`, `KugamonInvoiceLine`, `KugamonInvoiceAdditionalChargeCredit`, `OrderInvoiceRelationship`, `ShipmentLine`, `AppliedPayment`, `AdditionalProductDetail`, `ProductCost`) plus `Shipment`, and `ConfigurationOption` which uses a `CO-` prefix. DO NOT set `Name`.
  - **Group C** — `Text(80)` but overwritten by Kugamon Apex: `kugo2p__PaymentX__c` (set to `Payment for Order <SO#>` / `Payment for Invoice <INV#>`), `kugo2p__Payment_Method__c` (set to `<Card Brand> (<last 4>)`), `kugo2p__AdditionalAccountDetail__c` (mirrors `Account.Name`). DO NOT set `Name`.
  - **Group D** — `Text(80)`, user-supplied: line groups, invoice schedules, payment profiles, additional charges/credits, product catalogs/categories, tiers, tiered pricing, carriers, warehouses, tax locations, VAT, service delivery schedules, processor connections. DO set a meaningful `Name`.
  - Includes a "Common mistakes to avoid in sample data" checklist (don't invent your own prefix like `Q-…`, `QT-…`, `ORD-…`, `IN-…`, `INV2-…`, or `SQ-0001` without a date; the date portion is the org-local creation date; etc.).
- `SKILL.md` — Appendix A, Quote "Auto-Managed Fields" table: the `Name` row now shows the `SQ-{YYMMDD}-{0000000}` format inline and cross-references Appendix E.
- Background: a sample quote in an unrelated task had been created with the wrong `Name` convention. Appendix E documents the actual rules so future sample-data work follows them.

### Fixed

- `SKILL.md` frontmatter `version` was still `0.2.1` even though CHANGELOG entries for 0.2.1 and 0.2.2 both claimed the frontmatter had been bumped. Brought it up to `0.2.3` to match this release.

### Verification

- Tooling-API `FieldDefinition` metadata for every documented object (confirmed `Auto Number` vs `Text(80)` per object).
- Live sample records from the `kugamon.dev` org (Org ID `00D80000000cH27EAE`) across all four groups.

### Notes

- Docs-only change. No behavior change.

## [0.2.2] — 2026-04-23

### Fixed

- `kugamon-full-qtc-submgmt` skill — replaced remaining "Kugamon CPQ" references with the correct Kugamon package names. "Kugamon CPQ" is not one of the approved package names; the packages are Kugamon Quote to Cash (kugo2p), Kugamon Subscription Management (kuga_sub), and Kugamon Subscription Billing.
  - Object Model Overview heading: "kugo2p Objects (Kugamon CPQ — ~50 custom objects)" → "kugo2p Objects (Kugamon Quote to Cash — ~50 custom objects)".
  - Opportunity Fields > Strongly Recommended Fields (`AccountId`): "Required for Kugamon CPQ to work properly" → "Required for Kugamon Quote to Cash to work properly".
  - Appendix D: Renew Field Guide > Overview: "classification in Kugamon CPQ" → "classification in Kugamon Subscription Management" (`kuga_sub__Renew__c` is a kuga_sub field, so Subscription Management is the correct package reference).
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.1` to `0.2.2`.

### Notes

- Copy-only changes. No behavior change.

## [0.2.1] — 2026-04-23

### Fixed

- `kugamon-full-qtc-submgmt` skill — corrected the umbrella product name in the skill's opening line. Changed "Full lifecycle skill for Kugamon RevOps for Salesforce functionality" to "Full lifecycle skill for Kugamon RevOps for Salesforce" so the umbrella product name is used exactly as specified, with no modifiers.
- `README.md` — updated the `kugamon-full-qtc-submgmt` description from "Manages the full Kugamon lifecycle" to "Manages the full Kugamon RevOps for Salesforce lifecycle" so the umbrella product name is used consistently.
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.0` to `0.2.1`.

### Notes

- Copy-only changes. No behavior change.

## [0.2.0] — 2026-04-23

### Changed

- `kugamon-full-qtc-submgmt` skill — made the Opportunity pipeline forecasting field rule explicit and upfront:
  - Added a new "⚠️ CRITICAL: Opportunity Pipeline Forecasting Field" section near the top of `SKILL.md`, stating that when `HAS_KUGA_SUB = true` the skill MUST use `kuga_sub__Amount__c` for all Opportunity pipeline forecasting and MUST NOT use the standard Salesforce `Amount` field. Includes CORRECT vs WRONG SOQL examples.
  - Reinforced the same rule at the top of Appendix B (Amount Fields Guide) with a "🔑 Pipeline Forecasting Rule (read first)" callout.
  - Updated the Opportunity amount-fields table in Appendix B to flag the standard `Amount` field as "Do NOT use for forecasting when `HAS_KUGA_SUB = true`" and to mark `kuga_sub__Amount__c` as the authoritative forecasting field.
  - Promoted the forecasting rule to item #1 in the Amount Field Best Practices list.
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.1.0` to `0.2.0`.

### Notes

- **Behavior change**: in subscription orgs, the skill now defaults to `kuga_sub__Amount__c` for pipeline forecasting queries, reports, and summaries. Non-subscription orgs (`HAS_KUGA_SUB = false`) are unaffected — the standard `Amount` field is still used as before.

## [0.1.0] — 2026-04-19

### Added

- Initial private release of the `kugamon-skills` repository.
- `kugamon-full-qtc-submgmt` skill — full Kugamon Quote-to-Cash and Subscription Management lifecycle support:
  - Supports CPQ, Q2C, SubMgmt, and Subscription Billing deployment modes.
  - Auto-detects which Kugamon packages (`kugo2p`, `kuga_sub`) are installed in the target org.
  - Handles opportunities, quotes, orders, order releases, invoices, payments, shipments, contracts, subscriptions, renewal opportunities, and assets.
- `README.md` with install, usage, and testing guidance.
- `LICENSE` — proprietary, all rights reserved.
- `.gitignore` for common OS, editor, and secret files.

### Notes

- Status: **in-testing**. Repository is private during the validation phase.
