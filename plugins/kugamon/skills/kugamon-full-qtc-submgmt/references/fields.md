# Field Reference

Reference for the `kugamon-full-qtc-submgmt` skill. Package versions are listed only in SKILL.md (Package Detection → Package versions). Before writing to an object, run `get_object_fields` (or a describe) to confirm what the org has.

## Opportunity

| Field | Type | Notes |
|---|---|---|
| `Name`, `StageName`, `CloseDate` | standard | Required |
| `AccountId` | Lookup(Account) | Needed for Kugamon |
| `Pricebook2Id` | Lookup(Pricebook2) | Needed for Opportunity Products; must be active for Generate Quote |
| `Amount` | Currency | With products, the products' total. Don't use for pipeline when kuga_sub is installed |
| `RecordTypeId` | Lookup(RecordType) | With kuga_sub: New, Expansion, Renewal (`record-types-and-names.md`) |
| `kugo2p__PrimaryQuote__c`, `kugo2p__PrimaryOrder__c` | Lookups | Set to the Primary Quote / Order |
| `kugo2p__SyncSource__c` | | "Primary Quote" or "Primary Order" — which record last synced the products or Amount |

kuga_sub fields on Opportunity: `object-model.md`. Amount fields and what they mean: `amounts-and-renew.md`.

## OpportunityLineItem (Opportunity Product)

Required: `OpportunityId`, `PricebookEntryId` (not `Product2Id`), `Quantity`. Standard optional: `UnitPrice`, `ServiceDate`, `Discount`, `Description`. The Opportunity must have a Price Book and the Product a Price Book Entry in it.

| Field | Type | Notes |
|---|---|---|
| `UnitPrice` | Currency | Shown at 2 decimals; stores what the Quote sync writes, unrounded |
| `kugo2p__ParentLine__c` | Lookup(Opportunity Product) | Kit header or parent line (written by sync) |
| `kugo2p__ConfigurationOption__c` | Lookup | Written by sync |
| `kugo2p__AdditionalChargeCredit__c` | Lookup | Charge/Credit lines only |

**kuga_sub fields** (HAS_KUGA_SUB only):

| Field | Editable | Notes |
|---|---|---|
| `kuga_sub__Renew__c` | Yes | **CRITICAL**: recurring (`true`) or one-time (`false`, default) |
| `kuga_sub__ServiceTerm__c` | Yes | Term length, e.g. 12 |
| `kuga_sub__UnitofTerm__c` | Yes | Picklist, e.g. Month, Year |
| `kuga_sub__ServiceTermBehavior__c` | Yes | Picklist |
| `kuga_sub__NonUpliftSalesPrice__c` | Yes | Pre-uplift price, `Currency(12, 6)` |
| `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__ARRForecast__c` | Calculated | MRR/ARR when Renew = true, else 0 |
| `kuga_sub__NonRecurringRevenue__c` | Calculated | Line total when Renew = false, else 0. Writing it directly is overridden |
| `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c`, `kuga_sub__ListAmount__c` | Formula | Amounts |
| `kuga_sub__DateServiceEnd__c` | Calculated | ServiceDate + term |
| `kuga_sub__Service__c` | Formula | Is the line a service |
| `kuga_sub__LineTerm__c` | Formula | e.g. "12 Month" |
| `kuga_sub__DiscountSalesPrice__c`, `kuga_sub__EffectiveDiscount__c` | Formula | Discounted price and effective discount % |
| `kuga_sub__UpliftRenewalPrice__c` | Formula | Uplift flag from the Product |

## Quote (`kugo2p__SalesQuote__c`)

**Createable** (everything else is filled from the AAD and Kugamon Settings):

