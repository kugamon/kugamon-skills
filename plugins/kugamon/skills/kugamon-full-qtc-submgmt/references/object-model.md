# Object Model

Reference for the `kugamon-full-qtc-submgmt` skill. Package versions are listed only in SKILL.md (Package Detection → Package versions).

## kugo2p objects (Kugamon Quote to Cash)

51 custom objects. A record Id's first three characters (the key prefix) name its object, so a pasted Id such as `a0I…` is an Order and `a0P…` an Invoice. All API names carry the `kugo2p__` prefix.

| Group | Object (API name) | Label | Prefix | What it holds |
|---|---|---|---|---|
| Accounts | `AdditionalAccountDetail__c` | Additional Account Info | `a00` | Kugamon's record for each Account (AAD): default Contacts, Price Book, payment and delivery terms, tax setup |
| Quotes | `SalesQuote__c` | Quote | `a03` | Quote header |
| Quotes | `SalesQuoteProductLine__c` | Quote Product Line | `a0D` | Product (non-service) lines |
| Quotes | `SalesQuoteServiceLine__c` | Quote Service Line | `a0E` | Service lines |
| Quotes | `SalesQuoteOptionalLine__c` | Quote Optional Line | `a0o` | Optional lines (quotes only) |
| Quotes | `SalesQuoteAdditionalChargeCredit__c` | Quote Additional Charge/Credit | `a0G` | A charge or credit on a Quote |
| Quotes | `QuoteLineGroup__c` | Quote Line Group | `a0u` | Line groups on a Quote |
| Quotes | `ChatParticipant__c` | Chat Participant | `a0B` | One person in a Quote Chat, with their read pointer |
| Quotes | `ChatMessage__c` | Chat Message | `a0H` | One Quote Chat message |
| Line editing | `Favorite__c` | Favorite | `a0r` | A saved set of lines |
| Line editing | `FavoriteMember__c` | Favorite Member | `a0Q` | A line in a Favorite |
| Line editing | `FavoriteShare__c` | Favorite Share | `a0q` | A Favorite shared with a user |
| Orders | `SalesOrder__c` | Order | `a0I` | Order header |
| Orders | `SalesOrderProductLine__c` | Order Product Line | `a0J` | Product (non-service) lines |
| Orders | `SalesOrderServiceLine__c` | Order Service Line | `a0K` | Service lines |
| Orders | `SalesOrderAdditionalChargeCredit__c` | Order Additional Charge/Credit | `a0N` | A charge or credit on an Order |
| Orders | `OrderLineGroup__c` | Order Line Group | `a0s` | Line groups on an Order |
| Fulfillment | `Shipment__c` | Shipment | `a0X` | Shipment (`SHP-` numbered) |
| Fulfillment | `ShipmentLine__c` | Shipment Line | `a0a` | Shipment Line, one per Order Product Line shipped |
| Fulfillment | `ServiceDeliverySchedule__c` | Service Delivery | `a0g` | Delivery periods of a Service Line |
| Fulfillment | `Carrier__c` | Carrier | `a0C` | Shipping carriers |
| Fulfillment | `Warehouse__c` | Warehouse | `a05` | Warehouses |
| Invoices | `KugamonInvoice__c` | Invoice | `a0P` | Invoice header |
| Invoices | `KugamonInvoiceLine__c` | Invoice Line | `a0S` | Invoice Line (bills a Shipment Line or a Service Delivery row) |
| Invoices | `KugamonInvoiceAdditionalChargeCredit__c` | Invoice Additional Charge/Credit | `a0R` | A charge or credit on an Invoice |
| Invoices | `InvoiceSchedule__c` | Invoice Schedule | `a0L` | How often an Order is invoiced |
| Invoices | `OrderInvoiceRelationship__c` | Order/Invoice Relationship | `a0h` | Links an Order and an Invoice (many-to-many) |
| Payments | `PaymentX__c` | Payment | `a0b` | Payment, authorization or refund |
| Payments | `AppliedPayment__c` | Applied Payment | `a0j` | A Payment's amount applied to an Order or Invoice line |
| Payments | `Payment_Method__c` | Payment Method | `a0m` | A Contact's stored payment method |
| Payments | `Payment_Profile__c` | Recurring Charge | `a0c` | A recurring charge |
| Payments | `Processor_Connection__c` | Processor Connection | `a0d` | A payment gateway connection |
| Payments | `Shopping_Cart_Item__c` | Shopping Cart Item | `a0f` | Cart items for the older Site checkout page |
| Product Management | `AdditionalProductDetail__c` | Additional Product Info | `a01` | Kugamon's record for each Product (APD) |
| Product Management | `AccountPricing__c` | Account Pricing | `a0v` | Negotiated prices or discounts for an Account |
| Product Management | `TieredPricing__c` | Tiered Pricing | `a0W` | Tiered Pricing for a Product and Price Book |
| Product Management | `Tier__c` | Tier | `a0p` | A tier of a Tiered Pricing |
| Product Management | `ProductCost__c` | Product Cost | `a0t` | A Product's costs |
| Product Management | `AdditionalChargeCredit__c` | Additional Charge/Credit | `a0F` | Reusable charge and credit templates |
| Product Management | `KitBundleMember__c` | Kit/Bundle Member | `a02` | A kit's member Products |
| Product Management | `ConfigurationGroup__c` | Configuration Group | `a04` | A configurable Product's option groups |
| Product Management | `ConfigurationOption__c` | Configuration Option | `a0M` | A configurable Product's options |
| Product Management | `ProductCatalog__c` | Product Catalog | `a10` | Product Catalogs |
| Product Management | `ProductCategory__c` | Product Category | `a12` | A Catalog's categories |
| Product Management | `ProductCategoryProduct__c` | Product Category Product | `a11` | A Product in a category |
| Settings and tax | `KugamonSetting__c` | Kugamon Setting | `a06` | Kugamon Settings (one record) |
| Settings and tax | `Settings__c` | Payment Setting | `a0e` | Payment Settings (one record) |
| Settings and tax | `TaxLocation__c` | Tax Location | `a07` | Tax Locations |
| Settings and tax | `TaxRate__c` | Tax Rate | `a08` | A Tax Location's rates |
| Settings and tax | `VAT__c` | VAT/GST | `a09` | VAT/GST definitions |
| Settings and tax | `VATRate__c` | VAT/GST Rate | `a0A` | A VAT/GST's rates |

