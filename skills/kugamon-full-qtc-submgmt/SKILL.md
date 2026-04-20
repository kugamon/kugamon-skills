---
name: kugamon-full-qtc-submgmt
description: Manage the full Kugamon Quote-to-Cash lifecycle in Salesforce — opportunities, quotes, orders, invoices, payments, shipments, and assets, which uses the kugo2p namespace (Kugamon Quote to Cash). And optionally Managed the full Kugamon Subscription Billing lifecycle  in Salesforce - opportunities, quotes, orders, invoices, payments, shipments, contracts, subscriptions, and assets, which requires the kuga_sub namespace (Kugamon Subscription Management). Detects which packages are installed and adapts accordingly. Use when users request operations on any Kugamon object.
version: 0.1.0
status: Beta
---

# Kugamon Use Cases:

Full lifecycle skill for Kugamon RevOps for Salesforce functionality

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

## Org-Specific Setup

**NEVER hardcode Record Type IDs.** Always query dynamically:

```sql
SELECT Id, Name, SObjectType, DeveloperName
FROM RecordType
WHERE SObjectType IN (
  'kugo2p__SalesQuote__c', 'kugo2p__SalesOrder__c',
  'kugo2p__Payment_Profile__c', 'kugo2p__Processor_Connection__c',
  'kugo2p__Payment_Method__c', 'Opportunity'
)
AND IsActive = true
ORDER BY SObjectType, Name
```

Cache results for the session. Map by name: Opportunity "New" → Quote "New" → Order "New", etc.

---

## Object Model Overview

### kugo2p Objects (Kugamon CPQ — ~50 custom objects)

**Quote Stage:**
- `kugo2p__SalesQuote__c` (~95 fields) — master quote
- `kugo2p__SalesQuoteServiceLine__c` (~89 fields) — recurring service lines
- `kugo2p__SalesQuoteProductLine__c` (~76 fields) — one-time product lines
- `kugo2p__SalesQuoteOptionalLine__c` (~16 fields) — optional upsell lines
- `kugo2p__SalesQuoteAdditionalChargeCredit__c` (~34 fields) — surcharges/discounts
- `kugo2p__QuoteLineGroup__c` (~14 fields) — line grouping

**Order Stage:**
- `kugo2p__SalesOrder__c` (~110 fields) — master order
- `kugo2p__SalesOrderServiceLine__c` (~108 fields) — recurring service order lines
- `kugo2p__SalesOrderProductLine__c` (~105 fields) — one-time product order lines
- `kugo2p__SalesOrderAdditionalChargeCredit__c` (~39 fields) — surcharges/discounts
- `kugo2p__OrderLineGroup__c` (~13 fields) — line grouping

**Invoice Stage:**
- `kugo2p__KugamonInvoice__c` (~73 fields) — invoice
- `kugo2p__KugamonInvoiceLine__c` (~47 fields) — invoice line items
- `kugo2p__KugamonInvoiceAdditionalChargeCredit__c` (~39 fields) — invoice adjustments
- `kugo2p__OrderInvoiceRelationship__c` (~15 fields) — order-to-invoice link
- `kugo2p__InvoiceSchedule__c` (~15 fields) — recurring invoice generation

**Payment Stage:**
- `kugo2p__PaymentX__c` (~71 fields) — payment records
- `kugo2p__AppliedPayment__c` (~22 fields) — payment-to-invoice allocation
- `kugo2p__Processor_Connection__c` (~52 fields) — gateway configs (Stripe, AuthNet, PayPal, eWay)
- `kugo2p__Payment_Method__c` (~32 fields) — payment method definitions
- `kugo2p__Payment_Profile__c` (~57 fields) — customer payment profiles

**Fulfillment Stage:**
- `kugo2p__Shipment__c` (~31 fields) — shipment records
- `kugo2p__ShipmentLine__c` (~27 fields) — shipment line items
- `kugo2p__ServiceDeliverySchedule__c` (~30 fields) — service delivery tracking
- `kugo2p__Carrier__c` (~11 fields) — shipping carriers
- `kugo2p__Warehouse__c` (~17 fields) — warehouse/inventory locations

**Product & Pricing:**
- `kugo2p__AdditionalProductDetail__c` (~64 fields) — extended product metadata (Service flag, weight, dimensions, tax, etc.)
- `kugo2p__AccountPricing__c` (~26 fields) — customer-specific pricing overrides
- `kugo2p__TieredPricing__c` (~16 fields) — volume/tiered pricing headers
- `kugo2p__Tier__c` (~12 fields) — individual tier definitions
- `kugo2p__ProductCost__c` (~11 fields) — product cost tracking
- `kugo2p__AdditionalChargeCredit__c` (~29 fields) — reusable charge/credit templates
- `kugo2p__ProductCatalog__c` (~19 fields) — product catalog
- `kugo2p__ProductCategory__c` — product categories
- `kugo2p__ProductCategoryProduct__c` — category-product junction

**Configuration & Bundles:**
- `kugo2p__ConfigurationGroup__c` (~16 fields) — product configuration groups
- `kugo2p__ConfigurationOption__c` (~27 fields) — configuration options
- `kugo2p__KitBundleMember__c` (~15 fields) — kit/bundle components

**Tax:**
- `kugo2p__TaxLocation__c` (~15 fields) — US tax jurisdictions
- `kugo2p__TaxRate__c` (~13 fields) — US tax rates
- `kugo2p__VAT__c` (~12 fields) — international VAT definitions
- `kugo2p__VATRate__c` (~15 fields) — VAT rates

**Account:**
- `kugo2p__AdditionalAccountDetail__c` (~44 fields) — extended account metadata

**Settings:**
- `kugo2p__KugamonSetting__c` (~78 fields) — master org-wide settings
- `kugo2p__Settings__c` (~19 fields) — additional settings

**Utility:**
- `kugo2p__Favorite__c`, `kugo2p__FavoriteMember__c`, `kugo2p__FavoriteShare__c`
- `kugo2p__Shopping_Cart_Item__c` (~21 fields)

### kuga_sub Objects (Kugamon Subscriptions — only when HAS_KUGA_SUB = true)

**Custom Objects:**
- `kuga_sub__Subscription__c` (34 fields) — the subscription record linking orders to contracts

**Fields added to standard objects by kuga_sub:**

On **Product2** (4 fields):
- `kuga_sub__Renewable__c` (Checkbox) — product is renewable (generates subscription on order release)
- `kuga_sub__RenewalProduct__c` (Lookup Product) — substitute product for renewal quotes
- `kuga_sub__Track__c` (Checkbox) — track as asset on order release
- `kuga_sub__UpliftRenewalPrice__c` (Checkbox) — apply price uplift percentage on renewal

On **OpportunityLineItem** (18 fields):
- `kuga_sub__Renew__c` (Checkbox) — **CRITICAL**: marks line as recurring vs. one-time
- `kuga_sub__ARR__c`, `kuga_sub__MRR__c`, `kuga_sub__NonRecurringRevenue__c` — calculated revenue
- `kuga_sub__ServiceTerm__c`, `kuga_sub__UnitofTerm__c` — term length and unit
- `kuga_sub__DateServiceEnd__c` — service end date
- `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c`, `kuga_sub__ListAmount__c` — amounts
- `kuga_sub__Service__c` (Formula) — whether line is a service
- `kuga_sub__ARRForecast__c`, `kuga_sub__LineTerm__c` — forecasting
- `kuga_sub__DiscountSalesPrice__c`, `kuga_sub__EffectiveDiscount__c` — discount tracking
- `kuga_sub__NonUpliftSalesPrice__c`, `kuga_sub__UpliftRenewalPrice__c` — renewal pricing
- `kuga_sub__ServiceTermBehavior__c` — term behavior picklist

On **Opportunity** (19 fields):
- `kuga_sub__MonthlyRecurringRevenue__c` (Roll-Up SUM)
- `kuga_sub__AnnualRecurringRevenueCommitted__c` (Roll-Up SUM)
- `kuga_sub__NonRecurringRevenue__c` (Roll-Up SUM)
- `kuga_sub__AnnualContractValueInitial__c` (Formula: NonRecurring + ARR)
- `kuga_sub__TotalContractValue__c` (Formula)
- `kuga_sub__Amount__c` (Roll-Up SUM)
- `kuga_sub__AnnualRecurringRevenueForecast__c`, `kuga_sub__ExpectedRevenue__c`, `kuga_sub__OpportunityAmount__c` (Formulas)
- `kuga_sub__ContractEndDate__c`, `kuga_sub__ParentContractEndDate__c` (Formula Date)
- `kuga_sub__DateRequired__c` (Roll-Up MIN), `kuga_sub__ServiceDateExpires__c` (Roll-Up MAX)
- `kuga_sub__ParentContract__c` (Lookup Contract), `kuga_sub__ParentOrder__c` (Lookup Order)
- `kuga_sub__AutoEmailRenewalOrder__c` (Checkbox), `kuga_sub__AutoRenewedOrder__c` (Lookup Order)
- `kuga_sub__RenewalOrderAutoCreationDate__c` (Date), `kuga_sub__RenewalPriceUpliftPercent__c` (Percent)

On **kugo2p__SalesOrder__c** (16 fields — **ORDER RELEASE controls**):
- `kuga_sub__GenerateContract__c` (Checkbox) — create Contract on release
- `kuga_sub__GenerateAsset__c` (Checkbox) — create Assets on release
- `kuga_sub__GenerateSubscription__c` (Checkbox) — create Subscriptions on release
- `kuga_sub__GenerateRenewalOpportunity__c` (Checkbox) — create Renewal Opportunity on release
- `kuga_sub__ContractNumber__c` (Lookup Contract), `kuga_sub__ParentContract__c` (Lookup)
- `kuga_sub__RenewalOpportunity__c` (Lookup Opportunity)
- `kuga_sub__ContractEndDate__c`, `kuga_sub__ParentContractEndDate__c`, `kuga_sub__RenewalEndDate__c` (Formulas)
- `kuga_sub__RenewableProductsCount__c`, `kuga_sub__RenewableServicesCount__c` (Roll-Ups)
- `kuga_sub__TrackableProductsCount__c`, `kuga_sub__TrackableServicesCount__c` (Roll-Ups)
- `kuga_sub__ServiceDateExpires__c` (Roll-Up MAX)
- `kuga_sub__UpdateContractContacts__c` (Multi-Select Picklist)

On **kugo2p__SalesQuote__c** (2 fields):
- `kuga_sub__ContractEndDate__c` (Formula), `kuga_sub__ContractNumber__c` (Lookup Contract)

On **kugo2p__SalesOrderServiceLine__c** (2 fields):
- `kuga_sub__Renew__c` (Checkbox) — marks as renewable
- `kuga_sub__Track__c` (Checkbox) — marks for asset tracking

On **kugo2p__SalesOrderProductLine__c** (2 fields):
- `kuga_sub__Renew__c` (Checkbox), `kuga_sub__Track__c` (Checkbox)

On **kugo2p__SalesQuoteServiceLine__c** (1 field):
- `kuga_sub__Renew__c` (Checkbox)

On **kugo2p__SalesQuoteProductLine__c** (1 field):
- `kuga_sub__Renew__c` (Checkbox)