| Field | Type | Notes |
|---|---|---|
| `RecordTypeId` | Lookup | Only when the org has Quote record types |
| `kugo2p__Account__c` | Lookup(Account) | Required; can't be changed later |
| `kugo2p__Opportunity__c` | Lookup(Opportunity) | |
| `kugo2p__QuoteName__c` | Text(255) | Defaults to the Opportunity name |
| `kugo2p__Pricebook2Id__c` | Lookup(Price Book) | Defaults to the Opportunity's, else the AAD's, else the Kugamon Settings default. Kept in step with `kugo2p__PriceBookName__c` |
| `kugo2p__ContactBuying__c`, `kugo2p__ContactBilling__c`, `kugo2p__ContactShipping__c` | Lookup(Contact) | Default from the AAD |
| `kugo2p__IsPrimary__c` | Checkbox | The first Quote on an Opportunity becomes Primary |
| `kugo2p__DateOfferValidThrough__c` | Date | Expiration. Defaults to today + Quote Valid In Days; blank setting = today |
| `kugo2p__DiscountPercent__c` | Percent | Header discount; adds to each line's Line Discount % (they don't compound) |
| `kugo2p__CustomerPONumber__c`, `kugo2p__CustomerPODate__c` | | |
| `kugo2p__IsRestricted__c` | Checkbox | A Restricted Quote can't generate an Order or be sent |

**Status**: `kugo2p__RecordStatus__c` (Text 30: `Draft`, `Sent`, `Won`, `Lost`, `Cancelled`) holds the value; `kugo2p__Status__c` is a formula with action links. `kugo2p__DateSent__c`, `kugo2p__DateClosed__c` are stamped.

**Auto-managed (never set)**: `Name` (`SQ-{YYMMDD}-{0000000}`), `kugo2p__Status__c`, `kugo2p__OnlineApprovalKey__c` (set by Kugamon), and the totals. Formulas: Total, Subtotal, Net, Product and Service Total and Subtotal, Total Discount, Total Tax and Total VAT/GST Amounts. Roll-ups: Product, Service and Charge/Credit Amounts. Stored but recalculated by the line triggers: Product and Service Discount, Tax and VAT/GST Amounts.

kuga_sub on Quote: `kuga_sub__ContractNumber__c` (Lookup Contract), `kuga_sub__ContractEndDate__c` (formula).

## Quote lines

`kugo2p__SalesQuoteProductLine__c` and `kugo2p__SalesQuoteServiceLine__c` — master-detail children of the Quote (`kugo2p__SalesQuote__c`).

| Field | Product Line | Service Line | Notes |
|---|---|---|---|
| Item | `kugo2p__Product__c` | `kugo2p__Service__c` | Lookup to the **APD**, not Product2. Can't be changed once saved |
| Name | `kugo2p__ProductName__c` | `kugo2p__ServiceName__c` | |
| `kugo2p__Quantity__c` | ✓ | ✓ | Number(14, 4) |
| `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__TierPrice__c` | ✓ | ✓ | `Currency(12, 6)`; blank Sales Price via the API = List Price |
| `kugo2p__NonUpliftSalesPrice__c`, `kugo2p__UpliftPrice__c` | | ✓ | `Currency(12, 6)` |
| `kugo2p__LineDiscountPercent__c` | ✓ | ✓ | Percent |
| `kugo2p__DiscountSalesPrice__c` | ✓ | ✓ | "Effective Price" formula, rounded to the Product's Price Scale |
| `kugo2p__TotalAmount__c`, `kugo2p__NetAmount__c` | ✓ | ✓ | Formulas, 2 decimals |
| Dates | `kugo2p__DateRequired__c`, `kugo2p__DateAvailableToPromise__c` | `kugo2p__DateServiceStart__c`, `kugo2p__DateServiceEnd__c` | |
| Term | | `kugo2p__ServiceTerm__c`, `kugo2p__ServiceTermBehavior__c`, `kugo2p__UnitofTerm__c` (formula) | |
| `kugo2p__Line__c` | ✓ | ✓ | Formula line number |
| `kugo2p__LineDescription__c` | ✓ | ✓ | Text(255) |
| `kugo2p__LineGroup__c` | ✓ | ✓ | Lookup(Quote Line Group) |
| `kugo2p__ParentProductLine__c`, `kugo2p__ParentServiceLine__c` | ✓ | ✓ | Parent line (kits, configured products) |
| `kugo2p__OpportunityLineItem__c` | ✓ | ✓ | Its synced Opportunity Product |
| `kuga_sub__Renew__c` | ✓ | ✓ | HAS_KUGA_SUB only |

`Name` on both is an auto-number sequence — don't set it.

## Order (`kugo2p__SalesOrder__c`)