**Settings that aren't records of these objects.** `kugo2p__Application_Settings__c` is a hierarchy custom setting holding the switches `kugo2p__DisableInvoices__c`, `kugo2p__DisableShipments__c`, `kugo2p__EnableOnlineChat__c`, `kugo2p__EnableTwinFieldsMapping__c` and `kugo2p__DisableTriggers__c`. Like all of Kugamon's custom settings it is **protected**: a customer's org can't query it or see it in Setup. Customers change switches through Kugamon's own pages (the Kugamon Configuration page); a switch no page sets, such as Disable Invoices, is changed by Kugamon Support.

### How the main records connect

- **AAD** — master-detail child of the Account (`kugo2p__Account__c`); Quotes, Orders and Invoices point to it with `kugo2p__AdditionalAccountDetail__c`.
- **Quote and Order lines** — master-detail children of the header (`kugo2p__SalesQuote__c` / `kugo2p__SalesOrder__c`). Their `kugo2p__Product__c` (Product Lines) and `kugo2p__Service__c` (Service Lines) look up the **APD**, not Product2; the APD's `kugo2p__ReferenceProduct__c` is the Product2. A Quote Line links to its Opportunity Product with `kugo2p__OpportunityLineItem__c` / `kugo2p__OpportunityLineItemId__c`.
- **Order** — `kugo2p__SalesQuote__c` (source Quote), `kugo2p__Opportunity__c`, `kugo2p__InvoiceSchedule__c`.
- **Shipment Line** — child of the Shipment; `kugo2p__SalesOrderProductLine__c`, `kugo2p__SalesOrder__c`.
- **Service Delivery Schedule** — child of the Order; `kugo2p__SalesOrderServiceLine__c`.
- **Invoice Line** — `kugo2p__Invoice__c` (its Invoice), `kugo2p__SalesOrder__c`, `kugo2p__SalesOrderProductLine__c` or `kugo2p__SalesOrderServiceLine__c`, and the `kugo2p__ShipmentLine__c` or `kugo2p__ServiceDeliverySchedule__c` it bills.
- **Order/Invoice Relationship** — master-detail to both `kugo2p__SalesOrder__c` and `kugo2p__Invoice__c`.
- **Payment** — `kugo2p__SalesOrder__c` or `kugo2p__Invoice__c` (never both), `kugo2p__Account__c`, `kugo2p__Contact__c`, `kugo2p__Processor_Connection__c`, `kugo2p__Payment_Method__c`, `kugo2p__Payment_Profile__c`, `kugo2p__Parent_Transaction__c` (refunds).
- **Applied Payment** — master-detail child of the Payment (`kugo2p__Payment__c`); points at one of `kugo2p__InvoiceLine__c`, `kugo2p__InvoiceAdditionalChargeCredit__c`, `kugo2p__SalesOrderProductLine__c`, `kugo2p__SalesOrderServiceLine__c`, `kugo2p__SalesOrderAdditionalChargeCredit__c`, with `kugo2p__Invoice__c` / `kugo2p__SalesOrder__c`.
- **Chat** — Participant → Quote (`kugo2p__QuoteNumber__c`), User or Contact; Message → Participant (master-detail).
- **Kit/Bundle Member** — child of the kit's APD (`kugo2p__ReferenceKitBundle__c`); member `kugo2p__ReferenceMemberProductService__c` (an APD).
- **Configuration Group / Option** — children of the configurable Product's APD (`kugo2p__AdditionalProductInfo__c` / `kugo2p__ConfigureProduct__c`); an option's `kugo2p__OptionProduct__c` is an APD.
- **Tiered Pricing** — `kugo2p__Product__c` (Product2) and `kugo2p__PriceBook__c`; Tiers are its children.
- **Account Pricing** — master-detail child of the Account; `kugo2p__ProductService__c` (an APD) or a filter; optional `kugo2p__Contract__c`.