On **Contract** (21 fields):
- `kuga_sub__AnnualRecurringRevenue__c`, `kuga_sub__MonthlyRecurringRevenue__c` (Roll-Up SUM Subscription)
- `kuga_sub__TotalSubscriptionAmount__c`, `kuga_sub__TotalSubscriptionCount__c`, `kuga_sub__TotalSubscriptionQuantity__c` (Roll-Ups)
- `kuga_sub__SubscriptionStartDate__c` (Roll-Up MIN), `kuga_sub__SubscriptionEndDate__c` (Roll-Up MAX)
- `kuga_sub__AnnualRecurringRevenueForecast__c` (Formula)
- `kuga_sub__Effective__c` (Formula Checkbox) — is contract currently active
- `kuga_sub__Expanded__c` (Checkbox) — has been expanded
- `kuga_sub__ContractRenewalNoticeDate__c` (Formula), `kuga_sub__SendRenewalNoticeToday__c` (Formula)
- `kuga_sub__LastRenewalNoticeSentDate__c` (Date)
- `kuga_sub__AutoEmailRenewalNotice__c`, `kuga_sub__AutoEmailRenewalOrder__c` (Checkboxes)
- `kuga_sub__RenewalOpportunity__c` (Lookup Opportunity), `kuga_sub__RenewalTerm__c` (Number)
- `kuga_sub__Pricebook2Id__c` (Lookup Pricebook)
- `kuga_sub__ContactBuying__c`, `kuga_sub__ContactBilling__c`, `kuga_sub__ContactShipping__c` (Lookups)

On **Asset** (4 fields):
- `kuga_sub__ContractNumber__c` (Lookup Contract)
- `kuga_sub__ParentSubscription__c` (Lookup Subscription)
- `kuga_sub__ParentLine__c` (Formula)
- `kuga_sub__Renew__c` (Formula Checkbox)

---

## Apex Triggers

### kugo2p Triggers (34)

| Trigger | Object | Purpose |
|---------|--------|---------|
| SalesQuoteTrigger | SalesQuote__c | Quote lifecycle (status, totals, numbering) |
| SalesQuoteServiceLineTrigger | SalesQuoteServiceLine__c | Service line calculations |
| SalesQuoteProductLineTrigger | SalesQuoteProductLine__c | Product line calculations |
| SalesQuoteOptionalLineTrigger | SalesQuoteOptionalLine__c | Optional line handling |
| SalesQuoteACCTrigger | SalesQuoteAdditionalChargeCredit__c | Quote charge/credit calcs |
| SalesOrderTrigger | SalesOrder__c | Order lifecycle (status, totals, invoice gen) |
| SalesOrderServiceLineTrigger | SalesOrderServiceLine__c | Service order line calcs |
| SalesOrderProductLineTrigger | SalesOrderProductLine__c | Product order line calcs |
| SalesOrderACCTrigger | SalesOrderAdditionalChargeCredit__c | Order charge/credit calcs |
| InvoiceTrigger | KugamonInvoice__c | Invoice lifecycle |
| InvoiceLineTrigger | KugamonInvoiceLine__c | Invoice line calcs |
| InvoiceACCTrigger | KugamonInvoiceAdditionalChargeCredit__c | Invoice charge/credit calcs |
| PaymentXTrigger | PaymentX__c | Payment processing |
| AppliedPaymentTrigger | AppliedPayment__c | Payment-to-invoice allocation |
| PaymentMethodTrigger | Payment_Method__c | Payment method validation |
| PaymentProfileTrigger | Payment_Profile__c | Profile management |
| PaymentSettingTrigger | Settings__c | Payment settings validation |
| ProcessorConnectionTrigger | Processor_Connection__c | Processor connection mgmt |
| ShipmentTrigger | Shipment__c | Shipment lifecycle |
| ShipmentLineTrigger | ShipmentLine__c | Shipment line tracking |
| ServiceDeliveryScheduleTrigger | ServiceDeliverySchedule__c | Service delivery tracking |
| OpportunityTrigger | Opportunity | Opp-to-Kugamon sync |
| AccountTrigger | Account | Account data sync |
| AdditionalAccountDetailTrigger | AdditionalAccountDetail__c | Account metadata sync |
| AdditionalProductDetailTrigger | AdditionalProductDetail__c | Product metadata sync |
| Product2Trigger | Product2 | Product sync to AdditionalProductDetail |
| AccountPricingTrigger | AccountPricing__c | Customer pricing validation |
| ProductCostTrigger | ProductCost__c | Cost tracking |
| ProductCatalogTrigger | ProductCatalog__c | Catalog management |
| ProductCategoryTrigger | ProductCategory__c | Category management |
| ConfigurationGroupTrigger | ConfigurationGroup__c | Product configuration |
| KugamonSettingTrigger | KugamonSetting__c | Settings validation |
| LeadTrigger | Lead | Lead conversion handling |
| TaskTrigger | Task | Task automation |

### kuga_sub Triggers (14 — only when HAS_KUGA_SUB = true)

| Trigger | Object | Purpose |
|---------|--------|---------|
| Opportunities | Opportunity | Subscription revenue roll-ups, renewal opp linking |
| Quote | SalesQuote__c | Contract linking on renewal quotes |
| QuoteServiceLine | SalesQuoteServiceLine__c | Renew flag propagation to quote lines |
| QuoteProductLine | SalesQuoteProductLine__c | Renew flag propagation to quote lines |
| Order | SalesOrder__c | Order Release: generates Contract, Asset, Subscription, Renewal Opp |
| OrderServiceLine | SalesOrderServiceLine__c | Renew/Track flag handling, subscription creation |
| OrderProductLine | SalesOrderProductLine__c | Renew/Track flag handling, asset creation |
| Contracts | Contract | Subscription roll-ups, renewal notice scheduling |
| Asset | Asset | Contract/subscription linking |
| Subscription | Subscription__c | Subscription lifecycle management |
| Product | Product2 | Renewable/Track flag sync |
| AdditionalAccountDetail | AdditionalAccountDetail__c | Account subscription data sync |
| KugamonSetting | KugamonSetting__c | Subscription settings sync |
| ShipmentLine | ShipmentLine__c | Asset tracking on shipment |

---

## Apex Class Logic

### kuga_sub Architecture

All kuga_sub triggers use a **TriggerHandler** base class pattern with overridable methods: `beforeInsert`, `afterInsert`, `beforeUpdate`, `afterUpdate`, `beforeDelete`, `afterDelete`. Each trigger instantiates its handler and calls `handler.run()`.

**Central orchestration class:** `KugamonHelper` — contains all core business logic as static methods. Trigger handlers are thin dispatchers that call into KugamonHelper.

**Security:** All DML operations use `SecurityUtil.stripInaccessibleFromDML()` for FLS enforcement.

### Order Release Coordination

The most critical architectural pattern in kuga_sub. Three triggers (Order, ServiceLine, ProductLine) coordinate via static flags to ensure Order Release operations run exactly once regardless of trigger execution order.

**Static coordination flags on KugamonHelper:**
- `hasRenewableProducts` / `hasRenewableServices` — set by product/service line afterUpdate triggers
- `processedContract` / `processedSubscription` — prevent duplicate processing
- `processedProductLineAfterUpdateTrigger` / `processedServiceLineAfterUpdateTrigger` — track which line triggers have fired
- `mapNewContractOrders` — shared map of orders needing processing, populated by SalesOrderTriggerHandler

**Execution flow when Order status changes:**

1. **SalesOrderTriggerHandler.afterUpdate** detects status change, populates `mapNewContractOrders`, calls `createContract` → `createSubscription` → `createRenewalOpportunity`
2. Contract/Subscription creation updates order line items, which fires **ServiceLine** and **ProductLine** afterUpdate triggers
3. Line triggers check `mapNewContractOrders` — if populated and their counterpart has already fired, they call `createContract` → `createSubscription` → `createRenewalOpportunity` again
4. The `processedContract` / `processedSubscription` flags prevent duplicate execution

**Key setting:** `InitiateOrderSubscriptionManagement__c` on `kuga_sub__SubscriptionSetting__c` controls WHEN Order Release fires:
- `"Approve/Release"` — fires when order status changes to Approved AND Released
- `"Release"` — fires only when order status changes to Released

### KugamonHelper Key Methods

#### Order Release Methods

| Method | Purpose |
|--------|---------|
| `createContract(map<Id, SalesOrder__c>)` | Creates Contract from Order. Sets ContractTerm, StartDate, EndDate from order line dates. Copies contacts per `UpdateContractContacts__c` multi-select. If `ExtendContractonRenewal__c` = true for Renewal orders, extends existing contract instead of creating new one. Links contract back to order via `ContractNumber__c` |
| `createSubscription(map<Id, SalesOrder__c>)` | Creates Subscription records from order lines where `Renew__c = true`. Sets MRR, ARR, NetAmount, TotalAmount, ServiceTerm, dates. Links to Contract, Account, Order, Product. Also creates Assets from lines where `Track__c = true` |
| `createRenewalOpportunity(set<Id> orderIds)` | Creates Renewal Opportunity with matching RecordType. Copies line items from order to new opp as OpportunityLineItems. Sets `ParentContract__c` and `ParentOrder__c` on the renewal opp. Returns `List<OrderRenewalOpportunity>` wrapper |

#### Cancellation / Un-Release Methods

| Method | Purpose |
|--------|---------|
| `deActivateOrderContracts(set<Id>)` | When order is cancelled/un-released: deletes generated contracts, expires renewal opportunities (sets stage to "Closed Lost"), deactivates subscriptions |
| `unReleaseUpsellOrders(map<Id, Id>)` | Handles un-release of expansion/upsell orders — reverses quantity changes on parent subscriptions |
| `updateSubscriptionStatus(Set<Id>, String)` | Bulk updates subscription status for a set of order IDs |

#### Renewal and Pricing Methods

| Method | Purpose |
|--------|---------|
| `updateRenewalUpliftSalesPrice(map newOpps, map oldOpps)` | When a Renewal opportunity's `RenewalPriceUpliftPercent__c` changes, recalculates UnitPrice on all OLIs by applying uplift to `NonUpliftSalesPrice__c` |
| `updateRenewalOrderContractPriceBook(map newOrders, map oldOrders)` | When Renewal order's pricebook changes, syncs back to parent Contract's `Pricebook2Id__c` |
| `updateContractOpptyServiceTerm(map<Id, decimal>)` | Updates ServiceTerm on opportunity line items when contract renewal term changes |
| `deleteOLISchedule(set<Id>)` | Deletes OpportunityLineItemSchedule records when renewal pricing changes |

#### Line Item and Flag Propagation Methods

| Method | Purpose |
|--------|---------|
| `updateRenewandServiceTerm(list<SObject>, boolean isService)` | On order line beforeInsert: propagates `Renew__c` from quote line to order line. Sets ServiceTerm and UnitofTerm. If no quote line link, falls back to Product2's `Renewable__c` flag |
| `updateExpansionKitMemberServiceEndDate(list<ServiceLine>)` | For Expansion orders: adjusts service end dates on kit member lines to align with the parent kit line's end date |
| `setAssetDetails(list<Asset>)` | beforeInsert on Asset: links asset to Contract and Subscription via `ContractNumber__c` and `ParentSubscription__c` |
| `updateSubscriptionDetails(list<Asset>)` | afterInsert on Asset: updates the parent Subscription's `ParentAsset__c` to point back to the newly created asset |

