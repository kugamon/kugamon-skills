# Changelog

All notable changes to this repository are documented here. This project loosely follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [Released]

## [0.3.1] — 2026-10-27

### Fixed

- **Which package and trigger creates each Order Release record.** 0.2.7 said Subscriptions are created by the Order Service Line trigger and Assets by the Order Product Line trigger. The line-object split was right; the trigger attribution was not. Corrected from source in both packages (kugo2p in the kugo2p packaging org; kuga_sub in the kuga_sub packaging org with kugo2p v11.0 installed):
  - **Assets are created by kugo2p**, not kuga_sub — `SalesOrderHelper.CreateAssets`, called from the kugo2p **Order** trigger (after update) when the Order enters Approved or Released and has no Assets yet. This happens in every deployment mode, with or without kuga_sub.
  - **Contracts, Subscriptions and Renewal Opportunities are created by kuga_sub** (`KugamonHelper.createContract` → `createSubscription` → `createRenewalOpportunity`). The chain runs once per transaction from kuga_sub's Order, Order Product Line or Order Service Line after-update trigger, whichever completes it. kugo2p re-saves every order line on each status change, which is what fires kuga_sub's line triggers.
  - kuga_sub links the Assets in the same transaction (`Asset.kuga_sub__ContractNumber__c`, `Asset.kuga_sub__ParentSubscription__c`, `Subscription.kuga_sub__ParentAsset__c`, where applicable).
- **`kugo2p__CreateAsset__c` on APD is a picklist, not a checkbox.** Values: `Per Line` (one Asset per Order Product Line) and `Per Quantity` (one Asset per unit); blank = no Asset. Every "`CreateAsset__c = true`" reference was replaced.
- **Package Detection** no longer implies that Assets need kuga_sub.
- **Workflow 3: Order Release** is no longer labelled "HAS_KUGA_SUB = true only"; it now says what kugo2p creates in every mode and what kuga_sub adds.

### Added