## kuga_sub objects and fields (only when HAS_KUGA_SUB = true)

### Custom object

- `kuga_sub__Subscription__c` (`Name` `SUB-{0000000}`) — the subscription record linking an Order Service Line to a Contract. Its Contract link `kuga_sub__ContractNumber__c` is a Master-Detail; it can be re-parented (see Package versions), which is how Merge Contract moves Subscriptions.

### Fields kuga_sub adds to other objects

**Product2**

- `kuga_sub__Renewable__c` (Checkbox) — "Renewable" prefix on the Setup label, and a Renewal Opportunity on Order Release. Does NOT create a Subscription.
- `kuga_sub__Track__c` (Checkbox, label "Create Subscription") — a Subscription on Order Release for the product's **Order Service Lines** only.
- `kuga_sub__RenewalProduct__c` (Lookup Product2) — substitute product on renewals.
- `kuga_sub__UpliftRenewalPrice__c` (Checkbox) — apply the renewal price uplift.

Asset creation is not a Product2 flag: it is the APD picklist `kugo2p__CreateAsset__c` (`Per Line` / `Per Quantity`), used only for Order Product Lines, and kugo2p creates the Assets.

**OpportunityLineItem**

- `kuga_sub__Renew__c` (Checkbox) — **CRITICAL**: recurring vs one-time.
- `kuga_sub__ServiceTerm__c` (Number), `kuga_sub__UnitofTerm__c` (Picklist), `kuga_sub__ServiceTermBehavior__c` (Picklist), `kuga_sub__DateServiceEnd__c` (Date).
- `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__ARRForecast__c`, `kuga_sub__NonRecurringRevenue__c` (Currency, calculated).
- `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c`, `kuga_sub__ListAmount__c`, `kuga_sub__DiscountSalesPrice__c` (Formula Currency), `kuga_sub__EffectiveDiscount__c` (Formula Percent), `kuga_sub__LineTerm__c` (Formula Text), `kuga_sub__Service__c` (Formula Checkbox), `kuga_sub__UpliftRenewalPrice__c` (Formula Checkbox).
- `kuga_sub__NonUpliftSalesPrice__c` (Currency(12, 6)) — pre-uplift price.

**Opportunity**

- Roll-ups: `kuga_sub__MonthlyRecurringRevenue__c`, `kuga_sub__AnnualRecurringRevenueCommitted__c`, `kuga_sub__NonRecurringRevenue__c`, `kuga_sub__Amount__c` (SUM), `kuga_sub__DateRequired__c` (MIN), `kuga_sub__ServiceDateExpires__c` (MAX).
- Formulas: `kuga_sub__AnnualContractValueInitial__c` (ACV = NonRecurring + ARR), `kuga_sub__TotalContractValue__c`, `kuga_sub__AnnualRecurringRevenueForecast__c` (MRR × 12), `kuga_sub__ExpectedRevenue__c`, `kuga_sub__OpportunityAmount__c`, `kuga_sub__ContractEndDate__c`, `kuga_sub__ParentContractEndDate__c`.
- Editable: `kuga_sub__ParentContract__c` (Lookup Contract), `kuga_sub__ParentOrder__c` (Lookup Order), `kuga_sub__AutoEmailRenewalOrder__c`, `kuga_sub__AutoRenewedOrder__c` (Lookup Order), `kuga_sub__RenewalOrderAutoCreationDate__c`, `kuga_sub__RenewalPriceUpliftPercent__c`.