#### Matching Utility

| Method | Purpose |
|--------|---------|
| `getOLIKey(orderId, productId, price, discount, description, configOptionId, startDate)` | Generates a composite key for matching Subscriptions to OpportunityLineItems. Used by SubscriptionTriggerHandler when cancelling subscriptions to find and reduce/delete corresponding renewal OLIs |

### Trigger Handler Behaviors

#### SalesOrderTriggerHandler
- **beforeInsert**: Sets RecordType from linked Quote or Opportunity. For Expansion/Renewal: copies `ContractNumber__c` from Quote. Calls `setOrderDetails` which auto-calculates `GenerateContract__c`, `GenerateAsset__c`, `GenerateSubscription__c`, `GenerateRenewalOpportunity__c` based on whether line items have Renew/Track flags
- **afterUpdate**: Detects Order status change → triggers Order Release chain (createContract → createSubscription → createRenewalOpportunity). On cancellation/un-release → calls `deActivateOrderContracts` to reverse all generated records

#### SalesQuoteTriggerHandler
- **beforeInsert**: Sets RecordType from linked Opportunity's RecordType. For Expansion/Renewal quotes when `ExtendContractonRenewal__c` is enabled: auto-sets `ContractNumber__c` and copies contacts (ContactBilling, ContactBuying, ContactShipping) from the Contract

#### ContractTriggerHandler
- **beforeInsert/Update**: Validates single active contract per account (unless `AllowMultipleActiveContracts__c` = true). Auto-calculates `ContractTerm` from StartDate and EndDate
- **beforeUpdate**: `setContactDetails` syncs billing/shipping addresses from Contact records to Contract address fields. Validates required address fields are populated
- **afterUpdate — Activation**: When contract activates, syncs `IsActive__c` on all child Subscriptions
- **afterUpdate — Cancellation**: When contract is cancelled, cancels all child Subscriptions (sets `Status__c = 'Cancelled'`) and closes the linked Renewal Opportunity (stage → "Closed Lost")
- **afterUpdate**: Syncs `AutoEmailRenewalOrder__c` flag changes to the linked Renewal Opportunity

#### SubscriptionTriggerHandler
- **beforeInsert/Update**: Syncs `IsActive__c` (editable) with `Active__c` (formula) to keep them aligned
- **afterUpdate — Cancellation**: When a subscription is cancelled, finds matching OLIs on the Renewal Opportunity using `getOLIKey`. Reduces quantity on the matching OLI by the subscription's quantity. If resulting quantity ≤ 0, deletes the OLI entirely

#### OpportunityTriggerHandler
- **beforeUpdate**: Validates currency match for Expansion/Renewal opportunities — the opp's CurrencyIsoCode must match the parent Contract's currency. Prevents currency mismatch errors
- **afterUpdate**: When `RenewalPriceUpliftPercent__c` changes on a Renewal opp, triggers `updateRenewalUpliftSalesPrice` to recalculate all line item prices

#### AssetTriggerHandler
- **beforeInsert**: Calls `KugamonHelper.setAssetDetails` — links asset to Contract and Subscription
- **afterInsert**: Calls `KugamonHelper.updateSubscriptionDetails` — sets `ParentAsset__c` on the subscription

#### SalesOrderServiceLineTriggerHandler
- **beforeInsert**: Sets ListPrice from PricebookEntry. Propagates `Track__c` from `AdditionalProductDetail.ReferenceProduct.Track__c`. Propagates `Renew__c` from linked quote service line. Calls `updateRenewandServiceTerm` and `updateExpansionKitMemberServiceEndDate`
- **afterUpdate**: Sets `processedServiceLineAfterUpdateTrigger = true`, then conditionally calls Order Release chain if `mapNewContractOrders` is populated

#### SalesOrderProductLineTriggerHandler
- **beforeInsert**: Same pattern as service line handler. `Track__c` from `AdditionalProductDetail.CreateAsset`. Propagates `Renew__c` from linked quote product line
- **afterUpdate**: Sets `processedProductLineAfterUpdateTrigger = true`, then conditionally calls Order Release chain

### Scheduled Batch Jobs

Three scheduleable batch classes handle automated lifecycle operations:

| Batch Class | Schedule | Purpose |
|-------------|----------|---------|
| `ContractRenewalNoticeBatcher` | Daily recommended | Sends renewal notice emails to contracts where `SendRenewalNoticeToday__c = true`. Uses email template from `SubscriptionSetting__c.ContractRenewalEmailTemplateName__c`. Updates `LastRenewalNoticeSentDate__c` after sending |
| `RenewalOrderBatcher` | Daily recommended | Creates Renewal Orders from Renewal Opportunities where RecordType = 'Renewal' and `RenewalOrderAutoCreationDate__c <= today`. Creates `kugo2p__SalesOrder__c` with Renewal record type, copies contacts from Contract (falls back to AdditionalAccountDetail), splits line items into service/product lines based on `Service__c` flag |
| `SubscriptionBatcher` | Periodic | Syncs `IsActive__c` (editable checkbox) with `Active__c` (formula) on Subscriptions where they have diverged. Safety net to keep these fields aligned |

### Key Settings That Drive Behavior

These fields on `kuga_sub__SubscriptionSetting__c` control critical behavior:

| Setting Field | Values | Effect |
|---------------|--------|--------|
| `InitiateOrderSubscriptionManagement__c` | "Approve/Release" or "Release" | Controls WHEN Order Release fires — on approval+release or release only |
| `ExtendContractonRenewal__c` | Checkbox | If true, Renewal orders extend existing contract end date instead of creating a new contract |
| `AllowMultipleActiveContracts__c` | Checkbox | If true, allows multiple active contracts per account. If false, ContractTriggerHandler enforces single active contract |
| `ContractRenewalEmailTemplateName__c` | Text | Email template API name for renewal notice emails sent by ContractRenewalNoticeBatcher |

### Renew/Track Flag Propagation Chain

This is the full propagation path for how Renew and Track flags flow through the Q2C lifecycle:

```
Product2.kuga_sub__Renewable__c  ──→  OpportunityLineItem.kuga_sub__Renew__c
Product2.kuga_sub__Track__c      ──→  (via AdditionalProductDetail)

OLI.kuga_sub__Renew__c  ──→  QuoteServiceLine.kuga_sub__Renew__c  (if Renew=true)
                         ──→  QuoteProductLine.kuga_sub__Renew__c  (if Renew=false)

QuoteLine.kuga_sub__Renew__c  ──→  OrderServiceLine.kuga_sub__Renew__c
                              ──→  OrderProductLine.kuga_sub__Renew__c

OrderLine.kuga_sub__Renew__c = true   ──→  Subscription created on Order Release
OrderLine.kuga_sub__Track__c = true   ──→  Asset created on Order Release
```

### kugo2p Apex Classes

Below is the architectural overview of key kugo2p Apex classes.

#### Core Architecture Patterns

**Kontroller** — Central action router for all Visualforce/LWC button actions. Key design:
- `Kontroller.logicPath` static variable: `'trigger'` (default) or `'controller'`. When set to `'controller'`, trigger handlers skip auto-fill logic (e.g., SalesQuoteHelper.fillSalesQuote) so the controller can manage field values directly. This prevents double-processing when records are created programmatically via buttons
- `Director()` method routes based on `action` parameter: `createSalesQuote`, `createSalesOrder`, `createInvoice`, `updateQuoteStatus`, `updateOrderStatus`, `updateInvoiceStatus`, `deleteInvoice`, `goToPaymentTerminal`, `attachPDF`, `onlineOrderEmail`, `onlineInvoiceEmail`, `emailPaymentPDF`, `createPaymentPDF`, `emailOrderPDF`, `emailQuotePDF`, `emailInvoicePDF`, `refreshAssets`, `refreshPayment`, `cloneSalesQuote` (from updateQuoteStatus Won flow)
- `ValidateAccountDetails(Id acctId)` — validates billing address and contact exist before quote/order creation. Called by trigger handlers too
- Order status flow: Draft → Sent → Approved → Released → Cancelled. Special statuses: `ApproveOrderandPay` (approve + immediate payment), `Unrelease`, `CancelApproved`, `CancelReleased`
- Quote Won flow: `updateQuoteStatusToWonAndGenerateOrder()` sets quote status to Won, then internally routes to `createSalesOrder` to auto-generate order

