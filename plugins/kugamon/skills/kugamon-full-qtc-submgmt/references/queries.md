# Query Library

Reference for the `kugamon-full-qtc-submgmt` skill. Replace the `<…>` placeholders. Queries that use `kuga_sub__*` fields need `HAS_KUGA_SUB = true`. Package versions are listed only in SKILL.md (Package Detection → Package versions).

## Setup and settings

```sql
-- Kugamon Settings: sync and expiration settings
SELECT Id, kugo2p__SyncPrimaryQuotetoOpportunityLines__c, kugo2p__SyncPrimaryOrdertoOpportunityLines__c,
       kugo2p__SyncPrimaryQuotetoOpportunityAmount__c, kugo2p__SyncPrimaryOrdertoOpportunityAmount__c,
       kugo2p__UpdateOpportunityAmountBasis__c, kugo2p__SalesQuoteValidOfferInDays__c,
       kugo2p__OnlineOrderValidInDays__c
FROM kugo2p__KugamonSetting__c LIMIT 1

-- Record types (never hardcode the Ids)
SELECT Id, Name, DeveloperName, SobjectType
FROM RecordType
WHERE SobjectType IN ('kugo2p__SalesQuote__c', 'kugo2p__SalesOrder__c', 'kugo2p__Payment_Profile__c',
                      'kugo2p__Processor_Connection__c', 'kugo2p__Payment_Method__c', 'Opportunity')
  AND IsActive = true
ORDER BY SobjectType, Name
```

Application Settings (`kugo2p__Application_Settings__c` — Disable Invoices, Disable Shipments, Enable Online Chat) and Kugamon's other custom settings are **protected**: in a customer's org they can't be queried, and Setup doesn't list them. Infer them from behaviour (did Release build Invoices?) or ask the admin; chat is switched on from the Kugamon Configuration page, and Disable Invoices is changed only by Kugamon Support.

## Accounts and the AAD

```sql
-- Exactly one AAD per Account, and the defaults it will give a Quote or Order
SELECT Id, Name, kugo2p__ContactBuying__c, kugo2p__ContactBilling__c, kugo2p__ContactShipping__c,
       kugo2p__PriceBookName__c, kugo2p__PaymentMethod__c, kugo2p__PaymentTerms__c,
       kugo2p__DaysTillPaymentDue__c, kugo2p__Warehouse__c, kugo2p__InvoiceSchedule__c,
       kugo2p__TaxLocation__c, kugo2p__TaxableSales__c, kugo2p__TaxableVAT__c, kugo2p__VATRegistration__c
FROM kugo2p__AdditionalAccountDetail__c
WHERE kugo2p__Account__c = '<account_id>'

-- Address sources: Account billing/shipping, Contacts' Mailing (billing, buying) and Other (shipping)
SELECT Id, Name, BillingStreet, BillingCity, BillingState, BillingPostalCode, BillingCountry,
       ShippingStreet, ShippingCity
FROM Account WHERE Id = '<account_id>'

SELECT Id, Name, Email, MailingStreet, MailingCity, OtherStreet, OtherCity
FROM Contact WHERE AccountId = '<account_id>' LIMIT 20
```

## Quotes