**kugo2p__SalesOrder__c (Order Release controls)**

- `kuga_sub__GenerateContract__c`, `kuga_sub__GenerateAsset__c`, `kuga_sub__GenerateSubscription__c`, `kuga_sub__GenerateRenewalOpportunity__c` (Checkboxes).
- `kuga_sub__ContractNumber__c` (Lookup Contract), `kuga_sub__ParentContract__c` (Lookup Contract), `kuga_sub__RenewalOpportunity__c` (Lookup Opportunity).
- `kuga_sub__ContractEndDate__c`, `kuga_sub__ParentContractEndDate__c`, `kuga_sub__RenewalEndDate__c` (Formulas).
- `kuga_sub__RenewableProductsCount__c`, `kuga_sub__RenewableServicesCount__c`, `kuga_sub__TrackableProductsCount__c`, `kuga_sub__TrackableServicesCount__c` (Roll-ups), `kuga_sub__ServiceDateExpires__c` (Roll-up MAX).
- `kuga_sub__UpdateContractContacts__c` (Multi-select) — which Contacts are copied to the Contract.

**kugo2p__SalesQuote__c**: `kuga_sub__ContractNumber__c` (Lookup Contract), `kuga_sub__ContractEndDate__c` (Formula).

**Lines**

- `kugo2p__SalesOrderServiceLine__c`: `kuga_sub__Renew__c` (revenue classification) and `kuga_sub__Track__c` ("Create Subscription" — a Subscription on Order Release). Order Service Lines never generate Assets.
- `kugo2p__SalesOrderProductLine__c`, `kugo2p__SalesQuoteProductLine__c`, `kugo2p__SalesQuoteServiceLine__c`: `kuga_sub__Renew__c`. Order Product Lines never generate Subscriptions.

**Contract**

- Roll-ups: `kuga_sub__AnnualRecurringRevenue__c`, `kuga_sub__MonthlyRecurringRevenue__c` (SUM of Subscriptions), `kuga_sub__TotalSubscriptionAmount__c`, `kuga_sub__TotalSubscriptionCount__c`, `kuga_sub__TotalSubscriptionQuantity__c`, `kuga_sub__SubscriptionStartDate__c` (MIN), `kuga_sub__SubscriptionEndDate__c` (MAX).
- Formulas: `kuga_sub__Effective__c` (currently active), `kuga_sub__AnnualRecurringRevenueForecast__c`, `kuga_sub__ContractRenewalNoticeDate__c`, `kuga_sub__SendRenewalNoticeToday__c`.
- Editable: `kuga_sub__RenewalOpportunity__c` (the Current Renewal Opportunity), `kuga_sub__RenewalTerm__c`, `kuga_sub__AutoEmailRenewalNotice__c`, `kuga_sub__AutoEmailRenewalOrder__c`, `kuga_sub__Pricebook2Id__c`, `kuga_sub__ContactBuying__c`, `kuga_sub__ContactBilling__c`, `kuga_sub__ContactShipping__c`, `kuga_sub__Expanded__c`, `kuga_sub__LastRenewalNoticeSentDate__c`.

**Asset**: `kuga_sub__ContractNumber__c` (Lookup Contract), `kuga_sub__ParentSubscription__c` (Lookup Subscription), `kuga_sub__ParentLine__c` (Formula). kugo2p's own Asset fields: `kugo2p__OrderProductLine__c` (the Order Product Line it came from), `kugo2p__ShipmentLine__c`, `kugo2p__PurchasePrice__c` (`Currency(12, 6)`), and formulas `kugo2p__OrderNumber__c`, `kugo2p__ShipmentDate__c`, `kugo2p__ShipmentStatus__c`. kuga_sub stamps `ContractNumber__c` (and `ParentSubscription__c` where the Asset sits under a Subscription) on the Assets kugo2p creates.
