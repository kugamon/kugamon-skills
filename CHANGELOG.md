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

- `kugamon-full-qtc-submgmt` skill — documented Kugamon Quote to Cash v11.0 features: 6-decimal pricing (`Currency(12, 6)` on quote lines, order lines, Account Pricing, Tier, Asset; APD `kugo2p__PriceScale__c` picklist for per-product precision; Formula (Currency) inheritance into invoice lines) and the 2-way Quote Chat workflow (new objects `kugo2p__ChatParticipant__c` and `kugo2p__ChatMessage__c`, Quote → Participant → Message relationship, Type picklist Internal User / Contact, composite ParticipantKey, read-receipt tracking, HTML Body vs plain BodyPreview).

### Changed

- Bumped `SKILL.md` frontmatter from `0.2.7` to `0.3.0`.

### Notes

- Docs-only change — the behavior change lives in kugo2p v11.0 itself. Superseded in some respects by 0.3.1's trigger-attribution correction.

## [0.2.7] — 2026-05-24

### Fixed

- **Subscription and Asset creation are split by line-object** — earlier releases implied either could come from any line. Corrected rules: Subscriptions are created only from Order Service Lines; Assets only from Order Product Lines; the two never mix. (0.3.1 later revised the trigger attribution: Assets are created by kugo2p, not kuga_sub.)

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.6` to `0.2.7`.

### Notes

- Docs-only change.

## [0.2.6] — 2026-05-24

### Fixed

- Major Product2 / line-level field-semantics correction. Separated three independent Order Release triggers: `Product2.kuga_sub__Track__c` (label "Create Subscription"), `Product2.kuga_sub__Renewable__c` (drives Renewal Opportunity), and `kugo2p__AdditionalProductDetail__c.kugo2p__CreateAsset__c` (drives Asset creation; kugo2p namespace, on APD).

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.5` to `0.2.6`.

### Notes

- Docs-only change.

## [0.2.5] — 2026-05-24

### Added

- New **Setup Types** sub-section in `## Product Setup`. Documents the six possible Product Setup classifications (`{Term} {Unit} Service`, `Renewable {Term} {Unit} Service`, `Product`, `Shippable Product`, `Renewable Product`, `Renewable Shippable Product`) with a driver-fields table, classification table, the rule in plain English, and a downstream-implications table.

### Changed

- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.4` to `0.2.5`.

### Notes

- Docs-only change.

## [0.2.4] — 2026-05-24

### Fixed

- `README.md` — Usage section examples used invented record-name prefixes that contradicted the naming conventions in Appendix E: `Q-0042` (Quote) → `SQ-260519-0020461`; `O-0099` (Order) → `SO-260521-0113570`.
- Bumped `kugamon-full-qtc-submgmt` skill version in `SKILL.md` frontmatter from `0.2.3` to `0.2.4`.

### Notes

- Docs-only change.

## [0.2.3] — 2026-05-24

### Added

- New **Appendix E: Name Field & Sample Data Conventions** in `SKILL.md`. Documents the `Name`-field behavior of every key transactional and master-data Kugamon object in four groups: date-stamped AutoNumber, sequence-only AutoNumber, Apex-overwritten Text(80), and user-supplied Text(80).

### Fixed

- `SKILL.md` frontmatter `version` was still `0.2.1` even though CHANGELOG entries for 0.2.1 and 0.2.2 claimed the frontmatter had been bumped. Brought it up to `0.2.3` to match.

### Notes

- Docs-only change.

## [0.2.2] — 2026-04-23

### Fixed

- Replaced remaining "Kugamon CPQ" references with the correct Kugamon package names (Kugamon Quote to Cash / Kugamon Subscription Management / Kugamon Subscription Billing).
- Bumped `SKILL.md` frontmatter from `0.2.1` to `0.2.2`.

### Notes

- Copy-only changes.

## [0.2.1] — 2026-04-23

### Fixed

- Corrected the umbrella product name in the skill's opening line to "Full lifecycle skill for Kugamon RevOps for Salesforce".
- `README.md` — updated description for consistency.
- Bumped `SKILL.md` frontmatter from `0.2.0` to `0.2.1`.

### Notes

- Copy-only changes.

## [0.2.0] — 2026-04-23

### Changed

- Made the Opportunity pipeline forecasting field rule explicit and upfront: when `HAS_KUGA_SUB = true` use `kuga_sub__Amount__c`, never the standard `Amount` field. Added ⚠️ CRITICAL callout at the top of `SKILL.md` and a 🔑 Pipeline Forecasting Rule at the top of Appendix B.
- Bumped `SKILL.md` frontmatter from `0.1.0` to `0.2.0`.

### Notes

- **Behavior change** in subscription orgs: skill now defaults to `kuga_sub__Amount__c` for pipeline forecasting.

## [0.1.0] — 2026-04-19

### Added

- Initial release of the `kugamon-skills` repository.
- `kugamon-full-qtc-submgmt` skill — full Kugamon Quote-to-Cash and Subscription Management lifecycle support (CPQ / Q2C / SubMgmt / SB modes; auto-detects which Kugamon packages are installed).
- `README.md`, `LICENSE`, `.gitignore`.