| Field | Notes |
|---|---|
| `kugo2p__Account__c`, `kugo2p__Opportunity__c`, `kugo2p__SalesQuote__c` | Account can't be changed once set |
| `kugo2p__AdditionalAccountDetail__c` | The AAD |
| `kugo2p__Pricebook2Id__c` / `kugo2p__PriceBookName__c` | Price Book lookup and name |
| `kugo2p__ContactBuying__c`, `kugo2p__ContactBilling__c`, `kugo2p__ContactShipping__c` | All three required on a Released Order |
| `kugo2p__Warehouse__c`, `kugo2p__Carrier__c` | Warehouse required for most status changes |
| `kugo2p__OrderName__c`, `kugo2p__OrderDate__c` | Name and order date |
| `kugo2p__PaymentMethod__c`, `kugo2p__PaymentTerms__c`, `kugo2p__DeliveryTerms__c`, `kugo2p__DiscountPercent__c` | From the AAD |
| `kugo2p__InvoiceSchedule__c` | How often it's invoiced; `kugo2p__BillingFrequency__c` is the formula |
| `kugo2p__IsPrimary__c` | One Primary Order per Opportunity and per Quote |
| `kugo2p__IsRestricted__c` | A Restricted Order can't be released |
| `kugo2p__RecordStatus__c` | `Draft`, `Sent`, `Approved`, `Released`, `Cancelled` — writing it runs the status action |
| `kugo2p__Status__c` | Formula with action links |
| `kugo2p__DateSent__c`, `kugo2p__DateApproved__c`, `kugo2p__DateReleased__c`, `kugo2p__DateCancelled__c` | Stamped once |
| `kugo2p__UpdateAdditionalAccountInfoContacts__c` | Multi-select: which Contacts are copied to the AAD on Release |
| `kugo2p__AppliedPaymentAmount__c`, `kugo2p__BalanceDueAmount__c` | Formulas |

`Name` is `SO-{YYMMDD}-{0000000}`. Totals follow the Quote pattern. kuga_sub Order Release controls: `object-model.md`.

## Order lines

`kugo2p__SalesOrderProductLine__c` and `kugo2p__SalesOrderServiceLine__c` — master-detail children of the Order. Same price, discount, date and term fields as Quote lines (the `Product__c` / `Service__c` lookups point to the APD), plus:

| Field | On | Notes |
|---|---|---|
| `kugo2p__RecordStatus__c` | both | Copied from the Order on status changes |
| `kugo2p__Warehouse__c`, `kugo2p__ContactShipping__c` | Product Lines | Required on every Product Line for most status changes |
| `kugo2p__ShipmentScheduledQuantity__c`, `kugo2p__QuantityShippedToDate__c`, `kugo2p__DateLastShipment__c` | Product Lines | Updated from Shipment Lines |
| `kugo2p__Contract__c` | Service Lines | The kuga_sub Contract |
| `kugo2p__CancelDate__c`, `kugo2p__CancelledQuantity__c`, `kugo2p__CancelReason__c` | both | Cancel fields (below) |
| `kuga_sub__Renew__c` | both | Revenue classification; does NOT create Subscriptions |
| `kuga_sub__Track__c` ("Create Subscription") | Service Lines | A Subscription on Order Release |

Asset creation is driven by the APD picklist `kugo2p__CreateAsset__c`, only on Order Product Lines, and is performed by kugo2p.

### Order Service Line cancel fields (written by kuga_sub on Subscription cancel — see Package versions)

| Field | Type | Set by Cancel Subscription |
|---|---|---|
| `kugo2p__CancelDate__c` | Date | Today |
| `kugo2p__CancelledQuantity__c` | Number(10, 2) | The Subscription's quantity |
| `kugo2p__CancelReason__c` | Picklist, unrestricted. Package values: `Error`, `Price`, `Availablity` (sic), `Promise Date`, `Payment Problems`, `Contract Terminated`; orgs may add their own | `Subscription Cancelled` |

Quote to Cash cancels the line's `Scheduled` invoice lines and service delivery schedules when `kugo2p__CancelReason__c` changes from blank to a value — by any route, not only through kuga_sub. `Subscription Cancelled` may not appear in the picklist's list of values in every Quote to Cash version; because the picklist is unrestricted the value still saves. Filter on it with an exact string match.

## Shipment and Service Delivery

