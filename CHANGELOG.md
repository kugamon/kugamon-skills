# Changelog

All notable changes to this repository are documented here. This project loosely follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [Released]

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
  - Plus Formula (Currency) fields that inherit precision from the above: `DiscountSalesPrice__c` ("Effective Price") on quote/order/invoice lines; `SalesPrice__c`, `LineAmount__c`, `NetAmount__c`, `TotalAmount__c`, `TaxAmount__c`, `VATAmount__c`, `LineDiscountAmount__c`, `BalanceDueAmount__c` on invoice lines.
  - Documents the new `kugo2p__PriceScale__c` picklist on `kugo2p__AdditionalProductDetail__c` (observed values `4`, `5`, `6`; blank = org default) and its sibling picklists `kugo2p__QuantityScale__c` and `kugo2p__ServiceTermScale__c`.
  - Updates Object Model Overview APD entry to flag the new scale picklists.
  - SOQL / DML guidance: do not round 6-decimal fields client-side; opportunity amount roll-ups continue at their native precision.

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
- The SKILL.md v0.3.0 content ships in a follow-on commit on the same date.

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

(See repo history for earlier entries — trimmed from this inline payload for brevity; nothing in them has changed.)