- Order Release Trigger Map: new **RELEASE CHAIN**, **CONTRACT** and **UN-RELEASE / DELETE** sections; the **ASSET** section now covers `Per Line` / `Per Quantity` and when kugo2p deletes Assets (back to Draft/Sent, line or Order delete, Refresh Assets).
- Common Issues: "Order Release not creating Assets", and the kuga_sub release error for a `Per Quantity` parent product with a `Per Line` child or a child service that creates a Subscription.

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.3.0` to `0.3.1`.
- Bumped `plugins/kugamon/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` from `0.2.7` to `0.3.1` (they were not bumped for 0.3.0), so installed copies pick up the update.

### Notes

- Docs-only change. No behavior change in the skill's guidance on which line object produces what, and none in the packages.
- README unchanged — it has no per-skill version row and does not describe trigger attribution.

## [0.3.0] — 2026-10-09

### Added

- `kugamon-full-qtc-submgmt` skill — documents two new Kugamon Quote to Cash v11.0 feature areas:

  **1. Decimal price precision (6 decimals)**
  - New top-level section `## Decimal Price Precision (kugo2p v11.0+)` in `SKILL.md` listing every stored field widened to `Currency(12, 6)` and the **Formula (Currency)** fields that inherit 6-decimal precision from the stored fields they reference — including `kugo2p__DiscountSalesPrice__c` (label: "Effective Price") on quote lines, order lines, AND invoice lines, plus the full set of invoice-line formulas (`SalesPrice`, `LineAmount`, `NetAmount`, `TotalAmount`, `TaxAmount`, `VATAmount`, `LineDiscountAmount`, `BalanceDueAmount`). Clarifies that invoice-line **stored** currency fields (payments, applied-payment amount) stay at 2 decimals; the 6 decimals reach invoices via formulas, not stored fields. Also spells out what was NOT widened (payments, PricebookEntry, OpportunityLineItem, Opportunity roll-ups).
  - New Appendix A sub-section `### 6-Decimal Price Fields (v11.0+)` with the authoritative field matrix:
    - `kugo2p__SalesQuoteProductLine__c` — `ListPrice__c`, `SalesPrice__c`, `TierPrice__c`
    - `kugo2p__SalesQuoteServiceLine__c` — `ListPrice__c`, `SalesPrice__c`, `NonUpliftSalesPrice__c`, `UpliftPrice__c`, `TierPrice__c`
    - `kugo2p__SalesQuoteOptionalLine__c` — `ListPrice__c`, `SalesPrice__c`
    - `kugo2p__SalesOrderProductLine__c` — `ListPrice__c`, `SalesPrice__c`, `TierPrice__c`
    - `kugo2p__SalesOrderServiceLine__c` — `ListPrice__c`, `SalesPrice__c`, `NonUpliftSalesPrice__c`, `UpliftPrice__c`, `TierPrice__c`
    - `kugo2p__AccountPricing__c` — `Price__c`
    - `kugo2p__Tier__c` — `TierPrice__c`
    - `Asset` — `kugo2p__PurchasePrice__c`
  - Documents the new `kugo2p__PriceScale__c` picklist on `kugo2p__AdditionalProductDetail__c` (observed values `4`, `5`, `6`; blank = org default) and its sibling picklists `kugo2p__QuantityScale__c` and `kugo2p__ServiceTermScale__c`.
  - Updates Object Model Overview APD entry to flag the new scale picklists.
  - SOQL / DML guidance: do not round 6-decimal fields client-side; invoice-line **formula** currency fields render at 6 decimals via Formula (Currency) inheritance from the Order line, while invoice-line **stored** currency fields (AppliedPaymentAmount) stay at 2 decimals; opportunity amount roll-ups continue at their native precision.

  **2. 2-way Quote Chat**
  - New `## Workflow 8: 2-way Quote Chat (kugo2p v11.0+)` section covering the full data model:
    - `kugo2p__ChatParticipant__c` (key prefix `a0B`) — one row per internal User or external Contact authorized to chat on a quote, carries read-receipt state (`kugo2p__LastReadMessage__c`, `kugo2p__DateLastRead__c`).
    - `kugo2p__ChatMessage__c` (key prefix `a0H`) — one row per message, HTML body in `kugo2p__Body__c`, plain-text preview in `kugo2p__BodyPreview__c`.
    - Relationship diagram: Quote → Participants → Messages. The sender's identity + type live on the Participant, not the Message.
    - Internal chat on the Quote record and the external 2-way chat on the Online Quote page use the SAME two objects — only `kugo2p__ChatParticipant__c.kugo2p__Type__c` ('Internal User' vs 'Contact') distinguishes them.
    - Composite key `kugo2p__ParticipantKey__c = {QuoteId}-{UserOrContactId}` prevents duplicate participants.
    - Sample SOQL to read a full thread and to list participants with unread timestamps.
    - Programmatic send: resolve participant → insert message with `Body` (HTML) + `BodyPreview` (plain-text truncation) + denormalized `ParticipantName`.
  - Object Model Overview adds a new "Quote Chat (v11.0+)" sub-group listing both objects.
  - Appendix A adds field-reference tables for `kugo2p__ChatParticipant__c` and `kugo2p__ChatMessage__c`.
  - Appendix E (sample-data naming) adds both chat objects to Group B (AutoNumber, sequence-only): ChatMessage uses a 9-digit zero-padded sequence (e.g. `000000165`); ChatParticipant uses a `CP-` prefix (e.g. `CP-0000070`). DO NOT set `Name` on either.

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.7` to `0.3.0` (minor — new feature coverage, no behavior change to existing guidance).

### Verification

- Tooling-API `FieldDefinition` queries against the `kugamon.dev` org confirmed every 6-decimal stored field and every Formula (Currency) field on every object listed above.
- Live `kugo2p__ChatParticipant__c` and `kugo2p__ChatMessage__c` records in `kugamon.dev` confirmed the Type picklist values (`Internal User`, `Contact`), the composite ParticipantKey format, the HTML body / plain preview split, and the sequence formats (`CP-0000070`, `000000165`).
- Cross-referenced the Kugamon v11.0 "What's New" change log at https://help.kugamon.com/s/article/kugamon-quote-to-cash (sandbox push 10/02/2026, production push 10/16/2026).

### Notes

- Docs-only change in `kugamon-skills` — the behavior change lives in the Kugamon Quote to Cash v11.0 package itself.
- README unchanged — the current README has no per-skill version row to bump.
- SKILL.md 0.3.0 content pushed on the same date in a follow-on commit.

## [0.2.7] — 2026-05-24

### Fixed

- **Subscription and Asset creation are split by line-object** — earlier releases (notably 0.2.5 and 0.2.6) implied either could come from any line. The corrected rules are:
  - **Subscriptions** are created **only from Order Service Lines** (`kugo2p__SalesOrderServiceLine__c`). When `APD.kugo2p__Service__c = true` AND `Product2.kuga_sub__Track__c = true` ("Create Subscription"), the Order Service Line trigger generates a Subscription on Order Release.
  - **Assets** are created **only from Order Product Lines** (`kugo2p__SalesOrderProductLine__c`). When `APD.kugo2p__Service__c = false` AND `APD.kugo2p__CreateAsset__c = true`, the Order Product Line trigger generates an Asset on Order Release.
  - The two never mix: Order Service Lines never generate Assets, and Order Product Lines never generate Subscriptions — regardless of how the underlying flags are set.

### Sections updated

- `## Object Model Overview` — Product2 (4 fields), SalesOrderServiceLine, SalesOrderProductLine blocks now state the line-object split explicitly.
- `## Workflow 3: Order Release` — "What Gets Created" (Subscriptions, Assets blocks) and "Product2 (and APD) Flags for Order Release" table rewritten to make the Order Service Line ↔ Subscription and Order Product Line ↔ Asset pairing the headline.
- `## Apex Class Logic` — "Order Release Trigger Map" diagram restructured into three clearly-labeled sections: SUBSCRIPTION (Order Service Line trigger only), ASSET (Order Product Line trigger only), RENEWAL OPPORTUNITY (any line).
- `## Product Setup` → `### Setup Types` — driver-fields table now specifies the line-object each flag operates on; the "Label vs behavior" callout calls out the line-object split; the "What each type implies downstream" table no longer suggests services can create Assets or products can create Subscriptions.
- `## Appendix A` — "Order Line kuga_sub Fields" and "Product2 kuga_sub Fields" tables now use the Order Service Line / Order Product Line distinction.

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.6` to `0.2.7`.
- Bumped the README skills table row from `0.2.6` to `0.2.7` to match.

### Notes

- Docs-only change. No behavior change in the skill or the underlying package — the correction is purely to the documentation.
- Frontmatter, README, and CHANGELOG all bumped in the same commit.

## [0.2.6] — 2026-05-24

### Fixed

- **`kugamon-full-qtc-submgmt` skill — major Product2 / line-level field-semantics correction.** Multiple sections previously conflated three independent Order Release triggers. The corrected mappings are:
  - `Product2.kuga_sub__Track__c` (label: "Create Subscription") — generates a Subscription on Order Release, **but only for Services** (`APD.kugo2p__Service__c = true`). Products never generate Subscriptions, regardless of any flag.
  - `Product2.kuga_sub__Renewable__c` — triggers **Renewal Opportunity** creation on Order Release (NOT Subscription). Also drives the "Renewable" prefix on the Product Snapshot LWC setup label.
  - `kugo2p__AdditionalProductDetail__c.kugo2p__CreateAsset__c` — drives **Asset** creation on Order Release. Note: `kugo2p` namespace, on APD (not `kuga_sub`, not on Product2).
  - Line-level `kuga_sub__Track__c` (Order Line, Quote Line, OLI) — same "Create Subscription" semantics; propagated from `Product2.kuga_sub__Track__c`.
  - Line-level `kuga_sub__Renew__c` — revenue classification only (recurring MRR/ARR vs one-time NonRecurringRevenue). Does NOT create Subscriptions. See Appendix D.

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.5` to `0.2.6`.
- Bumped the README skills table row from `0.2.5` to `0.2.6` to match.