```sql
-- The Opportunity, with every amount field (HAS_KUGA_SUB = true)
SELECT Id, Name, AccountId, Amount, StageName, CloseDate, Pricebook2Id, RecordType.Name,
       kugo2p__PrimaryQuote__c, kugo2p__SyncSource__c, kuga_sub__Amount__c,
       kuga_sub__MonthlyRecurringRevenue__c, kuga_sub__AnnualContractValueInitial__c,
       kuga_sub__TotalContractValue__c, kuga_sub__AnnualRecurringRevenueCommitted__c,
       kuga_sub__NonRecurringRevenue__c
FROM Opportunity WHERE Id = '<opportunity_id>'

-- The Opportunity (HAS_KUGA_SUB = false)
SELECT Id, Name, AccountId, Amount, StageName, CloseDate, Pricebook2Id, RecordType.Name,
       kugo2p__PrimaryQuote__c, kugo2p__SyncSource__c
FROM Opportunity WHERE Id = '<opportunity_id>'

-- Opportunity Products: each needs a Price Book Entry, in an active Price Book
SELECT Id, Product2.Name, PricebookEntryId, PricebookEntry.IsActive, Quantity, UnitPrice, ServiceDate
FROM OpportunityLineItem WHERE OpportunityId = '<opportunity_id>'

-- Existing Quotes on the Opportunity
SELECT Id, Name, kugo2p__QuoteName__c, kugo2p__RecordStatus__c, kugo2p__IsPrimary__c,
       kugo2p__TotalAmount__c, kugo2p__DateOfferValidThrough__c
FROM kugo2p__SalesQuote__c WHERE kugo2p__Opportunity__c = '<opportunity_id>'

-- A Quote's header
SELECT Id, Name, kugo2p__QuoteName__c, kugo2p__RecordStatus__c, kugo2p__IsPrimary__c,
       kugo2p__PriceBookName__c, kugo2p__DiscountPercent__c, kugo2p__SubtotalAmount__c,
       kugo2p__TotalAmount__c, kugo2p__NetAmount__c, kugo2p__DateOfferValidThrough__c
FROM kugo2p__SalesQuote__c WHERE Id = '<quote_id>'

-- Its Product Lines
SELECT Id, kugo2p__Line__c, kugo2p__ProductName__c, kugo2p__Product__r.kugo2p__ReferenceProduct__c,
       kugo2p__Quantity__c, kugo2p__ListPrice__c, kugo2p__SalesPrice__c, kugo2p__LineDiscountPercent__c,
       kugo2p__DiscountSalesPrice__c, kugo2p__TotalAmount__c, kugo2p__DateRequired__c
FROM kugo2p__SalesQuoteProductLine__c
WHERE kugo2p__SalesQuote__c = '<quote_id>' ORDER BY kugo2p__Line__c

-- Its Service Lines
SELECT Id, kugo2p__Line__c, kugo2p__ServiceName__c, kugo2p__Quantity__c, kugo2p__ListPrice__c,
       kugo2p__SalesPrice__c, kugo2p__DiscountSalesPrice__c, kugo2p__TotalAmount__c,
       kugo2p__DateServiceStart__c, kugo2p__DateServiceEnd__c, kugo2p__ServiceTerm__c
FROM kugo2p__SalesQuoteServiceLine__c
WHERE kugo2p__SalesQuote__c = '<quote_id>' ORDER BY kugo2p__Line__c

-- The latest Sync Task on a Quote (only the latest run's Task is kept)
SELECT Id, Subject, Description, Priority, CreatedDate
FROM Task WHERE WhatId = '<quote_id>' AND Subject LIKE '%Opportunity Sync process%'
ORDER BY CreatedDate DESC LIMIT 1
```

## Orders

```sql
SELECT Id, Name, kugo2p__OrderName__c, kugo2p__Account__r.Name, kugo2p__RecordStatus__c,
       kugo2p__IsPrimary__c, kugo2p__TotalAmount__c, kugo2p__OrderDate__c, kugo2p__SalesQuote__r.Name,
       kugo2p__DateApproved__c, kugo2p__DateReleased__c
FROM kugo2p__SalesOrder__c WHERE kugo2p__Opportunity__c = '<opportunity_id>'

-- Ready to change status? Contacts, Warehouse and Restricted on the Order …
SELECT Id, Name, kugo2p__RecordStatus__c, kugo2p__IsRestricted__c, kugo2p__Warehouse__c,
       kugo2p__ContactBuying__c, kugo2p__ContactBilling__c, kugo2p__ContactShipping__c,
       kugo2p__AdditionalAccountDetail__r.kugo2p__PaymentMethod__c,
       kugo2p__AdditionalAccountDetail__r.kugo2p__PaymentTerms__c,
       kugo2p__AdditionalAccountDetail__r.kugo2p__DaysTillPaymentDue__c
FROM kugo2p__SalesOrder__c WHERE Id = '<order_id>'

-- … and Warehouse and Shipping Contact on every Product Line
SELECT Id, kugo2p__ProductName__c, kugo2p__Warehouse__c, kugo2p__ContactShipping__c, kugo2p__RecordStatus__c
FROM kugo2p__SalesOrderProductLine__c WHERE kugo2p__SalesOrder__c = '<order_id>'

SELECT Id, kugo2p__ServiceName__c, kugo2p__RecordStatus__c, kugo2p__DateServiceStart__c,
       kugo2p__DateServiceEnd__c, kugo2p__CancelDate__c
FROM kugo2p__SalesOrderServiceLine__c WHERE kugo2p__SalesOrder__c = '<order_id>'
```

