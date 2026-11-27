---
name: kugamon-full-qtc-submgmt
description: Manage the full Kugamon Quote-to-Cash lifecycle in Salesforce — opportunities, quotes, orders, invoices, payments, shipments, and assets, which uses the kugo2p namespace (Kugamon Quote to Cash). And optionally Managed the full Kugamon Subscription Billing lifecycle  in Salesforce - opportunities, quotes, orders, invoices, payments, shipments, contracts, subscriptions, and assets, which requires the kuga_sub namespace (Kugamon Subscription Management). Detects which packages and versions are installed and adapts accordingly. Covers cancelling subscriptions and contracts, merging contracts, and refreshing renewal opportunity products. Use when users request operations on any Kugamon object.
version: 0.5.0
status: Beta
---

# Kugamon Quote-to-Cash and Subscription Management

Full lifecycle skill for Kugamon RevOps for Salesforce.

| Deployment mode | Packages | What it combines | Flow |
|---|---|---|---|
| **CPQ** | `kugo2p` | CPQ functions (Shipment and Invoice functionality turned off) | Opportunity → Quote → Order → (Order Release) → Asset |
| **Q2C** — Kugamon Quote to Cash | `kugo2p` | CPQ and Billing functions | Opportunity → Quote → Order → (Order Release) → Asset + Shipment → Invoice → Payment |
| **SubMgmt** — Kugamon Subscription Management | `kugo2p` + `kuga_sub` | CPQ and Subscription Management functions | Opportunity → Quote → Order → (Order Release) → Asset + Contract + Subscription + Renewal Opportunity |
| **SB** — Kugamon Subscription Billing | `kugo2p` + `kuga_sub` | All Quote to Cash and Subscription Management functions | Opportunity → Quote → Order → (Order Release) → Asset + Shipment + Contract + Subscription + Renewal Opportunity → Invoice → Payment |

## Contents

**Start here** — Package Detection (and Package versions) · ⚠️ CRITICAL: Opportunity Pipeline Forecasting Field · Quote and Opportunity Sync · Writing Kugamon Records Through the API

**How Kugamon is built** — Object Model Overview · Additional Account Info (AAD) · Order Release Architecture · Product Setup · Pricing and Decimal Prices

**Workflows** — 1: Quote Creation · 2: Order Management · 3: Order Release · 4: Subscription & Contract Management · 4a: Cancel a Subscription · 4b: Cancel a Contract · 4c: Merge Contracts · 4d: Refresh Products · 5: Invoices · 6: Payments · 7: Shipments · 8: 2-way Quote Chat

**Troubleshooting** — Common Issues

**Reference files** — in the `references/` folder next to this file. Read one when the task needs it:

| File | Read it for |
|---|---|
| `references/object-model.md` | Every kugo2p object with its label, key prefix and purpose; the kuga_sub object and the fields kuga_sub adds to other objects |
| `references/fields.md` | Field-by-field reference: Opportunity, Opportunity Product, Quote and its lines, Order and its lines, Subscription, Contract, Asset, Product2, APD, Quote Chat |
| `references/queries.md` | Ready-to-run SOQL for every workflow |
| `references/order-release.md` | The Order Release trigger map, what Release builds, and size limits |
| `references/subscriptions-and-contracts.md` | Full detail for cancelling Subscriptions and Contracts, merging Contracts and refreshing renewal products (Workflows 4a–4d) |
| `references/pricing-and-decimals.md` | How a line's price is found, Account Pricing, Tiered Pricing, kits, configurable products, Price Scale and the full 6-decimal field matrix |
| `references/amounts-and-renew.md` | Amount fields and comparison rules; the `kuga_sub__Renew__c` revenue guide |
| `references/record-types-and-names.md` | Record types; Name field formats and sample-data rules |
| `references/troubleshooting.md` | Symptoms, causes and fixes (Common Issues in full) |

**Keeping it organized.** Keep this order when the skill is extended: put a new section in the group it belongs to and add it to this list. Rules and steps stay in this file; long field tables, query lists and matrices go in the matching reference file. Package versions appear only in the Package versions table.

---

## Package Detection

**First step: always detect which packages are installed.**

Check whether `kuga_sub__Renew__c` exists on `OpportunityLineItem` by describing the object's fields through the connected Salesforce MCP or API.

- `HAS_KUGA_SUB = true` → kuga_sub is installed (subscription management, contracts, subscriptions, renewals).
- `HAS_KUGA_SUB = false` → only kugo2p (no subscription lifecycle).

This flag controls revenue classification, line-item separation, and whether Order Release creates Contracts, Subscriptions and Renewal Opportunities. Assets are created by Kugamon Quote to Cash (kugo2p) whether or not kuga_sub is installed.

### Package versions