### Notes

- Docs-only change. No behavior change in the skill or the underlying package — the correction is purely to the documentation.
- Frontmatter, README, and CHANGELOG all bumped in the same commit.

## [0.2.5] — 2026-05-24

### Added

- `kugamon-full-qtc-submgmt` skill — new **Setup Types** sub-section in the existing `## Product Setup` section of `SKILL.md`. Documents the six possible Product Setup classifications that every Kugamon product resolves to (`{Term} {Unit} Service`, `Renewable {Term} {Unit} Service`, `Product`, `Shippable Product`, `Renewable Product`, `Renewable Shippable Product`), with a driver-fields table, a six-row classification table, the rule in plain English, and a downstream-implications table.

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.4` to `0.2.5`.
- Bumped the README skills table row from `0.2.4` to `0.2.5` to match.

### Notes

- Docs-only change. No behavior change.

## [0.2.4] — 2026-05-24

### Fixed

- `README.md` — Usage section examples used invented record-name prefixes that contradicted the naming conventions documented in Appendix E of `SKILL.md` (added in 0.2.3): `Q-0042` (Quote) → `SQ-260519-0020461`; `O-0099` (Order) → `SO-260521-0113570`.
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.3` to `0.2.4`.
- Bumped the README skills table row from `0.2.3` to `0.2.4` to match.