## Order Release

```sql
-- What Release built (kugo2p)
SELECT Id, Name, Product2.Name, Quantity, Status, kugo2p__PurchasePrice__c,
       kugo2p__OrderProductLine__c, kugo2p__ShipmentLine__c
FROM Asset WHERE kugo2p__OrderProductLine__r.kugo2p__SalesOrder__c = '<order_id>'

SELECT Id, Name, kugo2p__RecordStatus__c, kugo2p__ShipmentDate__c, kugo2p__Carrier__r.Name, kugo2p__TrackingNumber__c
FROM kugo2p__Shipment__c WHERE kugo2p__SalesOrder__c = '<order_id>'

SELECT Id, kugo2p__Invoice__r.Name, kugo2p__InvoiceDate__c, kugo2p__InvoiceAmount__c, kugo2p__InvoiceStatus__c
FROM kugo2p__OrderInvoiceRelationship__c WHERE kugo2p__SalesOrder__c = '<order_id>'

-- The Contract created from the Order (kuga_sub)
SELECT Id, ContractNumber, Status, StartDate, EndDate, kuga_sub__Effective__c,
       kuga_sub__AnnualRecurringRevenue__c, kuga_sub__MonthlyRecurringRevenue__c, kuga_sub__TotalSubscriptionCount__c
FROM Contract WHERE Id IN (SELECT kuga_sub__ContractNumber__c FROM kugo2p__SalesOrder__c WHERE Id = '<order_id>')

-- Subscriptions from the Order
SELECT Id, Name, kuga_sub__Service__r.Name, kuga_sub__Quantity__c, kuga_sub__PurchasePrice__c,
       kuga_sub__MRR__c, kuga_sub__ARR__c, kuga_sub__StartDate__c, kuga_sub__EndDate__c,
       kuga_sub__Status__c, kuga_sub__ContractNumber__r.ContractNumber
FROM kuga_sub__Subscription__c WHERE kuga_sub__Order__c = '<order_id>'

-- Assets linked to the Order's Contract
SELECT Id, Name, Product2.Name, Quantity, Status,
       kuga_sub__ContractNumber__r.ContractNumber, kuga_sub__ParentSubscription__r.Name
FROM Asset WHERE kuga_sub__ContractNumber__c IN (SELECT kuga_sub__ContractNumber__c FROM kugo2p__SalesOrder__c WHERE Id = '<order_id>')

-- The Renewal Opportunity
SELECT Id, Name, StageName, CloseDate, kuga_sub__Amount__c,
       kuga_sub__ParentContract__r.ContractNumber, kuga_sub__ParentOrder__r.Name
FROM Opportunity WHERE kuga_sub__ParentOrder__c = '<order_id>'
```

## Contracts and Subscriptions

```sql
-- Active Contracts for an Account
SELECT Id, ContractNumber, Account.Name, Status, StartDate, EndDate, kuga_sub__Effective__c,
       kuga_sub__AnnualRecurringRevenue__c, kuga_sub__MonthlyRecurringRevenue__c,
       kuga_sub__TotalSubscriptionCount__c, kuga_sub__SubscriptionStartDate__c,
       kuga_sub__SubscriptionEndDate__c, kuga_sub__RenewalOpportunity__r.Name
FROM Contract WHERE kuga_sub__Effective__c = true AND AccountId = '<account_id>'

-- Subscriptions on a Contract
SELECT Id, Name, kuga_sub__Service__r.Name, kuga_sub__Quantity__c, kuga_sub__PurchasePrice__c,
       kuga_sub__MRR__c, kuga_sub__ARR__c, kuga_sub__StartDate__c, kuga_sub__EndDate__c,
       kuga_sub__Status__c, kuga_sub__Renew__c, kuga_sub__Active__c, kuga_sub__IsActive__c
FROM kuga_sub__Subscription__c
WHERE kuga_sub__ContractNumber__c = '<contract_id>'
ORDER BY kuga_sub__Service__r.Name
```

## Cancellations