| Object | Key fields |
|---|---|
| `kugo2p__Shipment__c` | `Name` (`SHP-{0000000}`), `kugo2p__SalesOrder__c`, `kugo2p__RecordStatus__c` (`Scheduled`, `Shipped`, `Delivered`, `On Hold`, `Cancelled`), `kugo2p__Status__c` (formula), `kugo2p__ShipmentDate__c`, `kugo2p__DateEstimatedDelivery__c`, `kugo2p__Carrier__c`, `kugo2p__Warehouse__c`, `kugo2p__ContactShipping__c`, `kugo2p__TrackingNumber__c`, `kugo2p__ShippingCost__c` |
| `kugo2p__ShipmentLine__c` | `kugo2p__Shipment__c`, `kugo2p__SalesOrderProductLine__c`, `kugo2p__Product__c` (APD), `kugo2p__Quantity__c`, `kugo2p__RecordStatus__c`, `kugo2p__InvoicedQuantity__c` |
| `kugo2p__ServiceDeliverySchedule__c` | `kugo2p__SalesOrder__c`, `kugo2p__SalesOrderServiceLine__c`, `kugo2p__DateServiceStart__c`, `kugo2p__DateServiceDelivered__c`, `kugo2p__Quantity__c`, `kugo2p__ServiceAmount__c`, `kugo2p__RecordStatus__c` (`Scheduled` on Release, `Cancelled` on cancel) |

## Invoice (`kugo2p__KugamonInvoice__c`) and lines

| Field | Notes |
|---|---|
| `Name` | `INV-{YYMMDD}-{0000000}` |
| `kugo2p__Account__c`, `kugo2p__SalesOrder__c`, `kugo2p__AdditionalAccountDetail__c` | |
| `kugo2p__InvoiceDate__c`, `kugo2p__InvoiceDueDate__c` | Due = Invoice Date + Days Till Payment Due |
| `kugo2p__RecordStatus__c` | `Unsent`, `Sent`, `Cancelled` |
| `kugo2p__IsPosted__c`, `kugo2p__DatePosted__c` | Posted to accounting |
| `kugo2p__PaymentStatus__c` | Formula: `Unpaid`, `Partially Paid`, `Paid`, blank |
| `kugo2p__TotalAmount__c`, `kugo2p__NetAmount__c`, `kugo2p__AppliedPaymentAmount__c`, `kugo2p__BalanceDueAmount__c` | Formulas, 2 decimals. Balance Due = Net − Applied |
| `kugo2p__ContactBilling__c`, `kugo2p__ContactShipping__c`, `kugo2p__PaymentTerms__c`, `kugo2p__PaymentMethod__c` | |