Some behaviour depends on the installed version. Read the versions before using Quote Chat, relying on 6-decimal prices, or cancelling or merging. Query the Tooling API (`InstalledSubscriberPackage` can't be filtered on namespace in the query — select all rows and filter on `SubscriberPackage.NamespacePrefix` yourself):

```sql
-- Tooling API
SELECT SubscriberPackage.NamespacePrefix, SubscriberPackage.Name,
       SubscriberPackageVersion.MajorVersion, SubscriberPackageVersion.MinorVersion,
       SubscriberPackageVersion.PatchVersion
FROM InstalledSubscriberPackage
```

| Package version | What it adds or changes |
|---|---|
| `kugo2p` 11.0+ | Price Scale and up to 6 decimals on line prices (Pricing and Decimal Prices). Quote Chat (Workflow 8). New Quote acceptance and rejection email templates. |
| `kuga_sub` 7.0+ | Subscription Purchase Price holds 6 decimals. Renewal orders created by the Renewal Order automation take their contacts from the Contract (Workflow 4). |
| `kuga_sub` 7.1+ | Merge Contract (Workflow 4c). Cancelling a Subscription also cancels its Order Service Line and stops that line's scheduled billing (Workflow 4a). Refresh Products and Order Release leave out cancelled Order Service Lines (Workflow 4d). Subscriptions can be moved to another Contract. |

Below those versions the feature isn't there. In particular, below `kuga_sub` 7.1 there is no Merge Contract, and cancelling a Subscription changes only the Subscription and the renewal opportunity — the Order Service Line and its scheduled invoice lines are left as they were. This table is the only place the skill lists versions; a section below that says "see Package versions" means check it.

---

## ⚠️ CRITICAL: Opportunity Pipeline Forecasting Field

**When `HAS_KUGA_SUB = true`, ALWAYS use `kuga_sub__Amount__c` for Opportunity pipeline forecasting. DO NOT use the standard Salesforce `Amount` field.**

The standard `Amount` field is unreliable in subscription orgs — it can hold MRR, ACV, TCV or another figure, and the meaning varies by org and even by opportunity, including when the Primary Quote updates it. `kuga_sub__Amount__c` is a Roll-Up SUM maintained by Kugamon Subscription Management and is the authoritative pipeline figure.

1. **`HAS_KUGA_SUB = true`** → use `kuga_sub__Amount__c` for all pipeline forecasts, revenue reports, dashboards and opportunity-level aggregates. Never substitute `Amount`.
2. **`HAS_KUGA_SUB = false`** → use the standard `Amount` field (the `kuga_sub__*` fields don't exist).
3. When a user asks about "opportunity amount", "pipeline value" or "forecast" in a subscription org, default to `kuga_sub__Amount__c` and briefly explain why.
4. In SOQL, reports and list views for pipeline in a subscription org, select `kuga_sub__Amount__c` — not `Amount`.

```sql
-- CORRECT (HAS_KUGA_SUB = true):
SELECT Id, Name, kuga_sub__Amount__c FROM Opportunity WHERE IsClosed = false

-- WRONG (HAS_KUGA_SUB = true):
SELECT Id, Name, Amount FROM Opportunity WHERE IsClosed = false
```

`references/amounts-and-renew.md` has every amount field and the quote-to-opportunity comparison rules.

---

## Quote and Opportunity Sync

Kugamon keeps an Opportunity's Products (`OpportunityLineItem`) a copy of its **Primary** Quote's lines. The sync runs **one way, Quote → Opportunity**. A Primary Order syncs the same way. Before changing lines on either side, read the settings:

```sql
SELECT kugo2p__SyncPrimaryQuotetoOpportunityLines__c, kugo2p__SyncPrimaryOrdertoOpportunityLines__c,
       kugo2p__SyncPrimaryQuotetoOpportunityAmount__c, kugo2p__SyncPrimaryOrdertoOpportunityAmount__c,
       kugo2p__UpdateOpportunityAmountBasis__c
FROM kugo2p__KugamonSetting__c LIMIT 1
```

A Quote syncs when `kugo2p__SyncPrimaryQuotetoOpportunityLines__c` lists its status (`Quote Draft`, `Quote Sent`, `Quote Won`) and the Quote is Primary (`kugo2p__IsPrimary__c`) with an Opportunity. For a Primary Order the values are `Order Draft`, `Order Sent`, `Order Approved`, `Order Released`. Blank means sync is off.

1. **Sync on → change the Quote (or Order), never the Opportunity Products.** The sync runs in the background a few moments after the save. It updates the linked Opportunity Products, creates missing ones and **deletes every other Product on the Opportunity**, including ones added by hand. A direct edit to an Opportunity Product is overwritten or deleted by the next sync.
2. **Sync off, or the Quote isn't Primary → keep both sides consistent yourself**, unless the user says not to. Compare Product/Service, Quantity, Sales Price and Start Date (and Term and End Date when `HAS_KUGA_SUB`), and update both sides.
3. **A Price Book or currency mismatch empties the Opportunity first.** When the Opportunity has no Price Book, a different one, or a different currency, the sync deletes all its Products, sets the Opportunity's Price Book (and currency) to the Quote's, and rebuilds.
4. **A new Quote doesn't sync on insert.** It syncs on the next watched change — adding a line, a status change into a synced status, a header Discount Percent change. An empty Primary Quote's first sync therefore deletes all of the Opportunity's Products. Warn the user before changing an empty Primary Quote whose Opportunity has Products.
5. **Quote and Order on one Opportunity: the last sync wins.** The Opportunity's Sync Source says which record wrote the Products.
6. **Sync can't be switched off for one transaction** from outside the package. For a large data load, clear the sync setting, load, restore it, then re-sync with the **Sync Quotes** (or **Sync Orders**) list-view button.

**What each Opportunity Product gets.** `UnitPrice` = the line's Sales Price — for a Service Line when kuga_sub isn't installed and Service Term Behavior is blank or `Included`, Sales Price × Service Term. `Discount` = the Quote's Discount Percent plus the line's Line Discount %. `ServiceDate` = Date Required (Product Line) or Service Start Date (Service Line). Quantity is rounded to 2 decimals. With kuga_sub, also Renew, and on services Service End Date, Service Term, Service Term Behavior and Unit of Term. `UnitPrice` displays 2 decimals but stores the synced price unrounded, so price decimals never cause a mismatch.

**What the sync skips.** Lines whose APD has `kugo2p__ExcludefromOpportunitySync__c`, lines with Quantity 0, and lines with no Price Book Entry are skipped. A line whose Price Book Entry is **inactive** stops the whole sync. Skips, rounding and errors are recorded in one completed Task on the Quote ("Event occurred…" or "Error occurred in Quote to Opportunity Sync process"); each run deletes the previous run's Task. Start every sync question there.

**Opportunity Amount.** With `kugo2p__SyncPrimaryQuotetoOpportunityAmount__c` on and a basis in `kugo2p__UpdateOpportunityAmountBasis__c`, an Opportunity with **no Products** takes the Primary Quote's total as its Amount. With Products, Amount is the Products' total. Pipeline reporting in subscription orgs still uses `kuga_sub__Amount__c`.

---

## Writing Kugamon Records Through the API

| What you do | What Kugamon does |
|---|---|
| Insert a Quote with an Account (and Opportunity) | Fills the defaults from the AAD and Kugamon Settings (Workflow 1). Supply only what you want to override. |
| Insert a Quote or Order for an Account with no AAD | Kugamon creates the AAD from the Kugamon Settings defaults and links it. |
| Insert a Quote or Order for an Account with two or more AADs | Refused. Through the API the message may be about the Price Book instead. |
| Insert Orders | One Order per save. Two or more Orders in one save fail with "The Payment Method cannot be blank." |
| Insert lines | A service goes on a Service Line, a non-service on a Product Line; the Product's APD must be active; the Product or Service can't be changed once saved. A kit header line adds its member lines. |
| Leave a line's Sales Price blank | The line gets the Price Book's List Price. Account Pricing and Tiered Pricing are applied only in the Add Lines and Edit Lines screens. |
| Write a price with more decimals than the Product's Price Scale | Stored as written; Kugamon's screens and documents round it. Round it yourself if stored and shown values must match. |
| Change an Order's `kugo2p__RecordStatus__c` | Runs the same action as the Status Link — checks, Release, Unrelease, Cancel (Workflow 2). |
| Change a Primary Quote's or Order's lines | Opportunity sync runs if the settings and the record's status call for it. |
| Write header totals | Refused or overwritten: totals are formulas and roll-ups, and stored tax and VAT/GST amounts are recalculated by the line triggers. |
| Insert Quote Chat messages | Follow Workflow 8. Post only as yourself; don't update another participant's read pointer. |

Validation rules apply on every route. Never set auto-managed fields (`Name` on auto-numbered objects, `kugo2p__Status__c` formulas, totals).

---

## Object Model Overview

The main records, by stage: Account → AAD (`kugo2p__AdditionalAccountDetail__c`) · Quote (`kugo2p__SalesQuote__c`) with Product, Service, Optional and Charge/Credit lines, and Quote Chat · Order (`kugo2p__SalesOrder__c`) with Product, Service and Charge/Credit lines · Shipments, Service Delivery Schedules and Assets · Invoices (`kugo2p__KugamonInvoice__c`) with lines · Payments (`kugo2p__PaymentX__c`) and Applied Payments · with kuga_sub, Contracts, Subscriptions (`kuga_sub__Subscription__c`) and Renewal Opportunities. Products are set up on the APD (`kugo2p__AdditionalProductDetail__c`); org-wide settings are on the one `kugo2p__KugamonSetting__c` record and the `kugo2p__Application_Settings__c` custom setting. All 51 kugo2p objects with their key prefixes, and every field kuga_sub adds: `references/object-model.md`.

How they connect:

- **Quote and Order lines point to the APD, not to Product2.** `kugo2p__Product__c` (Product Lines) and `kugo2p__Service__c` (Service Lines) are lookups to `kugo2p__AdditionalProductDetail__c`; the APD's `kugo2p__ReferenceProduct__c` is the Product2. Product Lines carry `kugo2p__ProductName__c`, Service Lines `kugo2p__ServiceName__c`.
- **Orders and Invoices are many-to-many** through `kugo2p__OrderInvoiceRelationship__c`. Invoice Lines point to their Invoice with `kugo2p__Invoice__c` and to the Order Line they bill.
- **Applied Payments** are children of the Payment (`kugo2p__Payment__c`), each pointing at an Order or Invoice line.
- **A Quote's chat** is Chat Messages under Chat Participants, filtered on the participant's Quote.
- **Record Ids**: the first three characters name the object (e.g. `a0I` Order, `a0P` Invoice) — see `references/object-model.md`.

---

## Additional Account Info (AAD)

`kugo2p__AdditionalAccountDetail__c` is Kugamon's record for each Account and the source of every Quote, Order and Invoice default: Contacts (Buying, Billing, Shipping), Price Book, Payment Method and Terms, Days Till Payment Due, Delivery Terms, Discount Percent, Warehouse, Carrier, Tax Location, VAT Registration, taxable flags, Invoice Schedule, terms and conditions, and Ignore Parent Account Pricing.

- **Exactly one per Account.** Kugamon's screens refuse new Quotes, Orders, Invoices and Payments when the Account has none or more than one ("The Additional Account Info record is missing…" / "There are duplicate Additional Account Info records…"). Through the API, a Quote or Order for an Account with none creates it; two or more are refused. Check it before creating anything.
- **How an Account gets one.** The AAD is created when a Kugamon-licensed user creates or edits the Account, filled from Kugamon Settings. Accounts created by an unlicensed user or an integration, while Kugamon's triggers were off, or past the first 200 in one transaction may have none — edit and save the Account as a licensed user.
- **Wrong defaults on a new Quote or Order come from the AAD.** Fix the AAD; existing Quotes and Orders don't change. Creating a Quote or Order fills blank AAD Contacts when the Account has exactly one Contact, and the AAD takes Contacts from Quotes and Orders when it has none for that role.
- **Releasing an Order needs the AAD's Payment Method, Payment Terms and Days Till Payment Due**, even when the Order has its own.
- Its `Name` mirrors the Account name (set by Kugamon).

---

## Order Release Architecture

What creates what when an Order is released (the full trigger map and the size limits are in `references/order-release.md`):

- **kugo2p, every deployment mode:** Shipments, Service Delivery Schedules and Invoices on Release (per the Application Settings switches Disable Shipments and Disable Invoices), and **Assets** when the Order enters Approved or Released and has none yet.
- **kuga_sub, when installed:** the Contract, Subscriptions and Renewal Opportunity, and the links from Assets to them. The chain starts when the Order is Released (or Approved, when the kuga_sub setting starts subscription management on Approve/Release).
- **The line-object split always holds:** Subscriptions come only from Order Service Lines, Assets only from Order Product Lines, Renewal Opportunities from any renewable line.
- **Un-release or delete:** kugo2p removes what Release built and deletes the Assets; kuga_sub deletes the Order's Subscriptions and the Contract created for it, and deletes (or closes as Lost) the Renewal Opportunity.

**Separate concept — revenue classification, not Subscription creation:** `kuga_sub__Renew__c` on Opportunity Products, Quote Lines and Order Lines classifies revenue as recurring (MRR/ARR) or one-time. It does NOT create Subscriptions. See `references/amounts-and-renew.md`.

---

## Product Setup

### Additional Product Info (`kugo2p__AdditionalProductDetail__c`, APD)

Kugamon's record for each Product2, created automatically with the Product (`kugo2p__ReferenceProduct__c`, which can't be changed). Key fields:

- `kugo2p__Service__c` — **CRITICAL**: service or product. Decides which line object a line lands on, and so whether Subscription or Asset creation can fire. When `HAS_KUGA_SUB = false` it is also the only line-separation rule.
- `kugo2p__Active__c` — lines can be added only for active APDs.
- `kugo2p__CreateAsset__c` — `Per Line` / `Per Quantity` / blank; drives Asset creation (Order Product Lines only).
- `kugo2p__DisableShipments__c`, `kugo2p__DefaultServiceTerm__c`, `kugo2p__UnitofTerm__c` — the Setup type (below).
- `kugo2p__PriceScale__c` — decimal places (Pricing and Decimal Prices); `kugo2p__PricingMethod__c` (List or Cost); `kugo2p__ExcludefromOpportunitySync__c`.
- Tax flags are `kugo2p__TaxableSales__c`, `kugo2p__TaxableUsage__c`, `kugo2p__TaxableExcise__c`, `kugo2p__TaxableVAT__c`; kits and configuration use `kugo2p__KitBundle__c` and `kugo2p__ConfigurationMethod__c`. Every APD field: `references/fields.md`.

### Setup Types

Every product resolves to one of six Setup types:

| # | Setup type | `Service__c` | `DisableShipments__c` | `Renewable__c` | kuga_sub installed |
|---|---|---|---|---|---|
| 1 | `{Term} {Unit} Service` | `true` | n/a | `false` or n/a | optional |
| 2 | `Renewable {Term} {Unit} Service` | `true` | n/a | `true` | required |
| 3 | `Product` | `false` | `true` | `false` or n/a | optional |
| 4 | `Shippable Product` | `false` | `false` / blank | `false` or n/a | optional |
| 5 | `Renewable Product` | `false` | `true` | `true` | required |
| 6 | `Renewable Shippable Product` | `false` | `false` / blank | `true` | required |

**Rule**: if `Product2.kuga_sub__Renewable__c = true`, the label starts with "Renewable"; `APD.Service__c = true` ends it with `{Term} {Unit} Service` (`DefaultServiceTerm__c`, or 1 when blank, and `UnitofTerm__c`); otherwise it's `Product` (DisableShipments = true) or `Shippable Product`.

**Label vs behaviour**: the label reflects only Service, term, DisableShipments and Renewable. Subscription and Asset creation are governed by `Product2.kuga_sub__Track__c` and APD `CreateAsset__c`, split by line object.

| Setup type | On Order Release creates |
|---|---|
| `{Term} {Unit} Service` | Order Service Line + Subscription if `Track__c = true`. (No Asset — services never create Assets.) |
| `Renewable {Term} {Unit} Service` | Order Service Line + **Renewal Opportunity**. + Subscription if `Track__c = true`. (No Asset.) |
| `Product` | Order Product Line + Asset if APD `CreateAsset__c` is set. (No Subscription — products never create Subscriptions.) |
| `Shippable Product` | Order Product Line + Shipment. + Asset if APD `CreateAsset__c` is set. (No Subscription.) |
| `Renewable Product` | Order Product Line + **Renewal Opportunity**. + Asset if APD `CreateAsset__c` is set. (No Subscription.) |
| `Renewable Shippable Product` | Order Product Line + Shipment + **Renewal Opportunity**. + Asset if APD `CreateAsset__c` is set. (No Subscription.) |

---

## Pricing and Decimal Prices

**How a line's starting price is found** (most specific wins, in the Add Lines and Edit Lines screens): included $0 (configuration option or kit) → cost price (APD Pricing Method Cost) → purchase price from an existing Asset or Subscription → Account Pricing (the Account or its parent) → discounted configuration option → Tier Price → List Price. The line's Pricing Source shows which applied. **Through the API, Flow or Import Lines, a blank Sales Price gets the List Price only.** Every Product needs a Price Book Entry in the Quote's Price Book and currency.

**Price Scale** (see Package versions). The APD's `kugo2p__PriceScale__c` (2 to 6; blank = 2) sets how many decimals a Product's prices round to and display at; Products on one Quote can differ. These unit-price fields store up to 6 decimals (`Currency(12, 6)`): List, Sales, Tier, Non-Uplift and Uplift Price on Quote and Order lines, Account Pricing Price, Tier Price, Asset Purchase Price, and (see Package versions) Subscription Purchase Price.

- **Amounts and totals stay at 2 decimals** — line amounts, header totals, invoice amounts, payments, Charges/Credits, APD Standard Price, `PricebookEntry.UnitPrice`, and opportunity roll-ups.
- **Effective Price** (`kugo2p__DiscountSalesPrice__c`, a formula) is the Sales Price less the header and line discounts, rounded to the Product's Price Scale on Quote and Order lines. Invoice Lines carry their own Effective Price formula, and their `kugo2p__SalesPrice__c` is a formula that reads the Order line's price — so line prices reach invoices through formulas, while invoice amounts stay at 2 decimals.
- Kugamon's screens, configurators, PDFs and online pages show each line at its Product's scale. Standard record pages, list views and reports show these fields at 6 decimals for every Product.
- Read 6-decimal fields without rounding them unless the user asks. When writing, round to the Product's Price Scale yourself if the stored value must match what customers see.

Account Pricing, Tiered Pricing, kits, configurable products and the full field matrix: `references/pricing-and-decimals.md`.

---

## Workflow 1: Quote Creation

**Step 0 — Opportunity and Opportunity Products (if needed).** Required: `Name`, `StageName`, `CloseDate`; set `AccountId` and an active `Pricebook2Id`. Create Opportunity Products with `PricebookEntryId` (not `Product2Id`). When `HAS_KUGA_SUB = true`, always set `kuga_sub__Renew__c` (`true` = recurring → MRR/ARR; `false` = one-time → NonRecurringRevenue) and, for recurring lines, `kuga_sub__ServiceTerm__c` and `kuga_sub__UnitofTerm__c`. When `HAS_KUGA_SUB = false`, the APD's `kugo2p__Service__c` decides Service vs Product lines.

Whether a new Quote copies the Opportunity Products: with kuga_sub, always. Without kuga_sub, **not** when `kugo2p__SyncPrimaryQuotetoOpportunityLines__c` has any value — the Quote starts empty, and its first sync then deletes the Opportunity's Products (Quote and Opportunity Sync, rule 4).

**Step 1 — Checks before creating** (the Quote is refused otherwise):

- the Kugamon Settings record exists;
- the Account has **exactly one AAD** (through the API a missing one is created; two or more are refused);
- from an Opportunity: every Opportunity Product has a Price Book Entry, and the Opportunity's Price Book is active (if it has none, the AAD's Price Book is used and must be active);
- Contacts: the Quote takes Contact Buying, Billing and Shipping from the AAD (or the Opportunity's Primary Contact Role when Kugamon Settings is set up for it). Make sure the AAD has them or supply them. Bill To comes from Contact Billing's Mailing address, else the Account's Billing address; Ship To from Contact Shipping's Other address, else its Mailing address, else the Account's Shipping address.

**Step 2 — Create the Quote.** Insert `kugo2p__SalesQuote__c` with `kugo2p__Account__c` and `kugo2p__Opportunity__c`, plus anything to override: `kugo2p__QuoteName__c` (defaults to the Opportunity name), `kugo2p__Pricebook2Id__c` (defaults to the Opportunity's, else the AAD's, else the Kugamon Settings default), `kugo2p__ContactBuying__c` / `kugo2p__ContactBilling__c` / `kugo2p__ContactShipping__c`, `kugo2p__IsPrimary__c`, `kugo2p__DateOfferValidThrough__c`, and `RecordTypeId` only when the org has Quote record types (`references/record-types-and-names.md`). Kugamon fills everything else from the AAD and Kugamon Settings: terms, discount, warehouse, carrier, tax flags, addresses, currency, Record Status `Draft`, and the Online Approval Key.

- **Expiration Date** defaults to today + Kugamon Settings `kugo2p__SalesQuoteValidOfferInDays__c`; if that setting is blank, it is today.
- **Primary**: a Quote created on an Opportunity with no Primary Quote becomes Primary. Marking another Primary unchecks the old one; two Primaries in one save are refused.
- **Fixed once created**: the Account ("Account Cannot Be Changed"); the currency once there are lines. Change the Price Book with the **Choose Price Book** button — the Quote stores the Price Book name, which must match an existing Price Book.
- **Never set**: `Name` (`SQ-{YYMMDD}-{0000000}`), `kugo2p__Status__c` (formula), totals.

**Lines.** Insert `kugo2p__SalesQuoteProductLine__c` (`kugo2p__Product__c` = the APD Id of a non-service) or `kugo2p__SalesQuoteServiceLine__c` (`kugo2p__Service__c` = the APD Id of a service), with `kugo2p__SalesQuote__c` and `kugo2p__Quantity__c`. Set `kugo2p__SalesPrice__c` if the price shouldn't be the List Price (see Pricing). The Product can't be changed after save — delete the line and add a new one.

**Step 3 — Verify.** Re-query the Quote and both line objects (`references/queries.md`, Quotes).

**Step 4 — Status.** `kugo2p__RecordStatus__c` (text) holds the value and is the field to report on; `kugo2p__Status__c` is a formula that shows it with action links.

| Record Status | Links |
|---|---|
| `Draft` | Submit · Won · Lost · Cancel |
| `Sent` | Won · Lost · Cancel |
| `Won` | Generate Order |
| `Lost`, `Cancelled` | none |

- **Won doesn't create an Order** — Generate Order does (Workflow 2).
- **Sent by email**: a completed Task logged on the Quote whose subject contains the Quote number moves a Draft Quote to Sent. Emailing from Email PDF or Email Link does this.
- Date Sent is stamped the first time the Quote becomes Sent; Date Closed is stamped when a Status Link closes it.
- Writing Record Status directly corrects the value but skips Kugamon's side effects (no Date Closed, no Order).

**Step 5 — Amounts.** When `HAS_KUGA_SUB = true`, compare `kugo2p__TotalAmount__c` with `kuga_sub__AnnualContractValueInitial__c` or `kuga_sub__TotalContractValue__c`, not `Amount`; when false, with `Amount`. Show all amount fields in the summary (`references/amounts-and-renew.md`).

---

## Workflow 2: Order Management

**Creating an Order.**

- **From a Quote — Generate Order** (the link in a Won Quote's Status, or the button). UI action: `/apex/kugo2p__Kontroller?action=createSalesOrder&qId={Quote Id}`. A Draft or Sent Quote is marked Won first. Refused when the Quote is Restricted, Lost or Cancelled, the Quote's Price Book no longer exists by name or is inactive, a line is on the wrong line type, or the Account doesn't have exactly one AAD. A Restricted Quote or a bad Price Book leaves a Draft or Sent Quote **Won with no Order**; fix the cause and click Generate Order again. A Quote set to Per Line Group creates one Order per line group.
- **From an Opportunity, Account or Contact** — Generate Order / New Order. From an Opportunity the lines come from the Opportunity Products (with kuga_sub always; without it, not when `kugo2p__SyncPrimaryOrdertoOpportunityLines__c` has a value).
- **Through the API** — insert one `kugo2p__SalesOrder__c` per save with `kugo2p__Account__c` (and `kugo2p__Opportunity__c` / `kugo2p__SalesQuote__c`); it is filled from the AAD (Payment Method and Terms, Delivery Terms, Discount Percent, Tax Location, VAT Registration, Warehouse, terms). The Order date is `kugo2p__OrderDate__c` and the name `kugo2p__OrderName__c`; `Name` is `SO-{YYMMDD}-{0000000}`.

Lines split into `kugo2p__SalesOrderProductLine__c` and `kugo2p__SalesOrderServiceLine__c` by the APD Service flag, as on Quotes.

**Contacts and addresses.** Setting Contact Billing copies its Mailing address to Bill To; Contact Shipping copies its Other address to Ship To. Each needs Street and City, and Contact Buying needs a Mailing address. A Released Order must have all three Contacts. `kugo2p__UpdateAdditionalAccountInfoContacts__c` (multi-select) copies the chosen Contacts to the AAD on Release.

**Statuses.** `kugo2p__RecordStatus__c` holds the value; `kugo2p__Status__c` is the formula with links.

| Record Status | Means | Links |
|---|---|---|
| `Draft`, `Sent` | Edits allowed, no payments | Submit · Approve · Release · Cancel |
| `Approved` | Edits allowed, accepts payments, creates Assets | Release · Cancel |
| `Released` | No edits, accepts payments, builds Shipments and Invoices (Workflow 3) | Unrelease · Cancel |
| `Cancelled` | No payments; Account Balance not updated | none |

Date Sent, Date Approved (first move to Approved or Released), Date Released and Date Cancelled are stamped once and not cleared. An Order also becomes Sent when emailed with its `SO-` number in the subject, and Approved when the customer accepts it online.

**Checks before a status change** (Status Links, Release Order and direct Record Status writes all run them; a failed check changes nothing):

- every line still has its Product, on the right line type;
- for anything but cancelling: no associated Invoice is Posted and no Payment is applied to an associated Invoice; Billing, Shipping and Buying Contacts and a Warehouse on the Order; a Warehouse and Shipping Contact on every Product Line; every Product's Price Book Entry active;
- Unrelease: no Shipment is Shipped or Delivered;
- Release: not already Released, not Restricted, and the AAD has Payment Method, Payment Terms and Days Till Payment Due;
- leaving Approved or Released other than by Release, Unrelease or Approve (in practice, cancelling): no active Recurring Charge;
- always: exactly one AAD.

**Changing Record Status through the API** runs the same action as the link: releasing by API builds Shipments, schedules and Invoices; Released → Approved is Unrelease; Released → Cancelled is Cancel Released; Released → Draft or Sent removes the Release work; moving to Draft or Sent deletes the Assets. A failed check refuses the save with its message. **A direct write of `Cancelled` on a Draft, Sent or Approved Order cancels only the header** — its lines keep their status and get no Cancel Date; use the Cancel link to cancel the lines. Values outside Draft, Sent, Approved, Released and Cancelled aren't refused but trigger nothing.

**Primary Order.** One per Opportunity and one per Quote. It syncs to the Opportunity like a Primary Quote (Quote and Opportunity Sync).

---

## Workflow 3: Order Release

On an Order's first move into Released, Kugamon builds everything downstream **in one transaction** — an error anywhere undoes the whole Release. Only the Account Balance is recalculated afterwards in the background.

| Application Settings Disable Invoices | Disable Shipments | Release builds |
|---|---|---|
| off | off | Shipments, Service Delivery Schedules and Invoices |
| off | on | Service Delivery Schedules and Invoices |
| on | off | Shipments only |
| on | on | Nothing |

- Application Settings is a protected custom setting: in a customer's org it can't be queried. Infer the switches from what Release built, or ask the admin.
- Invoices aren't built again when the Order already has an Order/Invoice link, so re-releasing doesn't duplicate them; Unrelease removes the link.
- **Assets** are created by kugo2p for Order Product Lines whose APD `kugo2p__CreateAsset__c` is `Per Line` (one per line) or `Per Quantity` (one per unit), when the Order enters Approved or Released and has none yet. **Order Service Lines never generate Assets.** Refresh Assets (Approved or Released Orders) deletes and recreates them — edits on the old Assets, such as serial numbers, are lost.
- **Large Orders**: treat about 500 lines as the safe ceiling for one Release; much larger Orders fail with "Apex CPU time limit exceeded" and roll back. Split the Order.
- `references/order-release.md` has the trigger map.

### When kuga_sub is installed

Order Release checkboxes on `kugo2p__SalesOrder__c`:

| Field | What it creates |
|---|---|
| `kuga_sub__GenerateContract__c` | Contract |
| `kuga_sub__GenerateAsset__c` | Assets (only from Order Product Lines with APD `CreateAsset__c` set; the Assets themselves are created by kugo2p) |
| `kuga_sub__GenerateSubscription__c` | Subscriptions (only from Order Service Lines with `kuga_sub__Track__c = true`) |
| `kuga_sub__GenerateRenewalOpportunity__c` | Renewal Opportunity (from any line whose product has `kuga_sub__Renewable__c = true`) |

- **Contract** — linked to the Order via `kuga_sub__ContractNumber__c` and to its Order Service Lines via `kugo2p__Contract__c`; contacts copied per `kuga_sub__UpdateContractContacts__c`; Subscriptions roll up ARR, MRR, counts and dates; `kuga_sub__Effective__c` flags currently active Contracts.
- **Assets** — kuga_sub stamps `kuga_sub__ContractNumber__c` on the Assets kugo2p creates, and `kuga_sub__ParentSubscription__c` where the Asset sits under a Subscription.
- **Subscriptions** — one per Order Service Line with `kuga_sub__Track__c = true` (the line's Track usually comes from `Product2.kuga_sub__Track__c`, label "Create Subscription"). **Order Product Lines never generate Subscriptions.** `kuga_sub__Status__c` is set to `Purchased`.
- **Renewal Opportunity** — when any line's product has `Product2.kuga_sub__Renewable__c = true`; sets `kuga_sub__ParentContract__c` and `kuga_sub__ParentOrder__c`; close date from the Contract end date; `kuga_sub__RenewalPriceUpliftPercent__c` carries forward.

| Product flag | Object | Effect on Order Release |
|---|---|---|
| `kuga_sub__Track__c` ("Create Subscription") | `Product2` | On an **Order Service Line**: a Subscription. Ignored for products. |
| `kugo2p__CreateAsset__c` (`Per Line` / `Per Quantity`) | APD | On an **Order Product Line**: Assets, created by kugo2p. Blank = none. Ignored for services. |
| `kuga_sub__Renewable__c` | `Product2` | On any line: a Renewal Opportunity. Also drives the "Renewable" Setup label. |
| `kuga_sub__RenewalProduct__c` | `Product2` | Substitute product used on renewals. |
| `kuga_sub__UpliftRenewalPrice__c` | `Product2` | Apply the renewal price uplift percentage. |

Queries for what a Release created: `references/queries.md`, Order Release.

---

## Workflow 4: Subscription & Contract Management

Query active Contracts and their Subscriptions with the queries in `references/queries.md` (Contracts and Subscriptions). Key fields: `Contract.kuga_sub__Effective__c`, ARR/MRR roll-ups, `kuga_sub__RenewalOpportunity__c` (the Current Renewal Opportunity); `kuga_sub__Subscription__c.kuga_sub__Status__c` (`Purchased` / `Provisioned` / `Cancelled`), `kuga_sub__Active__c`.

**Renewal automation.** `ContractRenewalNoticeBatcher` sends renewal emails N days before the Contract ends; `RenewalOrderBatcher` creates renewal orders N days before the end. The renewal order takes its Buying, Billing and Shipping contacts from the Contract in the renewal opportunity's standard Contract field (`kuga_sub__ContactBuying__c`, `kuga_sub__ContactBilling__c`, `kuga_sub__ContactShipping__c`), falling back to the Account's AAD for any the Contract leaves blank (see Package versions). Renewal opportunities carry that standard Contract field only when Kugamon Settings has Extend Contract on Renewal on; the Parent Contract field is always set.

### Contract, Subscription and Renewal actions (buttons)

These actions are Visualforce pages behind detail-page buttons. Their logic is not callable from subscriber Apex or the API. Where a field change drives the action, the table says so — that is the API route.

| Button | Object | Page | API equivalent |
|---|---|---|---|
| Cancel Subscription | Subscription | `/apex/kuga_sub__kontroller?action=cancelSubscription&id={Subscription Id}` | Update `kuga_sub__Status__c` to `Cancelled` (Workflow 4a) |
| Cancel Contract | Contract | `/apex/kuga_sub__kontroller?action=cancelContract&id={Contract Id}` | Update `Status` to `Cancelled`, and `EndDate` to today if it is in the future (Workflow 4b) |
| Merge Contract | Contract | `/apex/kuga_sub__MergeContract?id={Contract Id}` | None — UI only (Workflow 4c; see Package versions) |
| Refresh Products | Opportunity, Opportunity Products list | `/apex/kuga_sub__Kontroller?action=refreshProducts&Id={Opportunity Id}` | None — UI only (Workflow 4d) |
| Manage Contract / Manage Contracts | Contract / Account | `/apex/kuga_sub__ManageContract?id={Contract Id}`, `/apex/kuga_sub__ManageContractAccount?id={Account Id}` | None — UI only |
| New Expansion Opportunity / New Expansion Order | Contract | `/apex/kuga_sub__kontroller?action=newExpansionOpportunity&id={Contract Id}` / `action=newExpansionOrder` | None — UI only. Both refuse a Cancelled Contract |

Give the user the page link with the record Id filled in when an action is UI only. Never try to reproduce a UI-only action with your own DML.

---

## Workflow 4a: Cancel a Subscription

**Read `references/subscriptions-and-contracts.md` (Workflow 4a) before cancelling.**

- **How.** Set `kuga_sub__Subscription__c.kuga_sub__Status__c = 'Cancelled'` — the button does only that, and the trigger does the rest, so an API update has the same effect. Refused when already Cancelled or `kuga_sub__EndDate__c` is before today ("Expired Subscriptions cannot be Cancelled.").
- **What follows**, for a renewable Order Service Line (with the kuga_sub version in Package versions): the line gets Cancel Date, Cancelled Quantity and Cancel Reason `Subscription Cancelled`; its `Scheduled` invoice lines and delivery schedules become `Cancelled`; the Current Renewal Opportunity's matching product is reduced (silently skipped if it doesn't match exactly); Contract ARR and MRR drop after the next Subscription job. Invoices already generated never change — credit or refund those separately.
- **Tell the user** when the line isn't renewable (only the Status changes and billing continues), when a child Subscription also needs cancelling, or when the Subscription has no Order Service Line (the cancel fails — report it, don't retry).
- **Bulk:** at most 200 Subscriptions per update and one cancelling update per transaction; then run the check in `references/queries.md` (Cancellations).

## Workflow 4b: Cancel a Contract

- **How.** Set `Contract.Status = 'Cancelled'` and move `EndDate` back to today if it is in the future (the button does the same after a confirmation). Refused unless Status is blank, `Draft` or `Activated`.
- **What follows:** the Current Renewal Opportunity goes Closed Lost; every Subscription ending **after** today is cancelled (Workflow 4a each) — those ending today or with blank dates are left; the Contract can no longer be used for expansions or Refresh Products.
- **Cancelling an Approved or Released Order** cancels its Subscriptions too; an expired Subscription on the Order blocks the cancel.
- Details and the Expansion Order exception: `references/subscriptions-and-contracts.md` (Workflow 4b).

## Workflow 4c: Merge Contracts

UI only, and needs the kuga_sub version in Package versions. Folds the **current** Contract into a **primary** Contract for the same Account, then **deletes the current Contract** and its Current Renewal Opportunity. Get the user's explicit confirmation and name the Contract that will be deleted.

- **Where:** `/apex/kuga_sub__MergeContract?id={Contract Id}` (an admin may have to add the button). Never imitate the merge with DML.
- **Blocked** when the current Contract's Current Renewal Opportunity has any Quote or Order — even ones the user can't see.
- **Eligible primaries:** Effective, same Price Book and Account, and the current renewal's Date Required inside the primary's renewal service window; the primary's renewal must be a `Renewal` record type pointing back to it.
- Check eligibility read-only first (`references/queries.md`, Merge Contract). What the merge moves, deletes and refreshes: `references/subscriptions-and-contracts.md` (Workflow 4c).

## Workflow 4d: Refresh Products on a Renewal Opportunity

UI only (Refresh Products button on the Opportunity). Deletes every product on the renewal opportunity and rebuilds them from the Parent Contract's released Orders' renewable lines — **products added by hand are lost**, so say so first. Cancelled Order Service Lines are left out (see Package versions); if every service line on an Order is cancelled, its product lines are dropped too. Refused unless the opportunity is the Parent Contract's Current Renewal Opportunity, has record type `Renewal`, and the Contract isn't Cancelled. Details: `references/subscriptions-and-contracts.md` (Workflow 4d).

---

## Workflow 5: Invoices

**How Invoices are created.**

- **Order Release** (Workflow 3) builds Invoices dated by the Order's Invoice Schedule: one Invoice Line per Shipment Line (products) and per Service Delivery Schedule row (services), each linked to its Order Line; an Order/Invoice link per pair; Payments already taken on the Order are applied to the new Invoices. Defaults: Record Status `Unsent`; Payment Terms from the AAD; Payment Method from the Order, else the AAD; Due Date = Invoice Date + the AAD's Days Till Payment Due (blank = 0); Contacts, PO and addresses from the Order.
- **New Invoice** (the Invoices tab, or the Account's Invoices related list) creates an empty `Unsent` Invoice for the Account. Charges/Credits can be added to it, but not Product or Service Lines. Its Due Date uses the **Kugamon Settings** Days Till Payment Due, not the AAD's. The Account must have exactly one AAD.

**Invoice Schedule.** The Order's `kugo2p__InvoiceSchedule__c` (else the AAD's) sets the period: Daily, Weekly, Monthly, Quarterly, Semi-Annually, Annually, or blank = one-time. Services are billed one delivery period (their Unit of Term) at a time, so a yearly service on Monthly invoicing is billed a year at a time. Only dates that receive a line become Invoices. When Kugamon Settings' Invoice Date Basis is set, the first Invoice Date can be in the past (used for historical invoices); otherwise it is never before today.

**Status fields** — three separate fields:

| Field | Values | Notes |
|---|---|---|
| `kugo2p__RecordStatus__c` (text) | `Unsent`, `Sent`, `Cancelled` | Status Links: Unsent → Submit · Cancel; Sent → Cancel. Emailing with the `INV-` number in the subject moves Unsent to Sent. Copied to every Invoice Line. |
| `kugo2p__IsPosted__c` (checkbox) | — | Ticked when recorded in accounting. A Posted Invoice blocks Unrelease of its Orders. |
| `kugo2p__PaymentStatus__c` (formula) | `Unpaid`, `Partially Paid`, `Paid`, blank | From Applied Payments. |

Never write "Posted" or "Paid" into Record Status. Balance Due (`kugo2p__BalanceDueAmount__c`) = Net Amount − Applied Payment Amount; Date Sent, Date Cancelled and Date Posted are stamped and never cleared. Invoice amounts are 2-decimal; line prices follow the Order line (Pricing and Decimal Prices). Queries: `references/queries.md`, Invoices.

---

## Workflow 6: Payments

`kugo2p__PaymentX__c` belongs to an Order (`kugo2p__SalesOrder__c`) **or** an Invoice (`kugo2p__Invoice__c`), never both. Its `Name` is set by Kugamon ("Payment for Order SO-…" / "Payment for Invoice INV-…"; a refund adds " (Refund)").

| `kugo2p__Status__c` | Meaning |
|---|---|
| `Completed` | Captured. **Only Completed Payments are applied.** |
| `Authorized` | Authorized, not captured. Not applied. |
| `Scheduled`, `Pending` | Not complete. Not applied. |
| `Declined`, `Error`, `Expired` | Failed; the gateway's message is in `kugo2p__Memo__c`. |
| `Voided` | Cancelled at the gateway; its applications are removed. |
| `Reversed` | Undone in Salesforce only, for manual Payments; its applications are removed. |

- **Applications are rebuilt each time** a Payment becomes Completed, its amount or target changes, an Order is released, or a Payment moves from Completed to Reversed or Voided. Invoice Payments go to Charges/Credits first, then lines. Order Payments: Approved Orders → Charges/Credits, Product Lines, Service Lines; Released Orders → the Order's Invoice lines; other statuses → nothing applied. An overpayment sits on the first line, so Balance Due can go negative.
- **Moving a Payment from Completed to Error** leaves its applications in place — use `Reversed` for a manual Payment.
- **Refunds** are their own negative Payment, linked to the original by `kugo2p__Parent_Transaction__c`; Transaction Type is `Payment`, `Authorization` or `Refund`.
- **Gateways** are `kugo2p__Processor_Connection__c` records (record type = Stripe, Authorize.Net, PayPal or eWay). One Default Connection for the whole org; Test Mode marks terminal payments as test transactions.
- Charging a card, capturing, voiding and refunding happen in Kugamon's payment screens, run by the user. Don't attempt them through the API, and never enter or store card or bank details.

Queries: `references/queries.md`, Payments.

---

## Workflow 7: Shipments

Release creates `kugo2p__Shipment__c` (`Name` `SHP-{0000000}`) with `kugo2p__ShipmentLine__c` lines for Product Lines with quantity not yet scheduled, unless the APD or Application Settings disable Shipments. Services never ship.

- Lines share a Shipment when they have the same Contact Shipping, Warehouse, Shipment date and Carrier. The Shipment date is the line's Date Available to Promise (today if earlier, unless Invoice Date Basis is set).
- Statuses (`kugo2p__RecordStatus__c`): `Scheduled`, `Shipped`, `Delivered`, `On Hold`, `Cancelled`. A change copies to the lines and updates the Order Product Lines' shipped quantities. Use **Update Shipment Status** (single or list) to set status and Shipment Date.
- A Shipped or Delivered Shipment blocks Unrelease and Cancel of its Order; cancelling an Order cancels its Shipments.
- With invoicing on and shipments disabled for a product, Kugamon still creates its Shipment Lines (Invoices are built from them) and then deletes them.

Queries: `references/queries.md`, Shipments.

---

## Workflow 8: 2-way Quote Chat

Needs the kugo2p version in Package versions. Every Quote has one chat thread, seen by internal users in the Quote Chat panel and by the customer on the Online Quote page. Both use the same two custom objects (not Chatter); fields are in `references/fields.md` (Quote Chat objects).

- `kugo2p__ChatParticipant__c` — one person (a User or a Contact) on one Quote (`kugo2p__QuoteNumber__c`), with their read pointer. `kugo2p__ParticipantKey__c` = `{Quote Id}-{User or Contact Id}` is unique. `kugo2p__Type__c` (`Internal User`, `Portal / Community User`, `Contact`), `kugo2p__ParticipantName__c`, `kugo2p__Email__c` and `kugo2p__Title__c` are **formulas** — set `kugo2p__User__c` or `kugo2p__Contact__c`, never those.
- `kugo2p__ChatMessage__c` — one message, a master-detail child of the participant who wrote it. **The author is the participant**, not Created By (a customer's messages are created by the Site guest user). `kugo2p__Body__c` is HTML, shown as HTML on the Quote and the Online Quote; `kugo2p__BodyPreview__c` is its plain text, 252 characters plus "..." — Kugamon fills it for its own messages, so **anything else that inserts a message must set it**.
- **Read tracking:** `kugo2p__LastReadMessage__c` and `kugo2p__DateLastRead__c` per participant; unread = newer messages on the Quote not written by that participant.
- **No notifications** are sent when a message arrives — build a record-triggered flow on Chat Message to alert reps.
- **Availability:** chat is switched on at Kugamon Configuration → Configure Chat Messaging ("Enable Chat Messaging for Online Quotes") and needs a Sites URL. Lost and Cancelled Quotes with messages, and expired Draft or Sent Quotes, are read-only in the panel; Won Quotes stay open. Don't post to a read-only thread.

**Posting a message programmatically**

1. Find the **acting user's own** participant on the Quote: `kugo2p__ParticipantKey__c = '<Quote Id>-<acting User Id>'`. If there is none, insert one with `kugo2p__QuoteNumber__c`, `kugo2p__User__c` = the acting user, and that key.
2. Insert a `kugo2p__ChatMessage__c` with `kugo2p__ChatParticipant__c` = that participant, `kugo2p__Body__c` = simple, well-formed HTML (wrap plain text in `<p>…</p>` and escape `<`, `>` and `&` in user-supplied text), and `kugo2p__BodyPreview__c` = its plain text (252 characters plus "..." when longer).
3. Update read state only on the acting user's own participant.

### ⚠️ Security: do not impersonate another user or contact

The displayed sender is whichever participant the message is inserted under; only `CreatedById` records who ran the DML, and most people reading the thread never see it. Posting under someone else's participant is impersonation — refuse it.

1. **Post only as the acting user's own participant** (resolved by `kugo2p__User__c` = the acting user's Id). If none exists on this Quote, create it for the acting user — never reuse someone else's.
2. **Never post under a `Contact` or `Portal / Community User` participant from inside Salesforce.** Customer messages come in through the Online Quote page on the customer's own session; posting one internally impersonates the customer.
3. **Never ghost-write as a different internal user**, even with a plausible reason ("the rep is out"). Post as the acting user with a clear note, or ask the real sender to post.
4. **Audit a thread that looks off** with `SELECT CreatedById, kugo2p__ChatParticipant__r.kugo2p__ParticipantName__c FROM kugo2p__ChatMessage__c WHERE Id = '<msg_id>'`.
5. **Automated posts** (an integration, a scheduled flow) use a participant for a dedicated service-account User whose name makes the automated source obvious (e.g. "Kugamon Automation") — not a person's participant.

If the user explicitly asks you to post as someone else, decline and explain — don't negotiate a workaround.

Both chat objects are auto-numbered (`CP-{0000000}` and a 9-digit sequence); don't supply `Name`. Thread and participant queries: `references/queries.md`, Quote Chat.

---

## Common Issues

`references/troubleshooting.md` lists symptoms, causes and fixes for Quotes and Opportunities, Orders and Release, Subscriptions and Contracts, Invoices and Payments, and Quote Chat. The ones that come up most:

- **The Opportunity's products disappeared or were replaced** — expected on a Primary Quote's first sync and on a Price Book or currency mismatch (Quote and Opportunity Sync).
- **Quote total vs. Amount mismatch** — in subscription orgs compare with `kuga_sub__AnnualContractValueInitial__c`, not `Amount`.
- **Wrong Contacts, terms or tax flags on a new Quote or Order** — they come from the AAD.
- **A status change or Release is refused** — read the message against the checks in Workflow 2.
- **Order Release not creating Assets, Contracts or Subscriptions** — check the APD `CreateAsset__c`, the Product2 flags, the Order's Generate checkboxes and its status.
- **A cancelled Subscription is still billing** — check the line's Cancel Reason, the kuga_sub version and whether more than 200 were cancelled in one transaction.