```sql
-- After cancelling Subscriptions: was each Order Service Line stamped?
SELECT Id, Name, kuga_sub__Status__c, kuga_sub__Quantity__c,
       kuga_sub__OrderServiceLine__r.kugo2p__CancelDate__c,
       kuga_sub__OrderServiceLine__r.kugo2p__CancelledQuantity__c,
       kuga_sub__OrderServiceLine__r.kugo2p__CancelReason__c
FROM kuga_sub__Subscription__c
WHERE Id IN ('<subscription_ids>')

-- … and did its scheduled billing stop?
SELECT Id, Name, kugo2p__RecordStatus__c, kugo2p__SalesOrderServiceLine__c
FROM kugo2p__KugamonInvoiceLine__c
WHERE kugo2p__SalesOrderServiceLine__c = '<order_service_line_id>'
ORDER BY CreatedDate
```

## Merge Contract (read-only eligibility check)

```sql
-- the current Contract
SELECT Id, ContractNumber, AccountId, kuga_sub__Pricebook2Id__c, kuga_sub__RenewalOpportunity__c,
       kuga_sub__RenewalOpportunity__r.kuga_sub__DateRequired__c
FROM Contract WHERE Id = '<current_contract_id>'

-- blocks the merge if either returns a row
SELECT Id FROM kugo2p__SalesQuote__c WHERE kugo2p__Opportunity__c = '<current_renewal_opp_id>' LIMIT 1
SELECT Id FROM kugo2p__SalesOrder__c WHERE kugo2p__Opportunity__c = '<current_renewal_opp_id>' LIMIT 1

-- eligible primary Contracts
SELECT Id, ContractNumber, StartDate, EndDate, kuga_sub__RenewalOpportunity__r.Name,
       kuga_sub__RenewalOpportunity__r.kuga_sub__DateRequired__c,
       kuga_sub__RenewalOpportunity__r.kuga_sub__ServiceDateExpires__c,
       kuga_sub__RenewalOpportunity__r.RecordType.Name,
       kuga_sub__RenewalOpportunity__r.kuga_sub__ParentContract__c
FROM Contract
WHERE kuga_sub__Effective__c = true
  AND AccountId = '<account_id>'
  AND kuga_sub__Pricebook2Id__c = '<pricebook_id>'
  AND Id != '<current_contract_id>'
  AND kuga_sub__RenewalOpportunity__r.kuga_sub__DateRequired__c <= <current_date_required>
  AND kuga_sub__RenewalOpportunity__r.kuga_sub__ServiceDateExpires__c >= <current_date_required>
ORDER BY kuga_sub__RenewalOpportunity__r.kuga_sub__DateRequired__c, ContractNumber
```

The query reads only what the running user can see; the merge itself is blocked by Quotes and Orders the user can't see.

## Invoices

```sql
SELECT Id, Name, kugo2p__Account__r.Name, kugo2p__RecordStatus__c, kugo2p__IsPosted__c,
       kugo2p__PaymentStatus__c, kugo2p__TotalAmount__c, kugo2p__BalanceDueAmount__c,
       kugo2p__InvoiceDate__c, kugo2p__InvoiceDueDate__c
FROM kugo2p__KugamonInvoice__c WHERE kugo2p__Account__c = '<account_id>'
ORDER BY kugo2p__InvoiceDate__c DESC

-- Invoice Lines
SELECT Id, kugo2p__ProductServiceName__c, kugo2p__Quantity__c, kugo2p__SalesPrice__c,
       kugo2p__DiscountSalesPrice__c, kugo2p__TotalAmount__c, kugo2p__BalanceDueAmount__c,
       kugo2p__SalesOrderProductLine__c, kugo2p__SalesOrderServiceLine__c
FROM kugo2p__KugamonInvoiceLine__c WHERE kugo2p__Invoice__c = '<invoice_id>'

-- The Order's Invoice Schedule
SELECT Id, Name, kugo2p__BillingFrequency__c, kugo2p__InvoiceSchedule__r.Name,
       kugo2p__InvoiceSchedule__r.kugo2p__SchedulePeriod__c,
       kugo2p__InvoiceSchedule__r.kugo2p__ServicesInvoicedAtServiceStart__c
FROM kugo2p__SalesOrder__c WHERE Id = '<order_id>'

-- Service Delivery Schedule rows for an Order
SELECT Id, kugo2p__SalesOrderServiceLine__c, kugo2p__DateServiceStart__c, kugo2p__DateServiceDelivered__c,
       kugo2p__Quantity__c, kugo2p__ServiceAmount__c, kugo2p__RecordStatus__c
FROM kugo2p__ServiceDeliverySchedule__c WHERE kugo2p__SalesOrder__c = '<order_id>'
ORDER BY kugo2p__DateServiceStart__c
```

