# Changelog

All notable changes to this repository are documented here. This project loosely follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [Released]

## [0.4.2] — 2026-11-26

### Added

- `kugamon-full-qtc-submgmt` skill — new **⚠️ Security: do not impersonate another user or contact** callout in `## Workflow 8: 2-way Quote Chat`, directly after the "Posting a message on behalf of a user (programmatic)" block. The DML pattern documented in that block stamps the displayed sender (`kugo2p__ParticipantName__c` and the Participant lookup) from whatever Participant row the acting user can reference, independent of who actually ran the DML. Only `CreatedById` carries the real actor, and most reviewers looking at the chat thread never see it — so the skill now treats cross-user posting as impersonation and refuses it.

  Hard rules the callout adds:
  1. Post only as the acting user's own Participant row (resolve by `kugo2p__ChatParticipant__r.kugo2p__User__c = UserInfo.getUserId()`); never reuse someone else's.
  2. Never post as a `Type = 'Contact'` participant from internal DML — external-contact messages come in through the Online Quote page on the contact's own session.
  3. Never "ghost-write" as a different internal user, even with a plausible reason. Post as the acting user with a note, or ask the real sender to post.
  4. `CreatedById` always reveals the real actor; the callout includes an audit SOQL.
  5. A service account posting on behalf of the system uses a dedicated `Type = 'Internal User'` participant tied to that service-account User, with a `ParticipantName__c` that makes the automation source obvious.
  6. If the user explicitly asks the agent to post as someone else, the agent declines and explains — no workaround negotiation.

### Changed

- Bumped skill version in `SKILL.md` frontmatter, `plugins/kugamon/.claude-plugin/plugin.json`, and `.claude-plugin/marketplace.json` from `0.4.1` to `0.4.2`.

### Notes

- Docs-only change — behavior change is on the agent's side, not in the Kugamon packages.
- No change to the package's own sharing / FLS; this is a guardrail layered into the skill to stop the agent from happily weaponising the DML path it documents.

## [0.4.1] — 2026-11-23

### Changed

- **License: MIT** (was proprietary, all rights reserved). Anyone may use, modify and redistribute the skills in this
  repository, including commercially, provided the license and copyright notice are preserved. The MIT license covers
  this repository only; the Kugamon managed packages keep their own commercial terms.
  - `LICENSE` replaced with the MIT text (© 2026 Kugamon LLC).
  - `license` field `Proprietary` → `MIT` in `plugin.json` and `marketplace.json`.
  - README: MIT badge, License section, repo-layout comment; the Contributing note no longer says the repo is private.
- Version `0.4.0` → `0.4.1` in `SKILL.md`, `plugin.json` and `marketplace.json`, so installed copies pick up the change.

### Notes

- No change to the skill's content. Every commit in the repository is Kugamon's own work, so no third-party notice applies.

## [0.4.0] — 2026-10-29

### Added

Kugamon Subscription Management **v7.1** (and v7.0) behaviour, read from the package source in the kuga_sub packaging org and checked against the kugo2p packaging org:

- **Version detection** — a Tooling API query on `InstalledSubscriberPackage` (filter on namespace client-side; the query cannot) and a table of what each kugo2p / kuga_sub version changes.
- **Contract, Subscription and Renewal actions** — every kuga_sub button with its page URL, and which ones have an API route (a field update) versus UI only.
- **Workflow 4a: Cancel a Subscription** — setting `kuga_sub__Status__c = 'Cancelled'` is the whole action. On v7.1 the trigger also cancels the Order Service Line (`kugo2p__CancelDate__c`, `kugo2p__CancelledQuantity__c`, `kugo2p__CancelReason__c = 'Subscription Cancelled'`), which makes Quote to Cash cancel that line's `Scheduled` invoice lines and service delivery schedules. Also: the renewal-opportunity decrement and its exact match rule, the expired guard, non-renewable lines, Subscriptions without an Order Service Line, child Subscriptions, the once-per-transaction / 200-record limit, and a verification query.
- **Workflow 4b: Cancel a Contract** — allowed statuses, the cascade (renewal opportunity Closed Lost; Subscriptions ending after today cancelled), what it blocks afterwards, and how cancelling an Approved or Released Order cancels its Subscriptions.
- **Workflow 4c: Merge Contracts** (v7.1) — the prerequisite (no Quote or Order on the current Contract's renewal opportunity), the four eligibility rules plus the merge-time check, read-only eligibility SOQL, everything the merge re-points, re-creates, moves and deletes, the Refresh Products step and its sharing caveat. UI only.
- **Workflow 4d: Refresh Products** — guards, which Orders and lines it rebuilds from, the v7.1 per-line exclusion of cancelled Order Service Lines (on Order Release too), and the empty-renewal case.
- Appendix A: `kuga_sub__Subscription__c.kuga_sub__ContractNumber__c` is a re-parentable Master-Detail (v7.1); `kuga_sub__Status__c` values (`Purchased`, `Provisioned`, `Cancelled`, restricted); the Order Service Line cancel fields.
- Four new Common Issues: a cancelled Subscription still billing; a cancelled Subscription returning after Refresh Products; no eligible Contracts to merge; a blocked merge.

### Changed

- 6-decimal pricing: `kuga_sub__Subscription__c.kuga_sub__PurchasePrice__c` is `Currency(12, 6)` from kuga_sub v7.0; the Subscription's MRR, ARR, Net Amount and Total Amount stay 2-decimal.
- Renewal Order automation (v7.0) takes contacts from the Contract on the renewal opportunity's standard Contract field, falling back to the Additional Account Detail.
- Skill version `0.3.1` → `0.4.0` in `SKILL.md`, `plugin.json` and `marketplace.json`.

### Notes

- Docs-only change. A separate fact-check against the source code found one wrong claim (a start date in the renewal match rule), which was corrected before release.

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

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.6` to `0.2.7`.
- Bumped the README skills table row from `0.2.6` to `0.2.7` to match.

### Notes

- Docs-only change. No behavior change in the skill or the underlying package — the correction is purely to the documentation.
- Frontmatter, README, and CHANGELOG all bumped in the same commit.

## [0.2.6] — 2026-05-24

(See repo history on GitHub for the earlier 0.2.6 through 0.1.0 entries — trimmed from this inline payload to fit the release's per-tool payload cap. Nothing below 0.2.7 has changed.)
