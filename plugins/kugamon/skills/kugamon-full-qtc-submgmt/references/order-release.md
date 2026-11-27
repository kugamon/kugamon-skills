# Order Release

Reference for the `kugamon-full-qtc-submgmt` skill (Workflow 3). Package versions are listed only in SKILL.md (Package Detection → Package versions).

## Which package and which trigger creates each record

The line-object split holds: Subscriptions come only from Order Service Lines, Assets only from Order Product Lines, and Renewal Opportunities from any renewable line. The records are not created by the line triggers alone:

- **Assets** are created by **kugo2p**, from the **Order** trigger (after update), by `SalesOrderHelper.CreateAssets`. This happens in every deployment mode, with or without kuga_sub.
- **Contracts, Subscriptions and Renewal Opportunities** are created by **kuga_sub** (`KugamonHelper.createContract` → `createSubscription` → `createRenewalOpportunity`). The chain can run from kuga_sub's Order, Order Product Line or Order Service Line after-update trigger — whichever completes it in that transaction.

```
RELEASE CHAIN  (kuga_sub — HAS_KUGA_SUB = true)
─────────────
Order status → Released
  (or → Approved, when the kuga_sub setting InitiateOrderSubscriptionManagement__c = 'Approve/Release')
   └─ kuga_sub Order trigger (after update) queues the Order and notes whether it has
      renewable products / renewable services
   └─ kugo2p copies the Order's Record Status to every Order Product Line and Order Service Line
      on each status change, which fires kuga_sub's line triggers (after update)
   └─ the chain runs once per transaction, from whichever kuga_sub trigger completes it:
        • the Order trigger, if the line triggers already fired, or
        • the Order Product Line / Order Service Line trigger
      createContract → createSubscription → createRenewalOpportunity
   └─ the order in which kugo2p's and kuga_sub's triggers run on the same object is not
      guaranteed; kuga_sub handles both orders

CONTRACT  (kuga_sub createContract)
────────
   └─ sets kuga_sub__ContractNumber__c on the Order and kugo2p__Contract__c on its
      Order Service Lines; links any existing Assets to the Contract

SUBSCRIPTION  (kuga_sub createSubscription — Order Service Lines only)
────────────
Order Service Line .kuga_sub__Track__c (label: "Create Subscription";
usually follows Product2.kuga_sub__Track__c)
   └─ on Order Release, for lines on kugo2p__SalesOrderServiceLine__c (APD.Service__c = true):
          └─→ one Subscription per Order Service Line with Track__c = true
   └─ Order Product Lines have no Track__c: no Subscription, ever

ASSET  (kugo2p SalesOrderHelper.CreateAssets — Order Product Lines only)
─────
kugo2p__AdditionalProductDetail__c.kugo2p__CreateAsset__c  (picklist)
   └─ kugo2p Order trigger (after update), when the Order enters Approved or Released
      and has no Assets yet, for lines on kugo2p__SalesOrderProductLine__c (APD.Service__c = false):
          • 'Per Line'     → one Asset per Order Product Line
          • 'Per Quantity' → one Asset per unit of Quantity
          • blank          → no Asset
   └─ each Asset points to its line (Asset.kugo2p__OrderProductLine__c) and, after Release,
      its Shipment Line (Asset.kugo2p__ShipmentLine__c)
   └─ on Order Service Lines: no Asset, ever
   └─ with kuga_sub installed, kuga_sub links them in the same transaction (where applicable):
      Asset.kuga_sub__ContractNumber__c, Asset.kuga_sub__ParentSubscription__c,
      Subscription.kuga_sub__ParentAsset__c
   └─ kugo2p deletes the Assets when the Order goes back to Draft/Sent, when lines or the
      Order are deleted, and on Refresh Assets

RENEWAL OPPORTUNITY  (kuga_sub createRenewalOpportunity — any line)
───────────────────
Product2.kuga_sub__Renewable__c
   └─ on Order Release, if true on any line's product (service or product):
       └─→ Renewal Opportunity created
   └─ also drives the "Renewable" prefix on the Product Snapshot LWC

UN-RELEASE / DELETE  (kuga_sub)
───────────────────
Order un-released or deleted
   └─ kuga_sub deletes the Order's Subscriptions and the Contract created for it, and deletes
      (or closes as Lost) the Renewal Opportunity; Expansion orders reduce the renewal's lines
```

## What kugo2p's Release builds

On an Order's **first** move into Released, in this order, in one synchronous transaction:

1. The status checks (SKILL.md, Workflow 2), then the date stamps; the Record Status is copied to every Product Line, Service Line and Charge/Credit.
2. The AAD Contacts are updated if the Order's `kugo2p__UpdateAdditionalAccountInfoContacts__c` says so.
3. The build, chosen by Application Settings:

| Disable Invoices (`kugo2p__DisableInvoices__c`) | Disable Shipments (`kugo2p__DisableShipments__c`) | Release builds |
|---|---|---|
| off | off | Shipments, Service Delivery Schedules and Invoices |
| off | on | Service Delivery Schedules and Invoices (Shipment Lines are created for invoicing, then removed) |
| on | off | Shipments only |
| on | on | Nothing |

   With invoicing on, the build is skipped when the Order already has an Order/Invoice link, so releasing again doesn't duplicate Invoices. Unrelease removes that link, so a later Release builds afresh.
4. **Assets** — created for the Order if it has none yet, and linked to the new Shipment Lines.
5. **The Account Balance** — recalculated in the background after the transaction. It is the only part that isn't immediate.

An error anywhere undoes the whole Release, so an Order is never half released. Unrelease is the exact reverse.

**Service Delivery Schedules.** One row per Service Line for a one-time billing frequency, $0 lines and Excluded lines; otherwise one row per Unit of Term from Start Date to End Date (Day, Week, Month and Quarter step by that length; any other unit, including Year, steps by a year), each with an equal share of the amount and the last row taking the remainder. Rows become `Scheduled` on Release and `Cancelled` when a Released Order is cancelled; Unrelease deletes them.

## Size limits

A 588-line Order (420 product, 168 service and 8 charge/credit lines) took 46 seconds to release and used more CPU than Salesforce's 10-second limit. Orders of roughly 700–900 lines are expected to fail with "Apex CPU time limit exceeded" and roll back fully. Treat about **500 lines** as the safe ceiling for one Release, and split larger Orders.

## kuga_sub settings that change the chain

- `InitiateOrderSubscriptionManagement__c = 'Approve/Release'` starts subscription management when the Order reaches Approved as well as Released; Refresh Products then also uses Approved Orders.
- The Order's Generate checkboxes (`kuga_sub__GenerateContract__c`, `kuga_sub__GenerateAsset__c`, `kuga_sub__GenerateSubscription__c`, `kuga_sub__GenerateRenewalOpportunity__c`) decide which records the chain creates.