## Payments

```sql
-- Gateways
SELECT Id, Name, RecordType.Name, kugo2p__Default_Connection__c, kugo2p__Assigned_to_Terminal__c,
       kugo2p__Test_Mode__c, kugo2p__Site_Checkout__c
FROM kugo2p__Processor_Connection__c

-- Recurring Charges for an Account
SELECT Id, Name, RecordType.Name, kugo2p__SalesOrder__r.Name, kugo2p__Amount__c,
       kugo2p__Frequency__c, kugo2p__Next_Payment_Date__c, kugo2p__Subscription_Status__c
FROM kugo2p__Payment_Profile__c WHERE kugo2p__Account__c = '<account_id>'

-- Payments for an Account
SELECT Id, Name, kugo2p__Amount__c, kugo2p__Status__c, kugo2p__Transaction_Type__c, kugo2p__Date__c,
       kugo2p__SalesOrder__r.Name, kugo2p__Invoice__r.Name, kugo2p__Memo__c
FROM kugo2p__PaymentX__c WHERE kugo2p__Account__c = '<account_id>'
ORDER BY kugo2p__Date__c DESC

-- Where a Payment was applied
SELECT Id, kugo2p__AmountApplied__c, kugo2p__Invoice__r.Name, kugo2p__InvoiceLine__c,
       kugo2p__SalesOrder__r.Name, kugo2p__SalesOrderProductLine__c, kugo2p__SalesOrderServiceLine__c
FROM kugo2p__AppliedPayment__c WHERE kugo2p__Payment__c = '<payment_id>'
```

## Shipments

```sql
SELECT Id, Name, kugo2p__RecordStatus__c, kugo2p__SalesOrder__r.Name, kugo2p__ShipmentDate__c,
       kugo2p__Carrier__r.Name, kugo2p__Warehouse__r.Name, kugo2p__TrackingNumber__c
FROM kugo2p__Shipment__c WHERE kugo2p__SalesOrder__c = '<order_id>'

SELECT Id, kugo2p__Shipment__r.Name, kugo2p__Product__r.Name, kugo2p__Quantity__c,
       kugo2p__SalesOrderProductLine__c, kugo2p__RecordStatus__c
FROM kugo2p__ShipmentLine__c WHERE kugo2p__SalesOrder__c = '<order_id>'
```

## Products and pricing

```sql
-- A Product's APD
SELECT Id, Name, kugo2p__ReferenceProduct__c, kugo2p__Active__c, kugo2p__Service__c,
       kugo2p__CreateAsset__c, kugo2p__DisableShipments__c, kugo2p__DefaultServiceTerm__c,
       kugo2p__UnitofTerm__c, kugo2p__PriceScale__c, kugo2p__PricingMethod__c,
       kugo2p__KitBundle__c, kugo2p__ConfigurationMethod__c, kugo2p__ExcludefromOpportunitySync__c
FROM kugo2p__AdditionalProductDetail__c WHERE kugo2p__ReferenceProduct__c = '<product2_id>'

-- Kit/Bundle members (the kit's APD Id)
SELECT Id, kugo2p__ReferenceMemberProductService__r.Name, kugo2p__Quantity__c, kugo2p__Required__c,
       kugo2p__HideKitBundleMember__c, kugo2p__SortOrder__c
FROM kugo2p__KitBundleMember__c WHERE kugo2p__ReferenceKitBundle__c = '<kit_apd_id>'
ORDER BY kugo2p__SortOrder__c

-- Configuration groups and options (the configurable Product's APD Id)
SELECT Id, Name, kugo2p__MinimumOption__c, kugo2p__MaximumOption__c, kugo2p__SortOrder__c
FROM kugo2p__ConfigurationGroup__c WHERE kugo2p__AdditionalProductInfo__c = '<apd_id>'
ORDER BY kugo2p__SortOrder__c

SELECT Id, kugo2p__ConfigurationGroup__r.Name, kugo2p__OptionProduct__r.Name, kugo2p__Required__c,
       kugo2p__Selected__c, kugo2p__PriceBehavior__c, kugo2p__LineDiscount__c, kugo2p__PriceDiscount__c
FROM kugo2p__ConfigurationOption__c WHERE kugo2p__ConfigureProduct__c = '<apd_id>'
ORDER BY kugo2p__SortOrder__c

-- Tiered Pricing and its Tiers
SELECT Id, Name, kugo2p__Product__r.Name, kugo2p__PriceBookName__c, kugo2p__Type__c, kugo2p__IsActive__c
FROM kugo2p__TieredPricing__c WHERE kugo2p__Product__c = '<product2_id>'

SELECT Id, kugo2p__QuantityStart__c, kugo2p__TermStart__c, kugo2p__TierPrice__c
FROM kugo2p__Tier__c WHERE kugo2p__TieredPricing__c = '<tiered_pricing_id>'
ORDER BY kugo2p__QuantityStart__c

-- Account Pricing for an Account
SELECT Id, kugo2p__ProductService__r.Name, kugo2p__FilterObject__c, kugo2p__FilterField__c,
       kugo2p__Operator__c, kugo2p__FilterValue__c, kugo2p__Price__c, kugo2p__Discount__c,
       kugo2p__DateStart__c, kugo2p__DateEnd__c, kugo2p__IsActive__c, kugo2p__Hide__c
FROM kugo2p__AccountPricing__c WHERE kugo2p__Account__c = '<account_id>'
```