**KugamonSyncService** — Bidirectional sync between Quotes/Orders and OpportunityLineItems. Implements `Queueable` for async processing:
- `syncOpportunity(list<SObject>, map oldHeaders, String objType)` — static entry point called from quote/order afterUpdate triggers. Detects changes to IsPrimary, Opportunity, PriceBookName, RecordStatus, DiscountPercent. Only syncs if record is primary (`IsPrimary__c = true`) and linked to an Opportunity. Enqueues a Queueable job to process
- `syncOppLines(list<SObject>, map oldLines, String objType, boolean isService)` — static entry point called from quote/order line afterAll triggers. Detects changes to Quantity, SalesPrice, LineDiscountPercent, ServiceDate, LineDescription, SortOrder, OpportunityLineItemId, ParentProductLine, ParentServiceLine, ConfigurationOption. Also checks "twin fields" (custom mapped fields between line types). If `HAS_KUGA_SUB`, also monitors `kuga_sub__Renew__c`, `DateServiceEnd__c`, `UnitofTerm__c`
- `processKugamonLines()` — core sync method. Creates/upserts OpportunityLineItems from quote/order lines. Maps: Quantity, UnitPrice (calculated via getSalesPrice), Discount, ServiceDate, Description, SortOrder. Copies "twin fields" via `Util.copyFields`. If `HAS_KUGA_SUB`: syncs `Renew__c`, `DateServiceEnd__c`, `ServiceTerm__c`, `ServiceTermBehavior__c`, `UnitofTerm__c` to OLI
- `disableOpportunitySync` static boolean — can be set to skip sync entirely
- Lines with "Exclude from Opportunity Sync" flag are skipped (creates a Task notification)
- Lines with Quantity = 0 are skipped (OLI doesn't support zero quantity)
- Lines with inactive PricebookEntry are skipped

**Kugamon** — Central caching/retrieval layer providing get/clear/refresh patterns for all Q2C objects: Account, Contact, Opportunity, OLI, Product2, ProductDetail, KitBundleMembers, Pricebook, PricebookEntry, SalesQuote (with all child lines), SalesOrder (with all child lines), Shipment, Invoice, Payment, AppliedPayment, ServiceDeliverySchedule, RecordType, TaxRate, VAT, AdditionalChargeCredit, EmailTemplate. This class is the data access layer — all trigger handlers and helpers query through it for caching

**SecurityUtil** — All DML operations across the entire package use `SecurityUtil.stripInaccessibleFromDML()` for FLS enforcement. Delete operations check `SecurityUtil.checkObjectIsDeletable()`

#### Helper Classes

| Class | Key Methods |
|-------|-------------|
| SalesOrderHelper | `createSalesOrder` (7 overloads: from Account, Contact, Opportunity, Quote, Payment), `fillSalesOrder`, `createSalesOrderLines` (from Quote and Opportunity), `calculateServiceEndDate`, `fillKitMemberOrderLines`, `unReleaseOrder`, `cancelReleasedOrder`, `hasOrderInvoice`/`hasOrderInvoicePayments`/`hasOrderInvoicePosted`, `updateShipmentStatus`/`updateServiceDeliveryStatus`/`updateAssetStatus`/`updateOrderInvoiceStatus`/`updateOrderStatus`, `cloneSalesOrder`, `validate`, `assignPrimaryOrder`, `CreateAssets`/`UpdateAssets`/`DeleteAssets`, `handleOrderStatusUpdate`, `syncPriceBook`, `okayToUpdateReleasedOrder`, `checkProductSalesLinesSynced` |
| SalesQuoteHelper | `createSalesQuote`, `fillSalesQuote`, `fillSalesQuoteProductLine`/`fillSalesQuoteServiceLine` (multiple overloads with tiered pricing and kit bundles), `calculateServiceEndDate` (multiple overloads), `fillKitMemberQuoteLines`, `createSalesQuoteLines`, `getDateAvailableToPromise`, `cloneSalesQuote`, `assignPrimaryQuote`, `handleQuoteStatusUpdate`, `syncPriceBook` |
| GroupHelper | Inner classes `LineGroup` and `LineGroupMember`. `processProductOrServiceLine`/`processACCLine`/`processOptionalLine`, `createLine`, `prepareLine`/`prepare`, `upsertGroups`/`upsertLines`, `getDBGroups`, `createLineGroupMap`, `processKitBundleLine`, `assignKitBundleMembersToLines` |
| ProductHelper | `CreateProductDetail`, `MapProductDetail`, inner classes `ProductTileData`/`Tier`/`Subscription`, `createPricebookEntryMap`, `getCurrencyCode`, `buildProductRecords`/`buildAssetRecords`/`buildSubscriptionRecords`, `getAccountIdsByHierarchy`, `setProductAccountPricingFields`, `evaluateAccountPricingFilter`, `getPricebookTieredPricing`/`getProductTieredPrice`, `setProductsInContractPricebook`, `setFavoritedProducts`, `setProductCostFields`, `validateAPD` |
| InvoiceHelper | `createInvoiceSchedule`, `buildInvoices`, `createInvoice`, `updateInvoicedQuantities`, `updateInvoiceLineAmounts`, `fillInvoice`, `handleInvoiceStatusUpdate`, `getInvoicePOKey` |
| PaymentHelper | `createPaymentProfile`, `createInvoicePayment`/`createOrderPayment`/`createAccountPayment`, `matchKugamonPayment`, `applyInvoicePayments`/`applyOrderPayments`, `applyPaymentsToLines`, `deleteInProcessPayments` |
| AccountHelper | `MapAccountDetail` (creates/upserts AdditionalAccountDetail from Account), `updateAccountBalance_Batch`, `isPersonAccount` |
| Util | Type conversion, URL/string processing, record type lookup (`getRecordType`), field copy (`copyFields`), sort, multi-currency helpers, email, error handling, `getTwinFields` (custom field mapping between objects) |

#### kugo2p Trigger Handler Behaviors

##### SalesQuoteTriggerHandler
- **beforeInsert**: Validates pricebook (must have PBE entries matching quote currency). Assigns currency from Pricebook if multi-currency. Generates `OnlineApprovalKey__c` random string. Checks `Kontroller.logicPath` — if `'trigger'` (manual creation), calls `SalesQuoteHelper.fillSalesQuote` to auto-fill defaults
- **afterInsert**: Copies OpportunityLineItems to quote lines via `SalesQuoteHelper.createSalesQuoteLines`. Calls `SalesQuoteHelper.assignPrimaryQuote` to set IsPrimary
- **beforeUpdate**: Contact address sync — when ContactBilling/ContactShipping changes, copies Contact's Mailing address → BillTo fields, Other address → ShipTo fields. Discount cascade — when `DiscountPercent__c` changes, recalculates `LineDiscountAmount__c` on ALL child service and product lines. Currency enforcement — prevents currency change if child lines exist. Pricebook validation — prevents pricebook change if child lines exist
- **afterUpdate**: Cascades ContactShipping, Carrier, Warehouse, Opportunity changes to all child lines. Calls `KugamonSyncService.syncOpportunity` for opportunity sync. Calls `SalesQuoteHelper.handleQuoteStatusUpdate` on status changes

##### SalesQuoteServiceLineTriggerHandler
- **beforeInsert**: Fills kit bundle member details from KitBundleMember records. Assigns auto-incrementing SortOrder via aggregate MAX query. Validates pricebook (checks PBE exists for the product, including kit member validation). If `Kontroller.logicPath == 'trigger'` (manual creation): auto-fills ServiceName from APD, calculates ServiceEndDate from ServiceTerm, applies "% of Unit Price" logic, enforces currency match with parent quote
- **afterAll** (insert/update/delete/undelete): Creates kit bundle member lines (both service and product members from KBM records). Rolls up Tax, Discount, VAT amounts to quote header. Syncs kit header quantity changes to member lines. Calls `KugamonSyncService.syncOppLines` for opportunity sync
- **beforeDelete**: Prevents deletion of required kit bundle members. Cascade deletes child kit members

##### SalesQuoteProductLineTriggerHandler
- **beforeInsert**: Assigns SortOrder. Validates pricebook (including kit bundle member validation via APD/KBM queries). If `Kontroller.logicPath == 'trigger'`: fills product line defaults, enforces currency match
- **afterAll**: Creates kit bundle member lines (can create BOTH product AND service member lines from product header). Tax/Discount/VAT roll-up to quote header. Kit quantity sync. Calls `KugamonSyncService.syncOppLines`
- **beforeDelete**: Prevents required kit member deletion. Cascade deletes child products AND child services

**Cross-trigger coordination:** `TriggerHelper.passedPricebookValidation` static flag prevents duplicate pricebook validation when both product and service line triggers fire in the same transaction

##### SalesOrderTriggerHandler
- **beforeInsert**: Validates pricebook, assigns currency. Generates OnlineApprovalKey. If `logicPath == 'trigger'`: calls `SalesOrderHelper.fillSalesOrder`
- **beforeUpdate**: Contact address sync (same pattern as quote). Discount cascade. Currency/pricebook enforcement. Validates `okayToUpdateReleasedOrder` for released orders. Calls `SalesOrderHelper.handleOrderStatusUpdate`
- **afterInsert/afterUpdate**: Copies OLI/quote lines to order lines. Assigns primary order. Cascades ContactShipping/Carrier/Warehouse/Opportunity changes to child lines. Calls `KugamonSyncService.syncOpportunity`
- **beforeDelete/afterDelete**: Validates no invoices exist before deletion. Cleans up related records

##### SalesOrderServiceLineTriggerHandler
- **beforeInsert**: Fills kit bundle member details. Assigns SortOrder. Validates pricebook. If `logicPath == 'trigger'`: fills service name, calculates end date, applies pricing, enforces currency
- **afterAll**: Kit member creation, tax/discount/VAT roll-up, kit quantity sync, opp line sync via KugamonSyncService
- **beforeDelete**: Prevents required kit member deletion, cascade delete

##### SalesOrderProductLineTriggerHandler
- **beforeInsert**: Assigns SortOrder. Validates pricebook. If `logicPath == 'trigger'`: fills product defaults, enforces currency
- **afterAll**: Kit member creation (both product and service members), roll-ups, kit quantity sync, opp line sync
- **beforeDelete**: Prevents required kit member deletion, cascade delete

##### OpportunityTriggerHandler
- **beforeInsert**: `setOpportunityPricebook` — if Opportunity has no Pricebook but has an Account with AdditionalAccountDetail, auto-assigns the pricebook from AAD.PricebookName
- **afterUpdate**: `handleClosedLostOpportunity` — when opportunity stage changes to a "Closed Lost" stage name (configurable via `Kugamon.closedLostOppStageNames`): if `AutoClosedLostQuote__c` setting is true, sets all Draft/Sent quotes to "Lost". If `AutoCancelOrder__c` setting is true, sets all Draft/Sent orders to "Cancelled"

##### AccountTriggerHandler
- **afterInsert/afterUpdate**: `updateAdditionalAccountDetails` — auto-creates or updates `AdditionalAccountDetail__c` (AAD) record for the account. Syncs Account.Name → AAD.Name. For Person Accounts: sets AAD.Name to FirstName + LastName. Syncs CurrencyIsoCode changes. Uses `AccountHelper.hasTriggerExecuted` static flag to prevent recursion

##### Product2TriggerHandler
- **afterInsert/afterUpdate**: `updateAPDs` — auto-creates or updates `AdditionalProductDetail__c` (APD) from Product2. Syncs IsActive, Name, ProductCode, Description, Family, QuantityUnitOfMeasure, CurrencyIsoCode. Uses `TriggerHelper.productUpdated` to prevent recursion with APD trigger
- **beforeUpdate**: Validates Kit/Bundle integrity — blocks product deactivation if product is a member of an active Kit/Bundle. Blocks product activation if it's a Kit/Bundle header with inactive members
- **beforeDelete**: Cascade deletes APD and TieredPricing records. If product has quote/order/invoice references, blocks deletion with error suggesting deactivation instead

##### AdditionalProductDetailTriggerHandler
- **beforeInsert/beforeDelete/afterUndelete**: `validateDuplicateAPD` — enforces exactly one APD per Product2. Blocks duplicate creation, blocks deletion if it's the only APD, blocks undelete if another APD already exists
- **beforeUpdate**: Syncs `ConfigurationMethod__c == 'Kit/Bundle'` → sets `KitBundle__c = true`. Validates Kit/Bundle integrity (same active/inactive member checks as Product2). When changing `Service__c` from false to true on a Kit/Bundle, deletes product-type KitBundleMembers (service kits can't have product members). Calls `ProductHelper.validateAPD`
- **afterUpdate**: Syncs APD field changes back to Product2 (IsActive, Name, ProductCode, Description, Family, UnitOfMeasure, CurrencyIsoCode). Uses `TriggerHelper.productUpdated` to prevent recursion

##### InvoiceTriggerHandler
- **beforeInsert**: Sets `DatePosted__c` if `IsPosted__c` is true. Generates `OnlineApprovalKey__c`. Validates `AddOnlinePaymentDetailsinPDF__c` against checkout configuration. Links to AdditionalAccountDetail
- **beforeUpdate**: Calls `InvoiceHelper.handleInvoiceStatusUpdate`. Recalculates `InvoiceDueDate__c` when `InvoiceDate__c` changes (adds `DaysTillPaymentDue__c` from AAD). Contact address sync (BillTo from MailingAddress, ShipTo from OtherAddress). Multi-currency enforcement — blocks currency change if child lines exist. Updates AAD.ContactBilling if previously null
- **afterInsert/afterUpdate**: Cascades invoice RecordStatus changes to all child InvoiceLine records. Updates Account balance via `AccountHelper.updateAccountBalance_Batch` when Account, AAD, BalanceDueAmount, or RecordStatus changes
- **afterDelete**: Updates Account balance for deleted invoice's account

##### InvoiceLineTriggerHandler
- **beforeInsert**: Assigns currency from parent invoice. Assigns auto-incrementing SortOrder
- **beforeUpdate**: Assigns currency from parent invoice
- **afterAll** (insert/update/delete/undelete): Updates invoiced quantities on order lines via `InvoiceHelper.updateInvoicedQuantities`. Rolls up line amounts to invoice header via `InvoiceHelper.updateInvoiceLineAmounts`. On insert: deletes disabled shipment lines/shipments
- **beforeDelete**: Blocks deletion if invoice line is assigned to an Order or Order Line (unless `InvoiceHelper.ignoreInvoiceLineDeleteValidation` is set)

### ManageContractController (LWC Controller)

The only kuga_sub Apex controller with `@AuraEnabled` methods:

| Method | Purpose |
|--------|---------|
| `getActiveContracts(accountId)` | Retrieves active contracts with child subscriptions and assets. Builds chart data for contract visualization. Calculates MRR trending by comparing renewal opp MRR vs contract MRR → returns up/down/neutral indicator |

---

## Workflow 1: Quote Creation

### Step 0: Create Opportunity and Line Items (If Needed)

#### Creating the Opportunity

**Required Fields:**
- `Name` — e.g., "Acme Corp - Annual Subscription"
- `StageName` — "Qualification" by default
- `CloseDate` — Expected close date
- `AccountId` — Required for Kugamon
- `Pricebook2Id` — Required if adding products

**Optional:** `Amount`, `Type`, `RecordTypeId` (match to quote type)

#### Creating Opportunity Line Items

**Always use `PricebookEntryId`** (not Product2Id).

##### If HAS_KUGA_SUB = true:

**CRITICAL:** Always set `kuga_sub__Renew__c`:
- `true` → recurring (subscriptions, support) → revenue flows to MRR/ARR
- `false` → one-time (hardware, implementation) → revenue flows to NonRecurringRevenue

```javascript
// Recurring
{ "OpportunityId": "006xxx", "PricebookEntryId": "01uxxx", "Quantity": 1,
  "UnitPrice": 2000, "kuga_sub__Renew__c": true,
  "kuga_sub__ServiceTerm__c": 12, "kuga_sub__UnitofTerm__c": "Month" }

// One-time
{ "OpportunityId": "006xxx", "PricebookEntryId": "01uxxx", "Quantity": 1,
  "UnitPrice": 15000, "kuga_sub__Renew__c": false }
```

##### If HAS_KUGA_SUB = false:

No Renew field. Line separation uses `kugo2p__AdditionalProductDetail__c.kugo2p__Service__c`:
- `Service = true` → Quote Service Lines
- `Service = false` → Quote Product Lines

### Step 1: Pre-Creation Validation

**Billing Address:**
```sql
SELECT Id, Name, BillingStreet, BillingCity, BillingState, BillingPostalCode, BillingCountry
FROM Account WHERE Id = '<account_id>'
```
All billing fields must be populated. If missing, ask user and update account.

**Contact:**
```sql
SELECT Id, Name, Email, Phone FROM Contact
WHERE AccountId = '<account_id>' AND Email != null LIMIT 10
```
At least one contact required. Ask which one for the quote.

**Opportunity:**

If HAS_KUGA_SUB = true:
```sql
SELECT Id, Name, AccountId, Amount, StageName, CloseDate, Pricebook2Id, RecordType.Name,
       kuga_sub__MonthlyRecurringRevenue__c, kuga_sub__AnnualContractValueInitial__c,
       kuga_sub__TotalContractValue__c, kuga_sub__AnnualRecurringRevenueCommitted__c,
       kuga_sub__NonRecurringRevenue__c
FROM Opportunity WHERE Id = '<opportunity_id>'
```

If HAS_KUGA_SUB = false:
```sql
SELECT Id, Name, AccountId, Amount, StageName, CloseDate, Pricebook2Id, RecordType.Name
FROM Opportunity WHERE Id = '<opportunity_id>'
```

**Existing Quotes:**
```sql
SELECT Id, Name, kugo2p__QuoteName__c, kugo2p__TotalAmount__c, kugo2p__IsPrimary__c
FROM kugo2p__SalesQuote__c WHERE kugo2p__Opportunity__c = '<opportunity_id>'
```

### Step 2: Create the Quote

**Createable fields:**
- `RecordTypeId` — dynamically queried, matched to opportunity type
- `kugo2p__Account__c`, `kugo2p__Opportunity__c`
- `kugo2p__QuoteName__c` — descriptive name
- `kugo2p__Pricebook2Id__c` — must match opportunity pricebook
- `kugo2p__ContactBuying__c` — **REQUIRED**
- `kugo2p__ContactBilling__c`, `kugo2p__ContactShipping__c` — optional
- `kugo2p__IsPrimary__c` — true if no other primary
- `kugo2p__DateOfferValidThrough__c` — default 30 days

**Never set:** `Name` (auto-generated), `kugo2p__Status__c` (workflow), `kugo2p__TotalAmount__c` / `kugo2p__SubtotalAmount__c` / `kugo2p__NetAmount__c` (calculated).

### Step 3: Verify Quote

```sql
SELECT Id, Name, kugo2p__QuoteName__c, kugo2p__TotalAmount__c, kugo2p__SubtotalAmount__c,
       kugo2p__Status__c, kugo2p__IsPrimary__c, kugo2p__DateOfferValidThrough__c
FROM kugo2p__SalesQuote__c WHERE Id = '<quote_id>'
```

```sql
-- Service Lines
SELECT Id, kugo2p__Line__c, kugo2p__ServiceName__c, kugo2p__Quantity__c,
       kugo2p__SalesPrice__c, kugo2p__TotalAmount__c
FROM kugo2p__SalesQuoteServiceLine__c
WHERE kugo2p__SalesQuote__c = '<quote_id>' ORDER BY kugo2p__Line__c

-- Product Lines
SELECT Id, kugo2p__Line__c, kugo2p__Product__r.Name, kugo2p__Quantity__c,
       kugo2p__SalesPrice__c, kugo2p__TotalAmount__c
FROM kugo2p__SalesQuoteProductLine__c
WHERE kugo2p__SalesQuote__c = '<quote_id>' ORDER BY kugo2p__Line__c
```

### Step 4: Amount Interpretation

If HAS_KUGA_SUB = true:
- Compare `kugo2p__TotalAmount__c` to `kuga_sub__AnnualContractValueInitial__c` or `kuga_sub__TotalContractValue__c`
- `Amount` field likely represents MRR, not ACV

If HAS_KUGA_SUB = false:
- Compare `kugo2p__TotalAmount__c` to `Amount`

Always show all amount fields in summary.

---

## Workflow 2: Order Management

Orders are created from quotes, typically via the Kugamon UI "Create Order" action.

### Key Fields on kugo2p__SalesOrder__c

- `kugo2p__Account__c`, `kugo2p__Opportunity__c`, `kugo2p__SalesQuote__c`
- `kugo2p__Pricebook2Id__c`, `RecordTypeId`
- `kugo2p__ContactBuying__c`, `kugo2p__ContactBilling__c`, `kugo2p__ContactShipping__c`
- `kugo2p__DateOrdered__c`

**Auto-managed:** `Name`, `kugo2p__Status__c`, `kugo2p__TotalAmount__c`, etc.

### Order Line Items

Same separation as quotes:
- `kugo2p__SalesOrderServiceLine__c` — recurring service lines
- `kugo2p__SalesOrderProductLine__c` — one-time product lines

### Querying Orders

```sql
SELECT Id, Name, kugo2p__Account__r.Name, kugo2p__Status__c,
       kugo2p__TotalAmount__c, kugo2p__DateOrdered__c, kugo2p__SalesQuote__r.Name
FROM kugo2p__SalesOrder__c WHERE kugo2p__Opportunity__c = '<opportunity_id>'
```

---

## Workflow 3: Order Release (HAS_KUGA_SUB = true only)

**This is the critical subscription lifecycle handoff.** When an order is "released," kuga_sub triggers create downstream records based on checkbox flags on the order.

### Order Release Checkboxes

| Field | What It Creates |
|-------|----------------|
| `kuga_sub__GenerateContract__c` | Standard Contract record linked to the order |
| `kuga_sub__GenerateAsset__c` | Asset records for trackable products/services |
| `kuga_sub__GenerateSubscription__c` | `kuga_sub__Subscription__c` records for renewable items |
| `kuga_sub__GenerateRenewalOpportunity__c` | Renewal Opportunity for next term |

### What Gets Created

**Contract** (`Contract` standard object):
- Linked to order via `kuga_sub__ContractNumber__c` on the order
- Populated with contacts from order (`kuga_sub__ContactBuying__c`, etc.)
- Subscription records roll up to contract (ARR, MRR, counts, dates)
- `kuga_sub__Effective__c` formula indicates if contract is currently active

**Assets** (`Asset` standard object):
- Created for line items where `kuga_sub__Track__c = true` on order lines
- OR where the Product2 has `kuga_sub__Track__c = true`
- Linked to contract via `kuga_sub__ContractNumber__c`

**Subscriptions** (`kuga_sub__Subscription__c`):
- Created for line items where `kuga_sub__Renew__c = true` on order lines
- OR where Product2 has `kuga_sub__Renewable__c = true`
- Key fields: Account, Contract, Order, Service, Quantity, MRR, ARR, Start/End dates, Status
- Linked to parent asset via `kuga_sub__ParentAsset__c`
- Roll up to Contract (ARR, MRR, counts, dates)

**Renewal Opportunity**:
- Created automatically with `kuga_sub__ParentContract__c` and `kuga_sub__ParentOrder__c` references
- Close date based on contract end date
- `kuga_sub__RenewalPriceUpliftPercent__c` carries forward for price adjustments

### Querying Order Release Results

```sql
-- Contract created from order
SELECT Id, ContractNumber, Status, StartDate, EndDate,
       kuga_sub__Effective__c, kuga_sub__AnnualRecurringRevenue__c,
       kuga_sub__MonthlyRecurringRevenue__c, kuga_sub__TotalSubscriptionCount__c
FROM Contract WHERE Id IN (
  SELECT kuga_sub__ContractNumber__c FROM kugo2p__SalesOrder__c WHERE Id = '<order_id>'
)

-- Subscriptions from order
SELECT Id, Name, kuga_sub__Account__r.Name, kuga_sub__Service__r.Name,
       kuga_sub__Quantity__c, kuga_sub__MRR__c, kuga_sub__ARR__c,
       kuga_sub__StartDate__c, kuga_sub__EndDate__c, kuga_sub__Status__c,
       kuga_sub__ContractNumber__r.ContractNumber
FROM kuga_sub__Subscription__c WHERE kuga_sub__Order__c = '<order_id>'

-- Assets from order
SELECT Id, Name, Product2.Name, Quantity, Status,
       kuga_sub__ContractNumber__r.ContractNumber, kuga_sub__ParentSubscription__r.Name
FROM Asset WHERE kuga_sub__ContractNumber__c IN (
  SELECT kuga_sub__ContractNumber__c FROM kugo2p__SalesOrder__c WHERE Id = '<order_id>'
)

-- Renewal Opportunity
SELECT Id, Name, StageName, CloseDate, Amount,
       kuga_sub__ParentContract__r.ContractNumber, kuga_sub__ParentOrder__r.Name
FROM Opportunity WHERE kuga_sub__ParentOrder__c = '<order_id>'
```

### Product2 Flags for Order Release

These Product2 fields (set by kuga_sub) control what happens on order release:

| Field | Effect |
|-------|--------|
| `kuga_sub__Renewable__c` | Product generates a Subscription on release |
| `kuga_sub__Track__c` | Product generates an Asset on release |
| `kuga_sub__RenewalProduct__c` | Substitute product used on renewal quotes |
| `kuga_sub__UpliftRenewalPrice__c` | Apply renewal price uplift percentage |

---

## Workflow 4: Subscription & Contract Management

### Querying Active Contracts

```sql
SELECT Id, ContractNumber, Account.Name, Status, StartDate, EndDate,
       kuga_sub__Effective__c, kuga_sub__AnnualRecurringRevenue__c,
       kuga_sub__MonthlyRecurringRevenue__c, kuga_sub__TotalSubscriptionCount__c,
       kuga_sub__SubscriptionStartDate__c, kuga_sub__SubscriptionEndDate__c,
       kuga_sub__RenewalOpportunity__r.Name
FROM Contract WHERE kuga_sub__Effective__c = true AND AccountId = '<account_id>'
```

### Querying Subscriptions

```sql
SELECT Id, Name, kuga_sub__Service__r.Name, kuga_sub__Quantity__c,
       kuga_sub__MRR__c, kuga_sub__ARR__c, kuga_sub__PurchasePrice__c,
       kuga_sub__StartDate__c, kuga_sub__EndDate__c, kuga_sub__Status__c,
       kuga_sub__Renew__c, kuga_sub__Active__c
FROM kuga_sub__Subscription__c
WHERE kuga_sub__ContractNumber__c = '<contract_id>'
ORDER BY kuga_sub__Service__r.Name
```

### Renewal Automation

kuga_sub provides automated renewal:
1. **Renewal Notice Email** — sent N days before contract end
2. **Renewal Order Auto-Creation** — order created N days before end

---

## Workflow 5: Invoice Management

```sql
SELECT Id, Name, kugo2p__Account__r.Name, kugo2p__Status__c,
       kugo2p__TotalAmount__c, kugo2p__AmountDue__c, kugo2p__DateInvoice__c, kugo2p__DateDue__c
FROM kugo2p__KugamonInvoice__c WHERE kugo2p__Account__c = '<account_id>'
ORDER BY kugo2p__DateInvoice__c DESC
```

### Invoice Lines
```sql
SELECT Id, kugo2p__Description__c, kugo2p__Quantity__c, kugo2p__UnitPrice__c, kugo2p__TotalAmount__c
FROM kugo2p__KugamonInvoiceLine__c WHERE kugo2p__KugamonInvoice__c = '<invoice_id>'
```

### Invoice Scheduling
```sql
SELECT Id, kugo2p__SalesOrder__r.Name, kugo2p__Frequency__c, kugo2p__NextInvoiceDate__c
FROM kugo2p__InvoiceSchedule__c WHERE kugo2p__SalesOrder__c = '<order_id>'
```

---

## Workflow 6: Payment Management

### Payment Gateways
Kugamon supports Stripe, Authorize.Net, PayPal, and eWay via `kugo2p__Processor_Connection__c`.

```sql
SELECT Id, Name, RecordType.Name, kugo2p__Active__c
FROM kugo2p__Processor_Connection__c WHERE kugo2p__Active__c = true
```

### Payment Profiles
```sql
SELECT Id, Name, RecordType.Name, kugo2p__Account__r.Name, kugo2p__Active__c
FROM kugo2p__Payment_Profile__c WHERE kugo2p__Account__c = '<account_id>'
```

### Payments
```sql
SELECT Id, Name, kugo2p__Amount__c, kugo2p__Status__c, kugo2p__DatePayment__c
FROM kugo2p__PaymentX__c WHERE kugo2p__Account__c = '<account_id>'
ORDER BY kugo2p__DatePayment__c DESC
```

### Applied Payments
```sql
SELECT Id, kugo2p__PaymentX__r.Name, kugo2p__KugamonInvoice__r.Name, kugo2p__Amount__c
FROM kugo2p__AppliedPayment__c WHERE kugo2p__PaymentX__c = '<payment_id>'
```

---

## Workflow 7: Shipment & Fulfillment

```sql
SELECT Id, Name, kugo2p__Status__c, kugo2p__SalesOrder__r.Name,
       kugo2p__Carrier__r.Name, kugo2p__TrackingNumber__c, kugo2p__DateShipped__c
FROM kugo2p__Shipment__c WHERE kugo2p__SalesOrder__c = '<order_id>'
```

---

## Product Setup

### AdditionalProductDetail (kugo2p__AdditionalProductDetail__c)
Extended metadata auto-created by Product2Trigger. Key fields:
- `kugo2p__Service__c` — **CRITICAL when HAS_KUGA_SUB = false**: product vs. service classification
- `kugo2p__Taxable__c`, `kugo2p__Configurable__c`, `kugo2p__Kit__c`

### Kit/Bundle (kugo2p__KitBundleMember__c)
```sql
SELECT Id, kugo2p__Product__r.Name, kugo2p__Quantity__c, kugo2p__Required__c
FROM kugo2p__KitBundleMember__c WHERE kugo2p__KitBundle__c = '<kit_product_id>'
```

### Product Configuration
```sql
SELECT Id, Name, kugo2p__Product__r.Name FROM kugo2p__ConfigurationGroup__c
WHERE kugo2p__Product__c = '<product_id>' ORDER BY kugo2p__SortOrder__c

SELECT Id, kugo2p__OptionProduct__r.Name, kugo2p__Required__c, kugo2p__Default__c
FROM kugo2p__ConfigurationOption__c WHERE kugo2p__ConfigurationGroup__c = '<group_id>'
```

### Tiered Pricing
```sql
SELECT Id, kugo2p__FromQuantity__c, kugo2p__ToQuantity__c, kugo2p__Price__c
FROM kugo2p__Tier__c WHERE kugo2p__TieredPricing__c = '<tiered_pricing_id>'
ORDER BY kugo2p__FromQuantity__c
```

### Account-Specific Pricing
```sql
SELECT Id, kugo2p__Account__r.Name, kugo2p__Product__r.Name, kugo2p__Price__c, kugo2p__Discount__c
FROM kugo2p__AccountPricing__c WHERE kugo2p__Account__c = '<account_id>'
```

---

## Tax Configuration

### US Sales Tax
```sql
SELECT Id, Name, kugo2p__State__c, kugo2p__County__c, kugo2p__City__c
FROM kugo2p__TaxLocation__c WHERE kugo2p__State__c = '<state>'

SELECT Id, kugo2p__TaxLocation__r.Name, kugo2p__Rate__c, kugo2p__EffectiveDate__c
FROM kugo2p__TaxRate__c WHERE kugo2p__TaxLocation__c = '<location_id>'
```

### International VAT
```sql
SELECT Id, Name, kugo2p__Country__c FROM kugo2p__VAT__c
SELECT Id, kugo2p__VAT__r.Name, kugo2p__Rate__c FROM kugo2p__VATRate__c WHERE kugo2p__VAT__c = '<vat_id>'
```

### Tax Exemption
```sql
SELECT Id, kugo2p__TaxExempt__c, kugo2p__TaxExemptNumber__c
FROM kugo2p__AdditionalAccountDetail__c WHERE kugo2p__Account__c = '<account_id>'
```

---

## Consistency Checking and Synchronization

**CRITICAL:** When updating quotes OR opportunity line items, ALWAYS check for consistency and synchronize both sides unless explicitly told not to.

### What to Compare

| Field | Quote Line | Opportunity Line |
|-------|-----------|-----------------|
| Product/Service | `kugo2p__ServiceName__c` / `kugo2p__Product__r.Name` | `Product2.Name` |
| Quantity | `kugo2p__Quantity__c` | `Quantity` |
| Unit Price | `kugo2p__SalesPrice__c` | `UnitPrice` |
| Start Date | `kugo2p__DateServiceStart__c` | `ServiceDate` |

If HAS_KUGA_SUB = true, also compare Term and End Date.

**Default:** Update BOTH sides when either changes. Skip sync only if user explicitly says so.

---

## Common Issues

**Quote total vs. Amount mismatch:**
If HAS_KUGA_SUB = true: `Amount` may be MRR; compare to `kuga_sub__AnnualContractValueInitial__c`.
If HAS_KUGA_SUB = false: Quote total should match opportunity line item sum.

**ACV double-counting (HAS_KUGA_SUB only):**
Products with `kuga_sub__Renew__c = false` but non-zero MRR/ARR → double-counted.
Fix: Set `kuga_sub__Renew__c = true` on recurring line items.

**Line items not auto-populating:**
Check BOTH `SalesQuoteProductLine__c` and `SalesQuoteServiceLine__c`. If HAS_KUGA_SUB = false, verify `kugo2p__AdditionalProductDetail__c.kugo2p__Service__c`.

**Order Release not creating contracts/subscriptions:**
Verify checkboxes: `kuga_sub__GenerateContract__c`, `kuga_sub__GenerateSubscription__c`, etc. Also check Product2 flags: `kuga_sub__Renewable__c`, `kuga_sub__Track__c`.

**kuga_sub fields don't exist:**
Set `HAS_KUGA_SUB = false`. Use `kugo2p__AdditionalProductDetail__c.kugo2p__Service__c` instead.

**Missing billing address / No contact:** Must be resolved before quote creation.

---

## Appendix A: Field Reference

### Opportunity Fields

#### Required Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `Name` | Text(120) | Opportunity name |
| `StageName` | Picklist | Current stage (use "Qualification" when creating with quote) |
| `CloseDate` | Date | Expected close date |

#### Strongly Recommended Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `AccountId` | Lookup(Account) | Required for Kugamon CPQ to work properly |
| `Pricebook2Id` | Lookup(Pricebook2) | Required if adding opportunity products |

#### Optional Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `Amount` | Currency | Auto-calculated from line items if products exist |
| `Type` | Picklist | E.g., "New Business", "Existing Business" |
| `RecordTypeId` | Lookup(RecordType) | Map to quote record type (New/Renewal/Expansion) |

#### kuga_sub Fields on Opportunity (HAS_KUGA_SUB = true only)

**Roll-Up Summary Fields (read-only, calculated from line items):**

| Field API Name | Description |
|----------------|-------------|
| `kuga_sub__MonthlyRecurringRevenue__c` | SUM of line item MRR |
| `kuga_sub__AnnualRecurringRevenueCommitted__c` | SUM of line item ARR |
| `kuga_sub__NonRecurringRevenue__c` | SUM of line item non-recurring revenue |
| `kuga_sub__Amount__c` | SUM of line item amounts |
| `kuga_sub__DateRequired__c` | MIN of line item service dates |
| `kuga_sub__ServiceDateExpires__c` | MAX of line item end dates |

**Formula Fields (read-only):**

| Field API Name | Description | Formula |
|----------------|-------------|---------|
| `kuga_sub__AnnualContractValueInitial__c` | ACV | NonRecurring + ARR |
| `kuga_sub__TotalContractValue__c` | TCV | Net amount from primary quote/order |
| `kuga_sub__AnnualRecurringRevenueForecast__c` | ARR forecast | MRR x 12 |
| `kuga_sub__ExpectedRevenue__c` | Expected revenue | Amount x Probability |
| `kuga_sub__OpportunityAmount__c` | Opp amount | Formula |
| `kuga_sub__ContractEndDate__c` | Contract end date | From parent contract |
| `kuga_sub__ParentContractEndDate__c` | Parent contract end | From parent contract |

**Editable Fields:**

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kuga_sub__ParentContract__c` | Lookup(Contract) | Parent contract (for renewals) |
| `kuga_sub__ParentOrder__c` | Lookup(Order) | Parent order (for renewals) |
| `kuga_sub__AutoEmailRenewalOrder__c` | Checkbox | Auto-email renewal order |
| `kuga_sub__AutoRenewedOrder__c` | Lookup(Order) | Auto-created renewal order |
| `kuga_sub__RenewalOrderAutoCreationDate__c` | Date | When renewal order auto-creates |
| `kuga_sub__RenewalPriceUpliftPercent__c` | Percent | Price uplift % for renewal |

### OpportunityLineItem Fields

#### Required Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `OpportunityId` | Lookup(Opportunity) | Parent opportunity |
| `PricebookEntryId` | Lookup(PricebookEntry) | Links to product via pricebook |
| `Quantity` | Number(10,2) | Minimum 1 |

#### kuga_sub Fields (HAS_KUGA_SUB = true only)

**Editable:**

| Field API Name | Type | Description | Default |
|----------------|------|-------------|---------|
| `kuga_sub__Renew__c` | Checkbox | **CRITICAL**: recurring vs one-time | false |
| `kuga_sub__ServiceTerm__c` | Number | Term length (e.g., 12, 24, 36) | |
| `kuga_sub__UnitofTerm__c` | Picklist | "Month" or "Year" | |
| `kuga_sub__ServiceTermBehavior__c` | Picklist | Term behavior | |
| `kuga_sub__NonUpliftSalesPrice__c` | Currency | Pre-uplift price | |

**Calculated (read-only):**

| Field API Name | Description | When Renew=true | When Renew=false |
|----------------|-------------|-----------------|------------------|
| `kuga_sub__MRR__c` | Monthly recurring revenue | Calculated | 0 |
| `kuga_sub__ARR__c` | Annual recurring revenue | MRR x 12 | 0 |
| `kuga_sub__ARRForecast__c` | ARR forecast | Calculated | 0 |
| `kuga_sub__NonRecurringRevenue__c` | One-time revenue | 0 | Line total |
| `kuga_sub__NetAmount__c` | Net amount (formula) | After discounts | After discounts |
| `kuga_sub__TotalAmount__c` | Total amount (formula) | Calculated | Calculated |
| `kuga_sub__ListAmount__c` | List amount (formula) | List price total | List price total |
| `kuga_sub__DateServiceEnd__c` | Service end date | ServiceDate + Term | |
| `kuga_sub__Service__c` | Is service? (formula) | Formula | Formula |
| `kuga_sub__LineTerm__c` | Term display (formula) | e.g., "12 Month" | |
| `kuga_sub__DiscountSalesPrice__c` | Discount amount (formula) | | |
| `kuga_sub__EffectiveDiscount__c` | Effective discount % (formula) | | |
| `kuga_sub__UpliftRenewalPrice__c` | Uplift flag (formula) | | |

#### Important Optional Fields (standard)

| Field API Name | Type | Description |
|----------------|------|-------------|
| `UnitPrice` | Currency | Override pricebook price |
| `ServiceDate` | Date | Service start date |
| `Discount` | Percent | Discount percentage |
| `Description` | Text | Line item description |

### Quote Fields (kugo2p__SalesQuote__c)

#### Createable Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `RecordTypeId` | Lookup(RecordType) | New/Renewal/Expansion (query dynamically) |
| `kugo2p__Account__c` | Lookup(Account) | Required |
| `kugo2p__Opportunity__c` | Lookup(Opportunity) | Required |
| `kugo2p__QuoteName__c` | Text | Quote name |
| `kugo2p__Pricebook2Id__c` | Lookup(Pricebook2) | Must match opportunity pricebook |
| `kugo2p__ContactBuying__c` | Lookup(Contact) | **REQUIRED** buying contact |
| `kugo2p__ContactBilling__c` | Lookup(Contact) | Optional billing contact |
| `kugo2p__ContactShipping__c` | Lookup(Contact) | Optional shipping contact |
| `kugo2p__IsPrimary__c` | Checkbox | Primary quote flag |
| `kugo2p__DateOfferValidThrough__c` | Date | Expiration (default: 30 days) |

#### kuga_sub Fields on Quote

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kuga_sub__ContractNumber__c` | Lookup(Contract) | Linked contract (renewals) |
| `kuga_sub__ContractEndDate__c` | Formula(Date) | Contract end date |

#### Auto-Managed Fields (Never Set)

| Field | Description |
|-------|-------------|
| `Name` | Auto-generated quote number |
| `kugo2p__Status__c` | Workflow-managed |
| `kugo2p__TotalAmount__c` | Calculated from lines |
| `kugo2p__SubtotalAmount__c` | Calculated |
| `kugo2p__NetAmount__c` | Calculated |

### Quote Line Item Objects

#### Service Lines (kugo2p__SalesQuoteServiceLine__c)

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kugo2p__SalesQuote__c` | Lookup | Parent quote |
| `kugo2p__ServiceName__c` | Text | Service name |
| `kugo2p__Quantity__c` | Number | Quantity |
| `kugo2p__SalesPrice__c` | Currency | Unit price |
| `kugo2p__TotalAmount__c` | Currency | Line total |
| `kugo2p__Line__c` | Formula | Line number |
| `kugo2p__DateServiceStart__c` | Date | Service start |
| `kugo2p__DateServiceEnd__c` | Date | Service end |
| `kugo2p__ServiceTerm__c` | Number | Term length |
| `kuga_sub__Renew__c` | Checkbox | Renewable (HAS_KUGA_SUB only) |

#### Product Lines (kugo2p__SalesQuoteProductLine__c)

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kugo2p__SalesQuote__c` | Lookup | Parent quote |
| `kugo2p__Product__c` | Lookup | Product reference |
| `kugo2p__Quantity__c` | Number | Quantity |
| `kugo2p__SalesPrice__c` | Currency | Unit price |
| `kugo2p__TotalAmount__c` | Currency | Line total |
| `kugo2p__Line__c` | Formula | Line number |
| `kuga_sub__Renew__c` | Checkbox | Renewable (HAS_KUGA_SUB only) |

### Order Fields (kugo2p__SalesOrder__c)

#### kuga_sub Fields — Order Release Controls (HAS_KUGA_SUB only)

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kuga_sub__GenerateContract__c` | Checkbox | Create Contract on release |
| `kuga_sub__GenerateAsset__c` | Checkbox | Create Assets on release |
| `kuga_sub__GenerateSubscription__c` | Checkbox | Create Subscriptions on release |
| `kuga_sub__GenerateRenewalOpportunity__c` | Checkbox | Create Renewal Opp on release |
| `kuga_sub__ContractNumber__c` | Lookup(Contract) | Generated contract |
| `kuga_sub__ParentContract__c` | Lookup(Contract) | Parent contract (renewals) |
| `kuga_sub__RenewalOpportunity__c` | Lookup(Opportunity) | Generated renewal opp |
| `kuga_sub__UpdateContractContacts__c` | Multi-Select | Which contacts to update |

#### Order Line kuga_sub Fields

| Field API Name | Type | On Object | Description |
|----------------|------|-----------|-------------|
| `kuga_sub__Renew__c` | Checkbox | Service & Product Lines | Creates Subscription on release |
| `kuga_sub__Track__c` | Checkbox | Service & Product Lines | Creates Asset on release |

### Subscription Fields (kuga_sub__Subscription__c)

| Field API Name | Type | Updateable | Description |
|----------------|------|------------|-------------|
| `Name` | String | No | Auto-generated |
| `kuga_sub__Account__c` | Lookup(Account) | No | Account |
| `kuga_sub__ContractNumber__c` | Lookup(Contract) | No | Parent contract |
| `kuga_sub__Order__c` | Lookup(SalesOrder) | Yes | Source order |
| `kuga_sub__OrderServiceLine__c` | Lookup | Yes | Source order line |
| `kuga_sub__Service__c` | Lookup(Product2) | Yes | Product/service |
| `kuga_sub__Quantity__c` | Number | Yes | Quantity |
| `kuga_sub__PurchasePrice__c` | Currency | Yes | Unit price |
| `kuga_sub__MRR__c` | Currency | Yes | Monthly recurring revenue |
| `kuga_sub__ARR__c` | Currency | Yes | Annual recurring revenue |
| `kuga_sub__NetAmount__c` | Currency | Yes | Net amount |
| `kuga_sub__TotalAmount__c` | Currency | Yes | Total amount |
| `kuga_sub__StartDate__c` | Date | Yes | Start date |
| `kuga_sub__EndDate__c` | Date | Yes | End date |
| `kuga_sub__Status__c` | Picklist | Yes | Status |
| `kuga_sub__Renew__c` | Checkbox | No | Is renewable |
| `kuga_sub__Active__c` | Checkbox | No | Is active (read-only) |
| `kuga_sub__IsActive__c` | Checkbox | Yes | Is active (editable) |
| `kuga_sub__ParentAsset__c` | Lookup(Asset) | Yes | Parent asset |
| `kuga_sub__ParentSubscription__c` | Lookup(Subscription) | Yes | Parent subscription |
| `kuga_sub__ServiceTerm__c` | Number | No | Service term |
| `kuga_sub__UnitofTerm__c` | String | No | Unit of term |
| `kuga_sub__OrderRecordType__c` | String | No | Order record type |

### Contract kuga_sub Fields

#### Roll-Up Summary Fields (read-only)

| Field API Name | Description |
|----------------|-------------|
| `kuga_sub__AnnualRecurringRevenue__c` | SUM of subscription ARR |
| `kuga_sub__MonthlyRecurringRevenue__c` | SUM of subscription MRR |
| `kuga_sub__TotalSubscriptionAmount__c` | SUM of subscription amounts |
| `kuga_sub__TotalSubscriptionCount__c` | COUNT of subscriptions |
| `kuga_sub__TotalSubscriptionQuantity__c` | SUM of subscription quantities |
| `kuga_sub__SubscriptionStartDate__c` | MIN start date |
| `kuga_sub__SubscriptionEndDate__c` | MAX end date |

#### Editable Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kuga_sub__RenewalOpportunity__c` | Lookup(Opportunity) | Renewal opportunity |
| `kuga_sub__RenewalTerm__c` | Number | Renewal term length |
| `kuga_sub__AutoEmailRenewalNotice__c` | Checkbox | Auto-send renewal notice |
| `kuga_sub__AutoEmailRenewalOrder__c` | Checkbox | Auto-send renewal order |
| `kuga_sub__Pricebook2Id__c` | Lookup(Pricebook) | Pricebook for renewals |
| `kuga_sub__ContactBuying__c` | Lookup(Contact) | Buying contact |
| `kuga_sub__ContactBilling__c` | Lookup(Contact) | Billing contact |
| `kuga_sub__ContactShipping__c` | Lookup(Contact) | Shipping contact |
| `kuga_sub__Expanded__c` | Checkbox | Has been expanded |
| `kuga_sub__LastRenewalNoticeSentDate__c` | Date | Last renewal notice date |

#### Formula Fields

| Field API Name | Description |
|----------------|-------------|
| `kuga_sub__Effective__c` | Is contract currently active |
| `kuga_sub__ContractRenewalNoticeDate__c` | When renewal notice should send |
| `kuga_sub__SendRenewalNoticeToday__c` | Should notice send today |
| `kuga_sub__AnnualRecurringRevenueForecast__c` | MRR x 12 |

### Asset kuga_sub Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kuga_sub__ContractNumber__c` | Lookup(Contract) | Parent contract |
| `kuga_sub__ParentSubscription__c` | Lookup(Subscription) | Parent subscription |
| `kuga_sub__ParentLine__c` | Formula(Text) | Parent line reference |
| `kuga_sub__Renew__c` | Formula(Checkbox) | Is renewable |

### Product2 kuga_sub Fields

| Field API Name | Type | Description |
|----------------|------|-------------|
| `kuga_sub__Renewable__c` | Checkbox | Generates Subscription on order release |
| `kuga_sub__Track__c` | Checkbox | Generates Asset on order release |
| `kuga_sub__RenewalProduct__c` | Lookup(Product2) | Substitute product for renewal |
| `kuga_sub__UpliftRenewalPrice__c` | Checkbox | Apply price uplift on renewal |

### Field Interdependencies

#### Opportunity Product Requirements
1. Opportunity must have `Pricebook2Id` set
2. Product must have a `PricebookEntry` in that pricebook
3. Use `PricebookEntryId` (not `Product2Id`) when creating line items

#### Revenue Calculation Dependencies (HAS_KUGA_SUB = true)
1. Set `kuga_sub__Renew__c` correctly on line items
2. For recurring: Set `ServiceTerm` and `UnitofTerm`
3. Kugamon automation populates MRR/ARR/NonRecurring
4. These roll up to opportunity-level fields
5. ACV = NonRecurring + ARR

#### Order Release Dependencies
1. Order line items need `kuga_sub__Renew__c` or `kuga_sub__Track__c` set
2. OR Product2 needs `kuga_sub__Renewable__c` or `kuga_sub__Track__c`
3. Order Generate checkboxes must be true
4. Triggers create Contract, Assets, Subscriptions, Renewal Opportunity

---

## Appendix B: Amount Fields Guide

### The Amount Field Problem

In Salesforce orgs with subscription management, the standard `Amount` field on Opportunity can represent different values depending on configuration: Monthly Recurring Revenue (MRR), Annual Recurring Revenue (ARR), Total Contract Value (TCV), or Annual Contract Value (ACV). This creates confusion when comparing opportunity amounts to quote totals.

### Subscription Amount Fields

#### On Opportunity (kuga_sub__* fields)

| Field API Name | Meaning | Example |
|----------------|---------|---------|
| `Amount` | **May be MRR or ACV** | $10,000 (could be monthly or annual) |
| `kuga_sub__MonthlyRecurringRevenue__c` | Monthly recurring revenue | $10,000/month |
| `kuga_sub__AnnualContractValueInitial__c` | Annual contract value | $120,000/year |
| `kuga_sub__TotalContractValue__c` | Total contract value | $120,000 (1-year) or $360,000 (3-year) |
| `kuga_sub__AnnualRecurringRevenueCommitted__c` | Annual recurring revenue | $120,000/year |
| `kuga_sub__NonRecurringRevenue__c` | One-time fees | $5,000 |

#### On Quote (kugo2p__* fields)

| Field API Name | Meaning |
|----------------|---------|
| `kugo2p__TotalAmount__c` | Total quote amount (typically annual or total contract value) |
| `kugo2p__SubtotalAmount__c` | Subtotal before discounts/taxes |

### Comparison Rules

**When Subscription Fields Exist** (any `kuga_sub__*` fields present):
1. Do NOT compare `kugo2p__TotalAmount__c` to `Amount`
2. DO compare `kugo2p__TotalAmount__c` to `kuga_sub__AnnualContractValueInitial__c` (preferred) or `kuga_sub__TotalContractValue__c`
3. Reasoning: In subscription orgs, `Amount` is often configured to show MRR, while quotes show annual/total values

**When Subscription Fields Do NOT Exist:**
1. DO compare `kugo2p__TotalAmount__c` to `Amount`
2. Reasoning: Without subscription management, `Amount` represents the total opportunity value

### Amount Comparison Examples

**Example 1 — Subscription Org (MRR in Amount field):**
- Opportunity `Amount`: $10,000 / `kuga_sub__MonthlyRecurringRevenue__c`: $10,000 / `kuga_sub__AnnualContractValueInitial__c`: $120,000
- Quote `kugo2p__TotalAmount__c`: $120,000
- Correct: Quote matches ACV ($120,000), not Amount ($10,000 MRR)

**Example 2 — Subscription Org (ACV in Amount field):**
- Opportunity `Amount`: $120,000 / `kuga_sub__AnnualContractValueInitial__c`: $120,000
- Quote `kugo2p__TotalAmount__c`: $120,000
- Correct: Quote matches both Amount and ACV ($120,000)

**Example 3 — Non-Subscription Org:**
- Opportunity `Amount`: $120,000 (no kuga_sub__* fields)
- Quote `kugo2p__TotalAmount__c`: $120,000
- Correct: Quote matches Amount ($120,000)

### Amount Field Best Practices

1. Always query ALL amount fields before making comparisons
2. Check for presence of subscription fields to determine comparison strategy
3. Show all relevant amounts in user-facing summaries
4. Never assume what `Amount` represents — let the data guide you
5. Document discrepancies clearly when amounts don't match expected patterns

---

## Appendix C: Record Types Guide

### Dynamic Record Type Discovery

**NEVER hardcode Record Type IDs.** They vary between orgs. Always query dynamically:

```sql
SELECT Id, Name, SObjectType, DeveloperName, IsActive
FROM RecordType
WHERE SObjectType IN (
  'kugo2p__SalesQuote__c',
  'kugo2p__SalesOrder__c',
  'kugo2p__Payment_Profile__c',
  'kugo2p__Processor_Connection__c',
  'kugo2p__Payment_Method__c',
  'Opportunity'
)
AND IsActive = true
ORDER BY SObjectType, Name
```

Cache results for the session and use the returned IDs.

### Mapping Opportunity to Quote/Order Record Types

Match by name:
- Opportunity RecordType.Name contains "Renewal" → use Quote/Order record type named "Renewal"
- Opportunity RecordType.Name contains "New" → use Quote/Order record type named "New Business" (or "New")
- Opportunity RecordType.Name contains "Expansion" → use Quote/Order record type named "Expansion"

If no match found, ask the user or default to "New Business."

### Known Record Type Categories

**Quote & Order Record Types** (names may vary by org): Renewal, New Business, Expansion

**Payment Profile Record Types:** Credit Card, AuthNet Subscription, Native Subscription, PayPal Recurring Payment, PayPal Subscription, Generic Profile

**Processor Connection Record Types:** Stripe, Authorize.Net, PayPal, eWay

**Payment Method Record Types:** Stripe, Authorize.Net, PayPal, eWay, Salesforce.com

### Record Type Notes

- Some orgs may not have all record types active
- Quote/Order record types may not exist in every org — if the query returns none, create quotes/orders without a RecordTypeId
- Always verify returned IDs before using them in DML operations

---

## Appendix D: Renew Field Guide

### Overview

The `kuga_sub__Renew__c` field on OpportunityLineItem is critical for proper revenue classification in Kugamon CPQ. It determines whether a product/service is treated as recurring or non-recurring.

### Renew Field Behavior

**When Renew = true:**
- Product is treated as a recurring subscription
- Revenue flows to `kuga_sub__MRR__c` (Monthly Recurring Revenue) and `kuga_sub__ARR__c` (Annual Recurring Revenue)
- `kuga_sub__NonRecurringRevenue__c` = 0

**When Renew = false (or null):**
- Product is treated as non-recurring/one-time
- Revenue flows to `kuga_sub__NonRecurringRevenue__c`
- MRR and ARR remain 0

### Revenue Roll-Ups to Opportunity

**Opportunity Line Item Fields (Source):** `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__NonRecurringRevenue__c`

**Opportunity Roll-Up Fields (Calculated):**
1. `kuga_sub__MonthlyRecurringRevenue__c` — Roll-up sum of all line item MRR
2. `kuga_sub__AnnualRecurringRevenueCommitted__c` — Roll-up sum of all line item ARR
3. `kuga_sub__NonRecurringRevenue__c` — Roll-up sum of all line item non-recurring revenue
4. `kuga_sub__AnnualContractValueInitial__c` — **FORMULA:** NonRecurringRevenue + AnnualRecurringRevenueCommitted (this is where double-counting occurs if Renew is set incorrectly)

### Double-Counting Problem

When a recurring product has `Renew = false`, the line item populates BOTH `kuga_sub__ARR__c` AND `kuga_sub__NonRecurringRevenue__c`, causing the ACV formula to count the product twice.

**Wrong Configuration:**
```
OpportunityLineItem: "Annual Support Contract"
- UnitPrice: $2,000/month
- kuga_sub__Renew__c: false  ← WRONG
- kuga_sub__ARR__c: $24,000
- kuga_sub__NonRecurringRevenue__c: $24,000  ← Should be $0
- ACV: $24,000 + $24,000 = $48,000  ← DOUBLE-COUNTED
```

**Correct Configuration:**
```
OpportunityLineItem: "Annual Support Contract"
- UnitPrice: $2,000/month
- kuga_sub__Renew__c: true  ← CORRECT
- kuga_sub__ARR__c: $24,000
- kuga_sub__NonRecurringRevenue__c: $0  ← Correct
- ACV: $0 + $24,000 = $24,000  ← Correct
```

### Product Types Guide

| Product Type | Renew Setting | Examples |
|--------------|---------------|----------|
| Subscriptions | `true` | SaaS licenses, recurring services |
| Support Contracts | `true` | Standard Support, Premium Support |
| Hardware | `false` | Servers, equipment, devices |
| Implementation | `false` | Setup fees, onboarding, training |
| Professional Services | `false` | Consulting hours (unless retainer) |
| One-time Licenses | `false` | Perpetual software licenses |
| Recurring Retainers | `true` | Monthly consulting retainers |

### Troubleshooting ACV Higher Than Expected

1. Query the opportunity:
```sql
SELECT kuga_sub__NonRecurringRevenue__c,
       kuga_sub__AnnualRecurringRevenueCommitted__c,
       kuga_sub__AnnualContractValueInitial__c
FROM Opportunity WHERE Id = '<opp_id>'
```

2. If NonRecurring seems too high, check line items:
```sql
SELECT Id, Product2.Name, kuga_sub__Renew__c,
       kuga_sub__MRR__c, kuga_sub__ARR__c,
       kuga_sub__NonRecurringRevenue__c
FROM OpportunityLineItem WHERE OpportunityId = '<opp_id>'
```

3. Look for products with ALL of: `kuga_sub__Renew__c = false`, `kuga_sub__ARR__c > 0`, and `kuga_sub__NonRecurringRevenue__c > 0`

4. Fix by updating: `{ "Id": "00kxxx", "kuga_sub__Renew__c": true }`

The NonRecurringRevenue will automatically recalculate to $0 via Kugamon's automation.

### Technical Detail

Kugamon automation calculates MRR/ARR based on product pricing and term, then populates NonRecurringRevenue based on the Renew field. When `Renew = false`, NonRecurringRevenue = line total. When `Renew = true`, NonRecurringRevenue = 0. Attempting to update `kuga_sub__NonRecurringRevenue__c` directly will be overridden — the Renew field is the source of truth.