### Notes

- Docs-only change. No behavior change.

## [0.2.3] — 2026-05-24

### Added

- `kugamon-full-qtc-submgmt` skill — new **Appendix E: Name Field & Sample Data Conventions** in `SKILL.md`. Documents the `Name`-field behavior of every key transactional and master-data Kugamon object (four groups: date-stamped AutoNumber, sequence-only AutoNumber, Apex-overwritten Text(80), and user-supplied Text(80)).
- `SKILL.md` — Appendix A Quote "Auto-Managed Fields" table now shows the `SQ-{YYMMDD}-{0000000}` format and cross-references Appendix E.

### Fixed

- `SKILL.md` frontmatter `version` was still `0.2.1` even though CHANGELOG entries for 0.2.1 and 0.2.2 both claimed the frontmatter had been bumped. Brought it up to `0.2.3` to match this release.

### Notes

- Docs-only change. No behavior change.

## [0.2.2] — 2026-04-23

### Fixed

- `kugamon-full-qtc-submgmt` skill — replaced remaining "Kugamon CPQ" references with the correct Kugamon package names (Kugamon Quote to Cash / Kugamon Subscription Management / Kugamon Subscription Billing).
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.1` to `0.2.2`.

### Notes

- Copy-only changes. No behavior change.

## [0.2.1] — 2026-04-23

### Fixed

- `kugamon-full-qtc-submgmt` skill — corrected the umbrella product name in the skill's opening line to "Full lifecycle skill for Kugamon RevOps for Salesforce" (no modifiers).
- `README.md` — updated description for consistency.
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.0` to `0.2.1`.

### Notes

- Copy-only changes. No behavior change.

## [0.2.0] — 2026-04-23

### Changed

- `kugamon-full-qtc-submgmt` skill — made the Opportunity pipeline forecasting field rule explicit and upfront: when `HAS_KUGA_SUB = true` use `kuga_sub__Amount__c`, never the standard `Amount` field. Added ⚠️ CRITICAL callout at the top of `SKILL.md` and a 🔑 Pipeline Forecasting Rule at the top of Appendix B.
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.1.0` to `0.2.0`.

### Notes

- **Behavior change** in subscription orgs: skill now defaults to `kuga_sub__Amount__c` for pipeline forecasting. Non-subscription orgs unaffected.

## [0.1.0] — 2026-04-19

### Added

- Initial private release of the `kugamon-skills` repository.
- `kugamon-full-qtc-submgmt` skill — full Kugamon Quote-to-Cash and Subscription Management lifecycle support: CPQ, Q2C, SubMgmt, and Subscription Billing deployment modes; auto-detects which Kugamon packages are installed; handles opportunities, quotes, orders, order releases, invoices, payments, shipments, contracts, subscriptions, renewal opportunities, and assets.
- `README.md`, `LICENSE`, `.gitignore`.

### Notes

- Status: **in-testing**. Repository is private during the validation phase.