## Tax and VAT/GST

```sql
-- US sales tax: Tax Locations and their rates
SELECT Id, Name, kugo2p__Description__c, kugo2p__TaxableSales__c, kugo2p__TaxableUsage__c, kugo2p__TaxableExcise__c
FROM kugo2p__TaxLocation__c

SELECT Id, kugo2p__TaxLocation__r.Name, kugo2p__Type__c, kugo2p__Jurisdiction__c, kugo2p__Rate__c
FROM kugo2p__TaxRate__c WHERE kugo2p__TaxLocation__c = '<tax_location_id>'

-- VAT/GST and its rates
SELECT Id, Name, kugo2p__Description__c FROM kugo2p__VAT__c

SELECT Id, kugo2p__VAT__r.Name, kugo2p__VATName__c, kugo2p__VATRate__c, kugo2p__DateEffective__c, kugo2p__DateExpire__c
FROM kugo2p__VATRate__c WHERE kugo2p__VAT__c = '<vat_id>'

-- An Account's tax setup lives on its AAD
SELECT Id, kugo2p__TaxLocation__r.Name, kugo2p__TaxableSales__c, kugo2p__TaxableUsage__c,
       kugo2p__TaxableExcise__c, kugo2p__TaxableVAT__c, kugo2p__VATRegistration__c,
       kugo2p__VATExemptReason__c, kugo2p__ResaleCertificate__c, kugo2p__ResaleCertificateExpires__c
FROM kugo2p__AdditionalAccountDetail__c WHERE kugo2p__Account__c = '<account_id>'
```

## Quote Chat

```sql
-- A Quote's full thread
SELECT Id, Name, CreatedDate, CreatedById,
       kugo2p__ChatParticipant__r.kugo2p__ParticipantName__c,
       kugo2p__ChatParticipant__r.kugo2p__Type__c,
       kugo2p__Body__c, kugo2p__BodyPreview__c
FROM kugo2p__ChatMessage__c
WHERE kugo2p__ChatParticipant__r.kugo2p__QuoteNumber__c = '<quote_id>'
ORDER BY CreatedDate ASC

-- Participants, with read state
SELECT Id, Name, kugo2p__ParticipantName__c, kugo2p__Type__c, kugo2p__User__c, kugo2p__Contact__c,
       kugo2p__Email__c, kugo2p__ParticipantKey__c, kugo2p__LastReadMessage__c, kugo2p__DateLastRead__c
FROM kugo2p__ChatParticipant__c
WHERE kugo2p__QuoteNumber__c = '<quote_id>'
ORDER BY CreatedDate ASC

-- The acting user's own participant on a Quote
SELECT Id FROM kugo2p__ChatParticipant__c WHERE kugo2p__ParticipantKey__c = '<quote_id>-<acting_user_id>'
```
