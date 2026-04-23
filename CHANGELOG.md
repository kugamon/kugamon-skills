# Changelog

All notable changes to this repository are documented here. This project loosely follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [Unreleased]

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
