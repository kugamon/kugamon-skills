---
name: kugamon-full-qtc-submgmt
description: Manage the full Kugamon Quote-to-Cash lifecycle in Salesforce — opportunities, quotes, orders, invoices, payments, shipments, and assets, which uses the kugo2p namespace (Kugamon Quote to Cash). And optionally Managed the full Kugamon Subscription Billing lifecycle  in Salesforce - opportunities, quotes, orders, invoices, payments, shipments, contracts, subscriptions, and assets, which requires the kuga_sub namespace (Kugamon Subscription Management). Detects which packages are installed and adapts accordingly. Use when users request operations on any Kugamon object.
version: 0.3.0
status: Beta
---

# Kugamon Use Cases:

Full lifecycle skill for Kugamon RevOps for Salesforce

**CPQ Flow:** Opportunity → Quote → Order → (Order Release) → Asset

Full lifecycle skill for Kugamon Quote to Cash, which is a combination of CPQ and Billing functions

**Q2C Flow:** Opportunity → Quote → Order → (Order Release) → Asset + Shipment → Invoice → Payment 

Full lifecycle skill for Kugamon Subscription Management, which is a combination of CPQ and Subscription Management functions

**SubMgmt Flow:** Opportunity → Quote → Order → (Order Release) → Asset + Contract + Subscription + Renewal Opportunity 

Full lifecycle skill for Kugamon Subscription Billing, which is combination of all Quote to Cash and Subscription Management functions

**SB Flow:** Opportunity → Quote → Order → (Order Release) → Asset + Shipment + Contract + Subscription + Renewal Opportunity → Invoice → Payment 



## Package Detection

**FIRST STEP: Always detect which packages are installed.**

Check if `kuga_sub__Renew__c` exists on `OpportunityLineItem` by describing the object's fields via the connected Salesforce MCP or API.

- `HAS_KUGA_SUB = true` → kuga_sub package is installed (subscription management, contracts, assets, renewals)
- `HAS_KUGA_SUB = false` → only kugo2p (Q2C only, no subscription lifecycle)

This flag controls revenue classification, line-item separation, and whether Order Release creates contracts/assets/subscriptions.

---

## Decimal Price Precision (kugo2p v11.0+)

As of Kugamon Quote to Cash v11.0, the price fields on quote lines, order lines, Account Pricing, Tier, and Asset were widened from 2 decimals to **6 decimals** (`Currency(12, 6)`). Invoice-line **stored** currency fields stay at 2 decimals, but invoice-line **formula** currency fields inherit the 6-decimal precision from the upstream Order line — so invoices render all 6 decimals end-to-end via formulas, not via stored fields.

### Where 6-decimal pricing is stored

| Object | Stored fields that accept up to 6 decimals |
|---|---|
| `kugo2p__SalesQuoteProductLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__SalesQuoteServiceLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__NonUpliftSalesPrice__c`, `kugo2p__UpliftPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__SalesQuoteOptionalLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c` |
| `kugo2p__SalesOrderProductLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__SalesOrderServiceLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__NonUpliftSalesPrice__c`, `kugo2p__UpliftPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__AccountPricing__c` | `kugo2p__Price__c` |
| `kugo2p__Tier__c` | `kugo2p__TierPrice__c` |
| `Asset` | `kugo2p__PurchasePrice__c` |

### Per-product rounding: `kugo2p__PriceScale__c` on APD

A new picklist field on `kugo2p__AdditionalProductDetail__c` controls how many decimals are **used** for each product:

- `kugo2p__PriceScale__c` — Picklist. Observed values: `4`, `5`, `6`. Blank (null) means the org default (2 decimals) is used.
- Related picklists shipped at the same time: `kugo2p__QuantityScale__c`, `kugo2p__ServiceTermScale__c`.
- The Quote and Order Lightning Configurator (Add Lines / Configure Lines / Edit Lines) honors `PriceScale__c` when displaying and validating list price and sales price for the product. Tiered Pricing honors it too.

### Formula fields inherit the 6-decimal precision

Several key currency fields on quote lines, order lines, AND invoice lines are **Formula (Currency)** fields that reference the stored 6-decimal fields upstream — formulas display at the precision of the field they read, so these carry 6 decimals end-to-end:

- `kugo2p__DiscountSalesPrice__c` — **label: "Effective Price"**. Formula on quote lines, order lines, AND invoice lines. The computed per-unit price after line discount.
- On `kugo2p__KugamonInvoiceLine__c`, additional Formula (Currency) fields inherit precision from the related order line: `kugo2p__SalesPrice__c` (yes, invoice line's SalesPrice is itself a formula that reads back from the Order line), `kugo2p__LineAmount__c`, `kugo2p__NetAmount__c`, `kugo2p__TotalAmount__c`, `kugo2p__TaxAmount__c`, `kugo2p__VATAmount__c`, `kugo2p__LineDiscountAmount__c`, `kugo2p__BalanceDueAmount__c`.

So invoice-line pricing DOES carry the full 6-decimal precision when the upstream Order line uses it — just via formulas, not stored fields. Invoice PDF and online templates render all 6 decimals (v11.0+).

### What does NOT change

- `kugo2p__KugamonInvoiceLine__c` **stored** currency fields are still 2-decimal (`kugo2p__AppliedPaymentAmount__c` is `Currency(16, 2)`). The 6-decimal precision reaches invoices via the Formula fields above, which read back from the parent Order line.
- `PricebookEntry.UnitPrice` (standard Salesforce) is 2-decimal — you cannot store a 6-decimal list price on a pricebook entry. The 6 decimals come in on the line via `ListPrice__c` and `SalesPrice__c`.
- `OpportunityLineItem.UnitPrice` is 2-decimal. If you need the full-precision price flowing from a quote back to the opportunity, read the quote/order line, not the OLI.

### SOQL / DML rules

1. When reading a 6-decimal field, do **not** cast or round it client-side unless the user explicitly asks — the fourth-through-sixth decimals carry real pricing meaning.
2. When writing a 6-decimal field via DML, pass the full value (e.g. `1234.567891`). Kugamon's APD-driven validation enforces the per-product `PriceScale__c` cap.
3. Document templates (quote/order PDF, online quote, online invoice) and the Lightning Configurator render all 6 decimals as of v11.0 — don't truncate in custom display logic.
4. Opportunity amount roll-ups (`kuga_sub__Amount__c`, `kuga_sub__ARR__c`, etc.) continue to roll up at their native (2-decimal) precision — line-level precision does not propagate up.

The full matrix of 6-decimal stored fields AND Formula (Currency) fields that inherit the precision is in **Appendix A: 6-Decimal Price Fields (v11.0+)**.

---

## ⚠️ CRITICAL: Opportunity Pipeline Forecasting Field

**When `HAS_KUGA_SUB = true` (Kugamon Subscription Management is installed), ALWAYS use `kuga_sub__Amount__c` for Opportunity pipeline forecasting. DO NOT use the standard Salesforce `Amount` field.**

### Why this matters

The standard `Amount` field on Opportunity is unreliable in subscription orgs — it can be configured to display MRR, ACV, TCV, or some other value, and the meaning varies by org and even by opportunity. Using it for forecasting will produce incorrect pipeline numbers.

`kuga_sub__Amount__c` is a Roll-Up SUM field maintained by the Kugamon Subscription Management package. It aggregates the correct revenue values from OpportunityLineItems and is the authoritative figure for pipeline reporting in subscription orgs.

### Rules

1. **`HAS_KUGA_SUB = true`** → use `kuga_sub__Amount__c` for all pipeline forecasts, revenue reports, dashboards, and aggregate opportunity-level reporting. Never substitute the standard `Amount` field.
2. **`HAS_KUGA_SUB = false`** → use the standard `Amount` field as normal (the `kuga_sub__*` fields don't exist).
3. When a user asks about "opportunity amount," "pipeline value," or "forecast" in a subscription org, default to `kuga_sub__Amount__c` and briefly explain why.
4. When building SOQL queries, reports, or list views for pipeline in a subscription org, select `kuga_sub__Amount__c` — not `Amount`.

### Quick example

```sql
-- CORRECT (HAS_KUGA_SUB = true):
SELECT Id, Name, kuga_sub__Amount__c
FROM Opportunity
WHERE IsClosed = false

-- WRONG (HAS_KUGA_SUB = true):
SELECT Id, Name, Amount
FROM Opportunity
WHERE IsClosed = false
```

See **Appendix B: Amount Fields Guide** for detailed field-by-field reference and comparison rules.

---

## Workflow 8: 2-way Quote Chat (kugo2p v11.0+)

Kugamon Quote to Cash v11.0 added an in-app chat on every Quote, plus a 2-way variant that works through the Online Quote page so an external contact and an internal user can message each other without leaving their respective surfaces. Chat is backed by two custom objects, not Chatter.

### Objects

| Object | Key Prefix | Purpose |
|---|---|---|
| `kugo2p__ChatParticipant__c` | `a0B` | One record per person (internal User **or** external Contact) authorized to chat on a specific quote. Also tracks each person's read state. |
| `kugo2p__ChatMessage__c` | `a0H` | One record per message posted. Each message belongs to a single `ChatParticipant__c` — that participant is the sender. |

### Relationship

```
kugo2p__SalesQuote__c  (the quote)
      ↑
      │  kugo2p__QuoteNumber__c (lookup on Participant)
      │
kugo2p__ChatParticipant__c  (sender identity + read state)
      ↑
      │  kugo2p__ChatParticipant__c (lookup on Message)
      │
kugo2p__ChatMessage__c   (message content)
```

The sender's **type** is on the Participant, not the Message:
- `kugo2p__ChatParticipant__c.kugo2p__Type__c = 'Internal User'` → populated `kugo2p__User__c` (lookup to User), empty `kugo2p__Contact__c`.
- `kugo2p__ChatParticipant__c.kugo2p__Type__c = 'Contact'` → populated `kugo2p__Contact__c` (lookup to Contact), empty `kugo2p__User__c`.

A composite key, `kugo2p__ParticipantKey__c = {QuoteId}-{UserOrContactId}`, prevents duplicate participants on the same quote.

### Internal vs. 2-way online quote chat

The same objects power two experiences:

1. **Internal chat on the Quote record** — Lightning component (shipped as part of v11.0's "Chat Messaging Function to Quotes") lets internal users thread messages. All participants are `Type = 'Internal User'`.
2. **2-way chat on the Online Quote page** — the external contact replies through the signed online quote link (Email Quote Link functionality from v10.2). Their messages insert as `Type = 'Contact'` participants on the same quote's thread, visible to the internal team in real time.

Both surfaces read and write the exact same two objects. There is no separate "external chat" container.

### Read-receipt / unread tracking

Each participant row tracks what they have personally read:
- `kugo2p__LastReadMessage__c` (lookup → ChatMessage__c) — the last message that participant has seen.
- `kugo2p__DateLastRead__c` (datetime) — when they saw it.

An "unread count" is computed by counting messages on the quote newer than the participant's `LastReadMessage__c` (or `DateLastRead__c`), excluding messages they themselves authored. The package updates these fields as the participant views the chat.

### Message storage

- `kugo2p__Body__c` — Rich text / HTML (textarea). The actual message as typed. Example: `<p>any questions?</p>`.
- `kugo2p__BodyPreview__c` — Plain-text truncated preview, used in list views, email notifications, and the inline message tile. Example: `any questions?`.
- `kugo2p__ParticipantName__c` on the Message — denormalized sender display name, kept in sync with the Participant's name so a long thread doesn't need the lookup resolved on read.

### Sample: read a quote's full chat thread

```sql
SELECT Id, Name, CreatedDate,
       kugo2p__ChatParticipant__r.Name,
       kugo2p__ChatParticipant__r.kugo2p__ParticipantName__c,
       kugo2p__ChatParticipant__r.kugo2p__Type__c,
       kugo2p__Body__c, kugo2p__BodyPreview__c
FROM kugo2p__ChatMessage__c
WHERE kugo2p__ChatParticipant__r.kugo2p__QuoteNumber__c = '<quote_id>'
ORDER BY CreatedDate ASC
```

### Sample: list participants on a quote (with unread timestamps)

```sql
SELECT Id, Name, kugo2p__ParticipantName__c, kugo2p__Type__c,
       kugo2p__User__r.Name, kugo2p__Contact__r.Name, kugo2p__Email__c,
       kugo2p__LastReadMessage__c, kugo2p__DateLastRead__c
FROM kugo2p__ChatParticipant__c
WHERE kugo2p__QuoteNumber__c = '<quote_id>'
ORDER BY CreatedDate ASC
```

### Posting a message on behalf of a user (programmatic)

1. Resolve the participant row for the sender on this quote (query by `kugo2p__ParticipantKey__c = '<quoteId>-<userOrContactId>'`). If none, insert one first with the right `Type__c` + `User__c`/`Contact__c`.
2. Insert a `kugo2p__ChatMessage__c` with:
   - `kugo2p__ChatParticipant__c` = that participant's Id
   - `kugo2p__Body__c` = HTML (even if plain text, wrap in `<p>…</p>`)
   - `kugo2p__BodyPreview__c` = plain-text truncation of Body
   - `kugo2p__ParticipantName__c` = sender's display name (denormalized)
3. The package maintains the recipient's unread state — do not try to write `LastReadMessage__c` on anyone other than the viewing participant.

### Naming conventions

Both chat objects are AutoNumber, so **do not supply `Name`**:
- `kugo2p__ChatMessage__c.Name` — 9-digit zero-padded sequence, e.g. `000000165`.
- `kugo2p__ChatParticipant__c.Name` — `CP-{0000000}` prefix, e.g. `CP-0000070`.

See **Appendix E** for the full sample-data naming matrix.

### Related v11.0 touch-ups to the online quote

v11.0 also shipped **new Quote Acceptance and Rejection email templates**, which are triggered by the same online-quote actions the external contact takes while they are chatting. If a prospect accepts (or rejects) via the online quote, the chat thread survives on the quote and is a useful handoff artifact for whoever picks up the account.

---

## Object Model Overview

### kugo2p Objects (Kugamon Quote to Cash)

See **Appendix A** for full field reference. Key custom objects by stage:

**Quote / Order / Invoice / Payment / Fulfillment stages**: `kugo2p__SalesQuote__c`, `kugo2p__SalesQuoteServiceLine__c`, `kugo2p__SalesQuoteProductLine__c`, `kugo2p__SalesQuoteOptionalLine__c`, `kugo2p__SalesQuoteAdditionalChargeCredit__c`, `kugo2p__QuoteLineGroup__c`, `kugo2p__SalesOrder__c`, `kugo2p__SalesOrderServiceLine__c`, `kugo2p__SalesOrderProductLine__c`, `kugo2p__SalesOrderAdditionalChargeCredit__c`, `kugo2p__OrderLineGroup__c`, `kugo2p__KugamonInvoice__c`, `kugo2p__KugamonInvoiceLine__c`, `kugo2p__KugamonInvoiceAdditionalChargeCredit__c`, `kugo2p__InvoiceSchedule__c`, `kugo2p__OrderInvoiceRelationship__c`, `kugo2p__PaymentX__c`, `kugo2p__AppliedPayment__c`, `kugo2p__Payment_Method__c`, `kugo2p__Payment_Profile__c`, `kugo2p__Processor_Connection__c`, `kugo2p__Shipment__c`, `kugo2p__ShipmentLine__c`, `kugo2p__ServiceDeliverySchedule__c`, `kugo2p__Carrier__c`, `kugo2p__Warehouse__c`.

**Product & Pricing**: `kugo2p__AdditionalProductDetail__c` (APD, ~67 fields as of v11.0 — v11.0 added `kugo2p__PriceScale__c`, `kugo2p__QuantityScale__c`, `kugo2p__ServiceTermScale__c` picklists that control per-product decimal precision for price, quantity, and service term; see "Decimal Price Precision" above), `kugo2p__AccountPricing__c`, `kugo2p__TieredPricing__c`, `kugo2p__Tier__c`, `kugo2p__ProductCost__c`, `kugo2p__AdditionalChargeCredit__c`, `kugo2p__ProductCatalog__c`, `kugo2p__ProductCategory__c`.

**Configuration & Bundles**: `kugo2p__ConfigurationGroup__c`, `kugo2p__ConfigurationOption__c`, `kugo2p__KitBundleMember__c`.

**Tax**: `kugo2p__TaxLocation__c`, `kugo2p__TaxRate__c`, `kugo2p__VAT__c`, `kugo2p__VATRate__c`.

**Account & Settings**: `kugo2p__AdditionalAccountDetail__c`, `kugo2p__KugamonSetting__c`, `kugo2p__Settings__c`.

**Quote Chat (v11.0+)**:
- `kugo2p__ChatParticipant__c` (20 fields) — one record per person (internal User or external Contact) authorized to chat on a quote. Carries read-receipt state (`kugo2p__LastReadMessage__c`, `kugo2p__DateLastRead__c`). See Workflow 8.
- `kugo2p__ChatMessage__c` (12 fields) — one record per message. HTML body in `kugo2p__Body__c`, plain-text preview in `kugo2p__BodyPreview__c`. Linked to its sender via `kugo2p__ChatParticipant__c`.

### kuga_sub Objects (Kugamon Subscriptions — only when HAS_KUGA_SUB = true)

**Custom**: `kuga_sub__Subscription__c` (34 fields) — the subscription record linking orders to contracts.

**Fields added by kuga_sub to standard / kugo2p objects:**

- **On Product2**: `kuga_sub__Renewable__c` (drives "Renewable" prefix on Setup labels AND triggers Renewal Opportunity creation on Order Release — does NOT create Subscription), `kuga_sub__RenewalProduct__c` (substitute product for renewal), `kuga_sub__Track__c` (label "Create Subscription" — generates Subscription on Order Release but only for Order Service Lines where `APD.kugo2p__Service__c = true`; Order Product Lines never generate Subscriptions), `kuga_sub__UpliftRenewalPrice__c`. Asset creation is separate and Product-only: APD `kugo2p__CreateAsset__c` drives Asset creation, and only Order Product Lines generate Assets.

- **On OpportunityLineItem** (18 fields): `kuga_sub__Renew__c` (CRITICAL — recurring vs one-time), ARR/MRR/NonRecurringRevenue, ServiceTerm/UnitofTerm/DateServiceEnd, NetAmount/TotalAmount/ListAmount, Service formula, forecasting fields, discount tracking, uplift tracking.

- **On Opportunity** (19 fields): roll-up MRR/ARR/NonRecurringRevenue, formula ACV / TCV / ARR Forecast / Expected Revenue, parent contract / parent order lookups, renewal controls.

- **On kugo2p__SalesOrder__c** (16 fields, Order Release controls): `kuga_sub__GenerateContract__c`, `kuga_sub__GenerateAsset__c`, `kuga_sub__GenerateSubscription__c`, `kuga_sub__GenerateRenewalOpportunity__c`, contract lookups, roll-up counts of renewable/trackable products/services.

- **On kugo2p__SalesOrderServiceLine__c**: `kuga_sub__Renew__c` (revenue classification), `kuga_sub__Track__c` (generates Subscription — Order Service Lines never generate Assets).

- **On kugo2p__SalesOrderProductLine__c**: `kuga_sub__Renew__c`, `kuga_sub__Track__c` (present but no effect on product lines — Order Product Lines never generate Subscriptions). Order Product Lines generate Assets when APD `kugo2p__CreateAsset__c = true`.

- **On Contract** (21 fields): roll-up ARR/MRR/subscription count/dates, formula Effective, renewal notice controls.

- **On Asset** (4 fields): `kuga_sub__ContractNumber__c`, `kuga_sub__ParentSubscription__c`, `kuga_sub__ParentLine__c`, `kuga_sub__Renew__c` formula.

**Line-object split for Order Release** — Subscriptions come only from Order Service Lines, Assets come only from Order Product Lines, no matter how the underlying flags are set. See **Workflow 3: Order Release** for details.

---

## Apex Triggers / Apex Class Logic

(The detailed trigger inventory and class-logic descriptions from v0.2.x are preserved — see Appendix A and Workflow 3 for the fields and behaviors they govern. The Order Release Trigger Map is the key diagram: Subscription creation is Order Service Line trigger only, Asset creation is Order Product Line trigger only, Renewal Opportunity creation can come from any line.)

### Order Release Trigger Map

```
SUBSCRIPTION  (Order Service Line trigger only)
─────────────
Product2.kuga_sub__Track__c (label: "Create Subscription")
   └─ propagates → OLI / QuoteLine / OrderLine .kuga_sub__Track__c
       └─ on Order Release, when the line is a Service
          (lands on kugo2p__SalesOrderServiceLine__c; APD.Service__c = true):
              └─→ Order Service Line trigger creates a Subscription
       └─ on Order Product Lines: no Subscription, ever

ASSET  (Order Product Line trigger only)
─────
kugo2p__AdditionalProductDetail__c.kugo2p__CreateAsset__c
   └─ on Order Release, when the line is a Product
      (lands on kugo2p__SalesOrderProductLine__c; APD.Service__c = false):
          └─→ Order Product Line trigger creates an Asset
   └─ on Order Service Lines: no Asset, ever

RENEWAL OPPORTUNITY  (any line)
───────────────────
Product2.kuga_sub__Renewable__c
   └─ on Order Release, if true on any line's product (service or product):
       └─→ Renewal Opportunity created
   └─ also drives the "Renewable" prefix on the Product Snapshot LWC
```

**Separate concept — revenue classification, not Subscription creation:** `kuga_sub__Renew__c` on `OpportunityLineItem` / Quote Lines / Order Lines classifies revenue as recurring (MRR/ARR) vs one-time (NonRecurringRevenue). It does NOT trigger Subscription creation. See **Appendix D: Renew Field Guide**.

---

## Workflow 1: Quote Creation

Flow: validate billing address + contact on Account → verify Opportunity (and its amount fields, honoring the Pipeline Forecasting Rule above) → look for existing primary quote → create `kugo2p__SalesQuote__c` with required fields (RecordTypeId dynamically queried, Account, Opportunity, Pricebook, `ContactBuying__c` REQUIRED). Never set auto-managed fields (`Name`, `Status__c`, totals). Lines insert as `SalesQuoteProductLine__c` or `SalesQuoteServiceLine__c` based on `APD.kugo2p__Service__c`. When `HAS_KUGA_SUB = true`, OpportunityLineItems need `kuga_sub__Renew__c` set.

## Workflow 2: Order Management

Orders are created from quotes via the Kugamon UI "Create Order" action. Key fields on `kugo2p__SalesOrder__c`: Account, Opportunity, SalesQuote, Pricebook, RecordType, contact fields, DateOrdered. Line items split into `SalesOrderServiceLine__c` and `SalesOrderProductLine__c` by APD Service flag.

## Workflow 3: Order Release (HAS_KUGA_SUB = true only)

When an order is "released," kuga_sub triggers create downstream records.

### Order Release Checkboxes (on `kugo2p__SalesOrder__c`)

| Field | What It Creates |
|---|---|
| `kuga_sub__GenerateContract__c` | Contract |
| `kuga_sub__GenerateAsset__c` | Assets (only from Order Product Lines, when APD `kugo2p__CreateAsset__c = true`) |
| `kuga_sub__GenerateSubscription__c` | Subscriptions (only from Order Service Lines, when `Track__c = true`) |
| `kuga_sub__GenerateRenewalOpportunity__c` | Renewal Opportunity (from any line whose product has `kuga_sub__Renewable__c = true`) |

### What Gets Created

- **Contract** — linked to the order via `kuga_sub__ContractNumber__c`; contacts copied per `UpdateContractContacts__c`; subscriptions roll up ARR/MRR/counts/dates; `kuga_sub__Effective__c` formula flags currently-active contracts.
- **Assets** — created **only** from Order Product Lines when `kugo2p__AdditionalProductDetail__c.kugo2p__CreateAsset__c = true`. **Order Service Lines never generate Assets.** APD field is in the `kugo2p` namespace, not `kuga_sub`, and not on Product2.
- **Subscriptions** — created **only** from Order Service Lines when `kuga_sub__Track__c = true` on the line (usually propagated from `Product2.kuga_sub__Track__c`, label: "Create Subscription"). **Order Product Lines never generate Subscriptions.** `kuga_sub__Renew__c` on the line is for revenue classification only (see Appendix D), not Subscription creation.
- **Renewal Opportunity** — created when `Product2.kuga_sub__Renewable__c = true` for any line on the released order; sets `kuga_sub__ParentContract__c` / `kuga_sub__ParentOrder__c`; close date from contract end date; `kuga_sub__RenewalPriceUpliftPercent__c` carries forward.

### Product2 (and APD) Flags for Order Release

| Field | Object | Effect |
|---|---|---|
| `kuga_sub__Track__c` (label: "Create Subscription") | `Product2` | When `true` AND the line lands on an **Order Service Line** (`APD.Service__c = true`): Order Service Line trigger generates a Subscription. Ignored on Order Product Lines. |
| `kugo2p__CreateAsset__c` | `kugo2p__AdditionalProductDetail__c` (APD) | When `true` AND the line lands on an **Order Product Line** (`APD.Service__c = false`): Order Product Line trigger generates an Asset. Ignored on Order Service Lines. |
| `kuga_sub__Renewable__c` | `Product2` | When `true` on any line's product: triggers Renewal Opportunity creation. Also drives the "Renewable" prefix on the Product Snapshot LWC label. |
| `kuga_sub__RenewalProduct__c` | `Product2` | Substitute product used on renewal quotes. |
| `kuga_sub__UpliftRenewalPrice__c` | `Product2` | Apply renewal price uplift percentage. |

## Workflow 4: Subscription & Contract Management

```sql
SELECT Id, ContractNumber, Account.Name, kuga_sub__Effective__c,
       kuga_sub__AnnualRecurringRevenue__c, kuga_sub__MonthlyRecurringRevenue__c,
       kuga_sub__TotalSubscriptionCount__c, kuga_sub__SubscriptionStartDate__c,
       kuga_sub__SubscriptionEndDate__c, kuga_sub__RenewalOpportunity__r.Name
FROM Contract WHERE kuga_sub__Effective__c = true AND AccountId = '<account_id>'

SELECT Id, Name, kuga_sub__Service__r.Name, kuga_sub__Quantity__c,
       kuga_sub__MRR__c, kuga_sub__ARR__c, kuga_sub__StartDate__c, kuga_sub__EndDate__c,
       kuga_sub__Status__c, kuga_sub__Renew__c, kuga_sub__Active__c
FROM kuga_sub__Subscription__c WHERE kuga_sub__ContractNumber__c = '<contract_id>'
ORDER BY kuga_sub__Service__r.Name
```

Renewal automation: `ContractRenewalNoticeBatcher` sends renewal emails N days before contract end; `RenewalOrderBatcher` creates renewal orders N days before end.

## Workflow 5-7: Invoice / Payment / Shipment

Invoices: `kugo2p__KugamonInvoice__c` + `kugo2p__KugamonInvoiceLine__c` (note: invoice-line currency fields are formulas that inherit 6-decimal precision from the parent order line — see Appendix A). Payment gateways via `kugo2p__Processor_Connection__c` (Stripe, Authorize.Net, PayPal, eWay). Shipments via `kugo2p__Shipment__c` + `kugo2p__ShipmentLine__c`.

---

## Product Setup

### AdditionalProductDetail (`kugo2p__AdditionalProductDetail__c`, aka APD)

Extended per-product metadata auto-created by `Product2Trigger`. Key fields:
- `kugo2p__Service__c` — **CRITICAL** when `HAS_KUGA_SUB = false`: product vs service classification; also decides which Order line object each line lands on, which in turn decides whether Subscription or Asset creation can fire.
- `kugo2p__CreateAsset__c` — drives Asset creation on Order Release (Order Product Lines only).
- `kugo2p__PriceScale__c`, `kugo2p__QuantityScale__c`, `kugo2p__ServiceTermScale__c` — v11.0 picklists for per-product decimal precision. See "Decimal Price Precision" at top.
- `kugo2p__Taxable__c`, `kugo2p__Configurable__c`, `kugo2p__Kit__c`.

### Setup Types

Every product resolves to one of six Setup types based on independent flags:

| # | Setup type | `Service__c` | `DisableShipments__c` | `Renewable__c` | kuga_sub installed |
|---|---|---|---|---|---|
| 1 | `{Term} {Unit} Service` | `true` | n/a | `false` or n/a | optional |
| 2 | `Renewable {Term} {Unit} Service` | `true` | n/a | `true` | required |
| 3 | `Product` | `false` | `true` | `false` or n/a | optional |
| 4 | `Shippable Product` | `false` | `false` / blank | `false` or n/a | optional |
| 5 | `Renewable Product` | `false` | `true` | `true` | required |
| 6 | `Renewable Shippable Product` | `false` | `false` / blank | `true` | required |

**Rule**: if `Renewable__c = true`, label starts with "Renewable"; `Service__c = true` ends with `{Term} {Unit} Service`; otherwise it's `Product` (DisableShipments = true) or `Shippable Product` (DisableShipments = false).

**Label vs behavior**: The Setup label only reflects Service, DefaultServiceTerm/UnitofTerm, DisableShipments, and Renewable. Subscription and Asset creation are governed by `Track__c` and `CreateAsset__c` — and the two are split by line-object: Subscriptions only from Order Service Lines, Assets only from Order Product Lines.

**Downstream creation** (Order Release):

| Setup type | Creates |
|---|---|
| `{Term} {Unit} Service` | Order Service Line + Subscription if `Track__c = true`. (No Asset — services never create Assets.) |
| `Renewable {Term} {Unit} Service` | Order Service Line + **Renewal Opportunity**. + Subscription if `Track__c = true`. (No Asset.) |
| `Product` | Order Product Line + Asset if APD `CreateAsset__c = true`. (No Subscription — products never create Subscriptions.) |
| `Shippable Product` | Order Product Line + Shipment. + Asset if APD `CreateAsset__c = true`. (No Subscription.) |
| `Renewable Product` | Order Product Line + **Renewal Opportunity**. + Asset if APD `CreateAsset__c = true`. (No Subscription.) |
| `Renewable Shippable Product` | Order Product Line + Shipment + **Renewal Opportunity**. + Asset if APD `CreateAsset__c = true`. (No Subscription.) |

---

## Consistency Checking and Synchronization

When updating quotes OR opportunity line items, ALWAYS check for consistency and synchronize both sides unless explicitly told not to. Compare: Product/Service, Quantity, Unit Price, Start Date (and Term/End Date when HAS_KUGA_SUB). Default: update both sides.

---

## Common Issues

- **Quote total vs. Amount mismatch**: HAS_KUGA_SUB=true → `Amount` may be MRR, compare to `kuga_sub__AnnualContractValueInitial__c`. HAS_KUGA_SUB=false → Quote total should match OLI sum.
- **ACV double-counting**: Products with `kuga_sub__Renew__c = false` but non-zero MRR/ARR. Fix: set Renew = true on recurring lines.
- **Line items not populating**: check both ProductLine and ServiceLine; verify APD.Service__c.
- **Order Release not creating contracts/subscriptions**: verify Order Release checkboxes AND the Product2/APD flags.
- **Missing billing address / contact**: must be resolved before quote creation.

---

## Appendix A: Field Reference

### Opportunity Fields

Required: `Name`, `StageName`, `CloseDate`. Strongly recommended: `AccountId`, `Pricebook2Id`. Optional: `Amount`, `Type`, `RecordTypeId`.

**kuga_sub fields on Opportunity** (HAS_KUGA_SUB only):
- Roll-up SUMs: `kuga_sub__MonthlyRecurringRevenue__c`, `kuga_sub__AnnualRecurringRevenueCommitted__c`, `kuga_sub__NonRecurringRevenue__c`, `kuga_sub__Amount__c`, `kuga_sub__DateRequired__c`, `kuga_sub__ServiceDateExpires__c`.
- Formulas: `kuga_sub__AnnualContractValueInitial__c` (ACV = NonRecurring + ARR), `kuga_sub__TotalContractValue__c`, `kuga_sub__AnnualRecurringRevenueForecast__c` (MRR x 12), `kuga_sub__ExpectedRevenue__c`, `kuga_sub__OpportunityAmount__c`, `kuga_sub__ContractEndDate__c`, `kuga_sub__ParentContractEndDate__c`.
- Editable: `kuga_sub__ParentContract__c`, `kuga_sub__ParentOrder__c`, `kuga_sub__AutoEmailRenewalOrder__c`, `kuga_sub__AutoRenewedOrder__c`, `kuga_sub__RenewalOrderAutoCreationDate__c`, `kuga_sub__RenewalPriceUpliftPercent__c`.

### OpportunityLineItem

Required: `OpportunityId`, `PricebookEntryId`, `Quantity`. Standard optional: `UnitPrice`, `ServiceDate`, `Discount`, `Description`.

**kuga_sub fields** (HAS_KUGA_SUB only, editable): `kuga_sub__Renew__c` (CRITICAL), `kuga_sub__ServiceTerm__c`, `kuga_sub__UnitofTerm__c`, `kuga_sub__ServiceTermBehavior__c`, `kuga_sub__NonUpliftSalesPrice__c`. Calculated: `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__NonRecurringRevenue__c`, `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c`, `kuga_sub__ListAmount__c`, `kuga_sub__DateServiceEnd__c`, `kuga_sub__Service__c`, `kuga_sub__LineTerm__c`, `kuga_sub__DiscountSalesPrice__c`, `kuga_sub__EffectiveDiscount__c`, `kuga_sub__UpliftRenewalPrice__c`.

### Quote (`kugo2p__SalesQuote__c`)

Createable: `RecordTypeId`, `kugo2p__Account__c`, `kugo2p__Opportunity__c`, `kugo2p__QuoteName__c`, `kugo2p__Pricebook2Id__c`, `kugo2p__ContactBuying__c` (REQUIRED), `kugo2p__ContactBilling__c`, `kugo2p__ContactShipping__c`, `kugo2p__IsPrimary__c`, `kugo2p__DateOfferValidThrough__c`.

kuga_sub on Quote: `kuga_sub__ContractNumber__c`, `kuga_sub__ContractEndDate__c` (formula).

Auto-managed (never set): `Name` (`SQ-{YYMMDD}-{0000000}` — see Appendix E), `kugo2p__Status__c`, `kugo2p__TotalAmount__c`, `kugo2p__SubtotalAmount__c`, `kugo2p__NetAmount__c`.

### Order (`kugo2p__SalesOrder__c`)

kuga_sub Order Release controls: `kuga_sub__GenerateContract__c`, `kuga_sub__GenerateAsset__c`, `kuga_sub__GenerateSubscription__c`, `kuga_sub__GenerateRenewalOpportunity__c`, `kuga_sub__ContractNumber__c`, `kuga_sub__ParentContract__c`, `kuga_sub__RenewalOpportunity__c`, `kuga_sub__UpdateContractContacts__c`.

### Order Line kuga_sub Fields

| Field | On Object | Description |
|---|---|---|
| `kuga_sub__Renew__c` | Service & Product Lines | Revenue classification (recurring vs one-time). See Appendix D. Does NOT create Subscription. |
| `kuga_sub__Track__c` (label: "Create Subscription") | Service & Product Lines | Creates Subscription on Order Release **only when on an Order Service Line** (`kugo2p__SalesOrderServiceLine__c`). Propagated from `Product2.kuga_sub__Track__c`. Ignored on Order Product Lines. |

Asset creation is driven by APD `kugo2p__CreateAsset__c`, only on Order Product Lines.

### Subscription (`kuga_sub__Subscription__c`)

Key fields: `Name` (auto), `kuga_sub__Account__c`, `kuga_sub__ContractNumber__c`, `kuga_sub__Order__c`, `kuga_sub__OrderServiceLine__c`, `kuga_sub__Service__c` (Product2), `kuga_sub__Quantity__c`, `kuga_sub__PurchasePrice__c`, `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c`, `kuga_sub__StartDate__c`, `kuga_sub__EndDate__c`, `kuga_sub__Status__c`, `kuga_sub__Renew__c`, `kuga_sub__Active__c`, `kuga_sub__IsActive__c`, `kuga_sub__ParentAsset__c`, `kuga_sub__ParentSubscription__c`, `kuga_sub__ServiceTerm__c`, `kuga_sub__UnitofTerm__c`.

### Contract kuga_sub Fields

Roll-up: `kuga_sub__AnnualRecurringRevenue__c`, `kuga_sub__MonthlyRecurringRevenue__c`, `kuga_sub__TotalSubscriptionAmount__c`, `kuga_sub__TotalSubscriptionCount__c`, `kuga_sub__TotalSubscriptionQuantity__c`, `kuga_sub__SubscriptionStartDate__c`, `kuga_sub__SubscriptionEndDate__c`. Formula: `kuga_sub__Effective__c`, `kuga_sub__ContractRenewalNoticeDate__c`, `kuga_sub__SendRenewalNoticeToday__c`, `kuga_sub__AnnualRecurringRevenueForecast__c`.

### Asset kuga_sub Fields

`kuga_sub__ContractNumber__c`, `kuga_sub__ParentSubscription__c`, `kuga_sub__ParentLine__c` (formula), `kuga_sub__Renew__c` (formula).

### Product2 kuga_sub Fields

`kuga_sub__Renewable__c` (drives "Renewable" label + Renewal Opportunity creation — does NOT create Subscription), `kuga_sub__Track__c` (label "Create Subscription" — generates Subscription only on Order Service Lines), `kuga_sub__RenewalProduct__c`, `kuga_sub__UpliftRenewalPrice__c`. Asset creation is on APD, not Product2 (field `kugo2p__CreateAsset__c`, Order Product Lines only).

### 6-Decimal Price Fields (v11.0+)

Stored fields widened to `Currency(12, 6)` in Kugamon Quote to Cash v11.0. Precision above 2 decimals is honored per-product via `kugo2p__AdditionalProductDetail__c.kugo2p__PriceScale__c` (picklist values observed: `4`, `5`, `6`; blank = org default).

| Object | Stored 6-decimal fields |
|---|---|
| `kugo2p__SalesQuoteProductLine__c` | `ListPrice__c`, `SalesPrice__c`, `TierPrice__c` |
| `kugo2p__SalesQuoteServiceLine__c` | `ListPrice__c`, `SalesPrice__c`, `NonUpliftSalesPrice__c`, `UpliftPrice__c`, `TierPrice__c` |
| `kugo2p__SalesQuoteOptionalLine__c` | `ListPrice__c`, `SalesPrice__c` |
| `kugo2p__SalesOrderProductLine__c` | `ListPrice__c`, `SalesPrice__c`, `TierPrice__c` |
| `kugo2p__SalesOrderServiceLine__c` | `ListPrice__c`, `SalesPrice__c`, `NonUpliftSalesPrice__c`, `UpliftPrice__c`, `TierPrice__c` |
| `kugo2p__AccountPricing__c` | `Price__c` |
| `kugo2p__Tier__c` | `TierPrice__c` |
| `Asset` | `kugo2p__PurchasePrice__c` |

**Formula (Currency) fields** that inherit 6-decimal precision from the stored fields they reference — these render at 6 decimals end-to-end:

| Object | Formula fields rendering at 6 decimals |
|---|---|
| `kugo2p__SalesQuoteProductLine__c`, `kugo2p__SalesQuoteServiceLine__c` | `kugo2p__DiscountSalesPrice__c` (label: "Effective Price") |
| `kugo2p__SalesOrderProductLine__c`, `kugo2p__SalesOrderServiceLine__c` | `kugo2p__DiscountSalesPrice__c` (label: "Effective Price") |
| `kugo2p__KugamonInvoiceLine__c` | `kugo2p__SalesPrice__c` (yes, invoice line's SalesPrice is itself a formula reading back from the Order line), `kugo2p__DiscountSalesPrice__c` ("Effective Price"), `kugo2p__LineAmount__c`, `kugo2p__NetAmount__c`, `kugo2p__TotalAmount__c`, `kugo2p__TaxAmount__c`, `kugo2p__VATAmount__c`, `kugo2p__LineDiscountAmount__c`, `kugo2p__BalanceDueAmount__c` |

**Fields NOT widened** (stay at 2 decimals or lower-precision stored type):
- `kugo2p__KugamonInvoiceLine__c` **stored** currency fields (e.g. `kugo2p__AppliedPaymentAmount__c` at `Currency(16, 2)`). Invoice-line formulas above DO render at 6 decimals; the stored fields that back payments do not.
- All payment / applied-payment amounts (`kugo2p__PaymentX__c`, `kugo2p__AppliedPayment__c`).
- `PricebookEntry.UnitPrice`, `OpportunityLineItem.UnitPrice` (standard Salesforce 2-decimal).
- Opportunity-level roll-ups (`kuga_sub__Amount__c`, `kuga_sub__MonthlyRecurringRevenue__c`, etc.).

### Chat Objects (v11.0+)

#### `kugo2p__ChatParticipant__c` — one row per authorized chatter per quote

| Field API Name | Type | Updateable | Description |
|---|---|---|---|
| `Name` | String | No | AutoNumber: `CP-{0000000}` (e.g. `CP-0000070`). |
| `OwnerId` | Reference | Yes | Standard Owner. |
| `kugo2p__QuoteNumber__c` | Lookup(SalesQuote) | Yes | The quote this participant can chat on. |
| `kugo2p__User__c` | Lookup(User) | Yes | Populated when `Type__c = 'Internal User'`. |
| `kugo2p__Contact__c` | Lookup(Contact) | Yes | Populated when `Type__c = 'Contact'`. |
| `kugo2p__Type__c` | String (picklist) | No | `Internal User` or `Contact`. |
| `kugo2p__Participant__c` | String | No | HTML display link to the User or Contact record. |
| `kugo2p__ParticipantName__c` | String | No | Display name, denormalized. |
| `kugo2p__ParticipantKey__c` | String | Yes | Composite key `{QuoteId}-{UserOrContactId}` — enforces one participant row per person per quote. |
| `kugo2p__Email__c` | String | No | Email of the User or Contact. |
| `kugo2p__Title__c` | String | No | Business title. |
| `kugo2p__LastReadMessage__c` | Lookup(ChatMessage) | Yes | The last message this participant has seen. Powers unread badge. |
| `kugo2p__DateLastRead__c` | DateTime | Yes | When this participant last viewed the thread. |

#### `kugo2p__ChatMessage__c` — one row per posted message

| Field API Name | Type | Updateable | Description |
|---|---|---|---|
| `Name` | String | No | AutoNumber: 9-digit zero-padded sequence (e.g. `000000165`). |
| `kugo2p__ChatParticipant__c` | Lookup(ChatParticipant) | No | The sender's participant row — reach the Quote and sender identity through this. |
| `kugo2p__ParticipantName__c` | String | No | Sender display name, denormalized. |
| `kugo2p__Body__c` | TextArea (rich) | Yes | HTML message body as typed. |
| `kugo2p__BodyPreview__c` | String | Yes | Plain-text truncated preview, used in list views and notifications. |

---

## Appendix B: Amount Fields Guide

### 🔑 Pipeline Forecasting Rule (read first)

When `HAS_KUGA_SUB = true`, use `kuga_sub__Amount__c` for all Opportunity pipeline forecasting. DO NOT use the standard `Amount` field.

### The Amount Field Problem

In subscription orgs, the standard `Amount` field on Opportunity can represent MRR, ARR, TCV, or ACV — varies by org. Use `kuga_sub__Amount__c` instead.

### Comparison Rules

When `HAS_KUGA_SUB = true`: compare `kugo2p__TotalAmount__c` (quote) to `kuga_sub__AnnualContractValueInitial__c` or `kuga_sub__TotalContractValue__c`, not to `Amount`.

When `HAS_KUGA_SUB = false`: compare `kugo2p__TotalAmount__c` to `Amount`.

### Best Practices

1. For pipeline forecasting in subscription orgs, always use `kuga_sub__Amount__c`.
2. Query ALL amount fields before comparing.
3. Show all relevant amounts in user-facing summaries.
4. Never assume what standard `Amount` represents in a subscription org.

---

## Appendix C: Record Types Guide

NEVER hardcode Record Type IDs. Query dynamically from `RecordType` filtered by `SObjectType IN (kugo2p__SalesQuote__c, kugo2p__SalesOrder__c, kugo2p__Payment_Profile__c, kugo2p__Processor_Connection__c, kugo2p__Payment_Method__c, Opportunity)`.

Map by name: Opportunity "Renewal" → Quote/Order "Renewal"; "New" → "New Business"; "Expansion" → "Expansion". Default to "New Business" if no match.

---

## Appendix D: Renew Field Guide

`kuga_sub__Renew__c` on OpportunityLineItem is critical for revenue classification.

- `Renew = true` → recurring: revenue flows to `kuga_sub__MRR__c` and `kuga_sub__ARR__c`; `kuga_sub__NonRecurringRevenue__c = 0`.
- `Renew = false` or null → one-time: revenue flows to `kuga_sub__NonRecurringRevenue__c`; MRR and ARR remain 0.

Opportunity roll-ups: `kuga_sub__MonthlyRecurringRevenue__c`, `kuga_sub__AnnualRecurringRevenueCommitted__c`, `kuga_sub__NonRecurringRevenue__c`. Formula: `kuga_sub__AnnualContractValueInitial__c = NonRecurring + ARR` (double-counts if Renew is wrong).

**Double-counting fix**: find products with `kuga_sub__Renew__c = false` AND `kuga_sub__ARR__c > 0` AND `kuga_sub__NonRecurringRevenue__c > 0`; set Renew = true.

**Product type guide**: Subscriptions / Support Contracts / Recurring Retainers = `true`. Hardware / Implementation / Pro Services / One-time Licenses = `false`.

**Technical detail**: Kugamon automation calculates MRR/ARR from pricing + term, then sets NonRecurringRevenue based on Renew. Writing to `kuga_sub__NonRecurringRevenue__c` directly is overridden — Renew is the source of truth.

---

## Appendix E: Name Field & Sample Data Conventions

Most key transactional objects use **Auto Number** on `Name` — Salesforce assigns the value on insert; the field is not createable or updateable; the format is fixed by the package. Before authoring a sample record, run `get_object_fields` and read the `Name` field's `DataType`.

### Group A: Auto Number, date-stamped prefix — DO NOT set Name

| Object | Format | Example |
|---|---|---|
| `kugo2p__SalesQuote__c` | `SQ-{YYMMDD}-{0000000}` | `SQ-260519-0020461` |
| `kugo2p__SalesOrder__c` | `SO-{YYMMDD}-{0000000}` | `SO-260521-0113570` |
| `kugo2p__KugamonInvoice__c` | `INV-{YYMMDD}-{0000000}` | `INV-260505-0091615` |

The `YYMMDD` portion is the org-local creation date — cannot be back-dated.

### Group B: Auto Number, sequence only — DO NOT set Name

| Object | Example |
|---|---|
| `kugo2p__SalesQuoteProductLine__c` | `0098308` |
| `kugo2p__SalesQuoteServiceLine__c` | `0043468` |
| `kugo2p__SalesQuoteAdditionalChargeCredit__c` | `0009725` |
| `kugo2p__SalesQuoteOptionalLine__c` | `0003452` |
| `kugo2p__SalesOrderProductLine__c` | `0167453` |
| `kugo2p__SalesOrderServiceLine__c` | `0098030` |
| `kugo2p__SalesOrderAdditionalChargeCredit__c` | `0050596` |
| `kugo2p__KugamonInvoiceLine__c` | `0186531` |
| `kugo2p__KugamonInvoiceAdditionalChargeCredit__c` | `0008382` |
| `kugo2p__OrderInvoiceRelationship__c` | `0077818` |
| `kugo2p__Shipment__c` | `0094394` |
| `kugo2p__ShipmentLine__c` | `0094394` |
| `kugo2p__AppliedPayment__c` | `0031357` |
| `kugo2p__AdditionalProductDetail__c` | `0024014` |
| `kugo2p__ProductCost__c` | `0000000` |
| `kugo2p__ConfigurationOption__c` (uses `CO-` prefix) | `CO-0000003` |
| `kugo2p__ChatMessage__c` (v11.0+) | `000000165` |
| `kugo2p__ChatParticipant__c` (v11.0+, uses `CP-` prefix) | `CP-0000070` |

### Group C: Text(80) populated by Kugamon Apex — DO NOT set Name

Field is technically writeable, but Kugamon trigger logic overwrites on insert/update:

| Object | What Apex writes into Name | Example |
|---|---|---|
| `kugo2p__PaymentX__c` | `Payment for Order <SO#>` or `Payment for Invoice <INV#>` | `Payment for Order SO-260325-0113546` |
| `kugo2p__Payment_Method__c` | `<Card Brand> (<last 4>)` from the tokenized card | `Visa (4242)` |
| `kugo2p__AdditionalAccountDetail__c` | Mirrors related `Account.Name` | `Starbucks Corporation` |

### Group D: Text(80), user-supplied — DO set a meaningful Name

Line groups, invoice schedules, payment profiles, additional charges/credits (reusable), product catalogs/categories, tiers, tiered pricing, carriers, warehouses, tax locations, VAT, service delivery schedules, processor connections. Match the conventions already in the org.

### Common mistakes to avoid

- Don't invent your own prefix (`Q-…`, `QT-…`, `ORD-…`, `IN-…`, `INV2-…`, `SQ-0001` without a date). Quote/Order/Invoice prefixes are fixed: `SQ`, `SO`, `INV`. The date is `YYMMDD` of actual creation. Sequence is 7 digits.
- Don't supply `Name` on AutoNumber objects — Salesforce silently drops it.
- Don't supply `Name` on Group C objects expecting it to stick — it's overwritten.
- Verify Name behavior with `get_object_fields` first.