`kugo2p__KugamonInvoiceLine__c`: `kugo2p__Invoice__c` (its Invoice), `kugo2p__SalesOrder__c`, `kugo2p__SalesOrderProductLine__c` / `kugo2p__SalesOrderServiceLine__c`, `kugo2p__ShipmentLine__c` / `kugo2p__ServiceDeliverySchedule__c`, `kugo2p__ProductServiceName__c`, `kugo2p__Quantity__c`, `kugo2p__SalesPrice__c` (formula — reads the Order line's price), `kugo2p__DiscountSalesPrice__c` (Effective Price formula), `kugo2p__LineAmount__c`, `kugo2p__NetAmount__c`, `kugo2p__TotalAmount__c`, `kugo2p__TaxAmount__c`, `kugo2p__VATAmount__c`, `kugo2p__AppliedPaymentAmount__c`, `kugo2p__BalanceDueAmount__c`, `kugo2p__RecordStatus__c`. Amounts are 2 decimals.

`kugo2p__InvoiceSchedule__c`: `Name`, `kugo2p__SchedulePeriod__c` (blank = one-time), `kugo2p__ServicesInvoicedAtServiceStart__c`, `kugo2p__ExcludeProductsfromSchedule__c`, `kugo2p__InvoiceDateBasis__c` (formula showing the Kugamon Settings value).

`kugo2p__OrderInvoiceRelationship__c`: `kugo2p__SalesOrder__c`, `kugo2p__Invoice__c` (both master-detail), plus formula copies of each side's amount, date and status.

## Payment (`kugo2p__PaymentX__c`) and Applied Payment

| Field | Notes |
|---|---|
| `Name` | Set by Kugamon: "Payment for Order SO-…" / "Payment for Invoice INV-…" (refunds add " (Refund)") |
| `kugo2p__SalesOrder__c` or `kugo2p__Invoice__c` | One, never both |
| `kugo2p__Account__c`, `kugo2p__Contact__c`, `kugo2p__Opportunity__c` | |
| `kugo2p__Amount__c`, `kugo2p__Date__c` | 2 decimals |
| `kugo2p__Status__c` | `Completed`, `Authorized`, `Scheduled`, `Pending`, `Declined`, `Error`, `Expired`, `Voided`, `Reversed` |
| `kugo2p__Transaction_Type__c` | `Payment`, `Authorization`, `Refund` |
| `kugo2p__Parent_Transaction__c` | The original Payment, on a refund |
| `kugo2p__Payment_Type__c`, `kugo2p__Payment_Processor__c`, `kugo2p__Processor_Connection__c`, `kugo2p__Payment_Method__c`, `kugo2p__Payment_Profile__c` | How it was paid |
| `kugo2p__Transaction_Id__c`, `kugo2p__Memo__c` | Gateway reference and message |
| `kugo2p__Is_Test_Transaction__c` | Test-mode payment |

`kugo2p__AppliedPayment__c`: `kugo2p__Payment__c` (master-detail), `kugo2p__AmountApplied__c`, `kugo2p__Invoice__c`, `kugo2p__InvoiceLine__c`, `kugo2p__InvoiceAdditionalChargeCredit__c`, `kugo2p__SalesOrder__c`, `kugo2p__SalesOrderProductLine__c`, `kugo2p__SalesOrderServiceLine__c`, `kugo2p__SalesOrderAdditionalChargeCredit__c`; `kugo2p__Status__c`, `kugo2p__TransactionType__c`, `kugo2p__TransactionId__c` are formulas reading the Payment. Applications are rebuilt by Kugamon — don't write them.

`kugo2p__Processor_Connection__c`: record type = gateway; `kugo2p__Default_Connection__c` (one per org), `kugo2p__Assigned_to_Terminal__c`, `kugo2p__Test_Mode__c`, `kugo2p__Site_Checkout__c`. `kugo2p__Payment_Profile__c` (Recurring Charge): `kugo2p__Account__c`, `kugo2p__SalesOrder__c`, `kugo2p__SalesOrderServiceLine__c`, `kugo2p__Amount__c`, `kugo2p__Frequency__c`, `kugo2p__Next_Payment_Date__c`, `kugo2p__Subscription_Status__c`.

## Subscription (`kuga_sub__Subscription__c`)

| Field | Type | Notes |
|---|---|---|
| `Name` | Auto Number | `SUB-{0000000}` |
| `kuga_sub__Account__c` | Master-Detail(Account) | |
| `kuga_sub__ContractNumber__c` | Master-Detail(Contract) | Re-parentable (see Package versions). Deleting a Contract deletes its Subscriptions, which is why Merge Contract moves them first |
| `kuga_sub__Order__c`, `kuga_sub__OrderServiceLine__c` | Lookups | Source Order and Order Service Line |
| `kuga_sub__Service__c` | Lookup(Product2) | |
| `kuga_sub__Quantity__c` | Number(16, 2) | |
| `kuga_sub__PurchasePrice__c` | Currency(12, 6) | 6 decimals (see Package versions) |
| `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c` | Currency(16, 2) | |
| `kuga_sub__StartDate__c`, `kuga_sub__EndDate__c` | Date | |
| `kuga_sub__Status__c` | Restricted picklist | `Purchased` (set on Order Release), `Provisioned`, `Cancelled`. Setting `Cancelled` runs Workflow 4a |
| `kuga_sub__Active__c` | Formula | Active now (needs an Effective Contract) |
| `kuga_sub__IsActive__c` | Checkbox | Resynced from `Active__c` by the scheduled job; drives the Contract's ARR/MRR roll-ups |
| `kuga_sub__Renew__c`, `kuga_sub__ServiceTerm__c`, `kuga_sub__UnitofTerm__c`, `kuga_sub__OrderRecordType__c` | Formulas | |
| `kuga_sub__ParentAsset__c`, `kuga_sub__ParentSubscription__c` | Lookups | Kit or bundle members point at their parent |

Contract, Asset and Product2 kuga_sub fields: `object-model.md`.

## Additional Product Info (`kugo2p__AdditionalProductDetail__c`)

| Area | Fields |
|---|---|
| Identity | `kugo2p__ReferenceProduct__c` (Product2; can't be changed), `kugo2p__ProductCode__c`, `kugo2p__ProductFamily__c`, `kugo2p__Active__c`, `kugo2p__Hide__c` |
| Type | `kugo2p__Service__c`, `kugo2p__DefaultServiceTerm__c`, `kugo2p__UnitofTerm__c`, `kugo2p__ServiceTermBehavior__c`, `kugo2p__DisableShipments__c`, `kugo2p__CreateAsset__c` (`Per Line` / `Per Quantity`) |
| Pricing | `kugo2p__PricingMethod__c` (List, Cost), `kugo2p__StandardPrice__c` (2 decimals), `kugo2p__UnitCost__c`, `kugo2p__DefaultCostMarkupPercent__c`, `kugo2p__PriceScale__c`, `kugo2p__QuantityScale__c`, `kugo2p__ServiceTermScale__c`, `kugo2p__DefaultLineDiscount__c` |
| Locks and limits | `kugo2p__LockSalesPrice__c`, `kugo2p__LockLineDiscount__c`, `kugo2p__LockQuantity__c`, `kugo2p__LockServiceTerm__c`, `kugo2p__LockLineDescription__c`, `kugo2p__EditListPrice__c`, `kugo2p__EditProductName__c`, `kugo2p__DefaultQuantity__c`, `kugo2p__MinimumQuantity__c`, `kugo2p__MaximumQuantity__c` |
| Kits and configuration | `kugo2p__KitBundle__c`, `kugo2p__KitBundlePricing__c`, `kugo2p__PrintKitBundleMembersonPDFs__c`, `kugo2p__ConfigurationMethod__c`, `kugo2p__ConfigurationType__c`, `kugo2p__ConfigurationAction__c` |
| Tax | `kugo2p__TaxableSales__c`, `kugo2p__TaxableUsage__c`, `kugo2p__TaxableExcise__c`, `kugo2p__TaxableVAT__c`, `kugo2p__VAT__c` |
| Sync | `kugo2p__ExcludefromOpportunitySync__c` |

## Quote Chat objects

### `kugo2p__ChatParticipant__c` — one per person per Quote

| Field | Type | Writable | Notes |
|---|---|---|---|
| `Name` | Auto Number | No | `CP-{0000000}` |
| `kugo2p__QuoteNumber__c` | Lookup(Quote) | Yes | The Quote; deleted with it |
| `kugo2p__User__c` | Lookup(User) | Yes | For an internal user |
| `kugo2p__Contact__c` | Lookup(Contact) | Yes | For a customer contact |
| `kugo2p__ParticipantKey__c` | Text(255), External ID, unique | Yes | `{Quote Id}-{User or Contact Id}` |
| `kugo2p__Type__c` | Formula (Text) | No | `Internal User`, `Portal / Community User` or `Contact` |
| `kugo2p__ParticipantName__c`, `kugo2p__Email__c`, `kugo2p__Title__c`, `kugo2p__Participant__c` | Formulas | No | From the User or Contact |
| `kugo2p__LastReadMessage__c` | Lookup(Chat Message) | Yes | Read pointer — update only your own |
| `kugo2p__DateLastRead__c` | Date/Time | Yes | When the participant last read the thread |

### `kugo2p__ChatMessage__c` — one per message

| Field | Type | Writable | Notes |
|---|---|---|---|
| `Name` | Auto Number | No | 9-digit sequence, e.g. `000000165` |
| `kugo2p__ChatParticipant__c` | Master-Detail(Chat Participant) | On insert | The author |
| `kugo2p__Body__c` | Rich Text Area | Yes | HTML, shown as HTML on the Quote and the Online Quote |
| `kugo2p__BodyPreview__c` | Text(255) | Yes | Plain text, 252 characters plus "..." when longer. Integrations must set it |
| `kugo2p__ParticipantName__c` | Formula | No | The author's name |
