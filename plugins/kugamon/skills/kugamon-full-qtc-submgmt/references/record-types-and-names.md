# Record Types and Name Fields

Reference for the `kugamon-full-qtc-submgmt` skill. Package versions are listed only in SKILL.md (Package Detection → Package versions).

## Record types

**Never hardcode Record Type Ids** — they differ between orgs. Query them (`queries.md`, Setup and settings), cache the result for the session, and match on `DeveloperName` (labels can be renamed).

| Object | Record types | Comes from |
|---|---|---|
| Quote (`kugo2p__SalesQuote__c`), Order (`kugo2p__SalesOrder__c`), Opportunity | `New`, `Expansion`, `Renewal` | kuga_sub. Quote to Cash alone adds no Quote or Order record types |
| Payment Method (`kugo2p__Payment_Method__c`) | Developer names `Stripe`, `AuthNet_CIM_Payment_Profile`, `eWay_Customer_Profile`, `PayPal_Transaction_Reference`, `Credit_Card` | kugo2p |
| Recurring Charge (`kugo2p__Payment_Profile__c`) | Developer names `AuthNet_Subscription`, `Credit_Card`, `Generic_Profile`, `PayPal_Recurring_Payment`, `PayPal_Subscription`, `Scheduled_Payment` | kugo2p |
| Processor Connection (`kugo2p__Processor_Connection__c`) | `Stripe`, `Authorize_Net`, `PayPal`, `eWay` (plus older, deprecated types) | kugo2p |

### Choosing a Quote or Order record type

- **The org has none** (Quote to Cash without kuga_sub, or no custom record types): create Quotes and Orders without `RecordTypeId`.
- **With kuga_sub**, match the Opportunity's record type: Opportunity `New` → Quote/Order `New`; `Expansion` → `Expansion`; `Renewal` → `Renewal`. If nothing matches, ask the user or use `New`.
- Some orgs rename or add record types, or leave some inactive — use only the Ids the query returned.
- Record type matters with kuga_sub: Refresh Products and Merge Contract need a `Renewal` renewal opportunity, and the Per Line Group order method is valid only on New and Expansion Quotes.

## Name fields and sample data

When creating sample or test records, **`Name` is rarely yours to set.** Most transactional objects use **Auto Number** — Salesforce assigns the value on insert, the field is neither createable nor updateable (a value you pass is ignored), and the format is fixed by the package. A hand-typed Name like `Q-001`, `Test Quote` or `SQ-0001` is a sign the record was hand-crafted rather than created through the normal flow.

Before authoring a sample record, run `get_object_fields` on the object and read the `Name` field's type.

### Group A: Auto Number with a date stamp — don't set Name

| Object | Format | Example |
|---|---|---|
| `kugo2p__SalesQuote__c` | `SQ-{YYMMDD}-{0000000}` | `SQ-260519-0020461` |
| `kugo2p__SalesOrder__c` | `SO-{YYMMDD}-{0000000}` | `SO-260521-0113570` |
| `kugo2p__KugamonInvoice__c` | `INV-{YYMMDD}-{0000000}` | `INV-260505-0091615` |

`YYMMDD` is the org-local creation date — it can't be back-dated. Kugamon also recognises these numbers in email subjects (logging a completed Task whose subject contains the number moves a Draft Quote or Order, or an Unsent Invoice, to Sent).

### Group B: Auto Number with a prefix or sequence — don't set Name

| Object | Format | Example |
|---|---|---|
| `kugo2p__Shipment__c` | `SHP-{0000000}` | |
| `kuga_sub__Subscription__c` | `SUB-{0000000}` | `SUB-0003218` |
| `kugo2p__ChatParticipant__c` | `CP-{0000000}` | `CP-0000070` |
| `kugo2p__ConfigurationOption__c` | `CO-{0000000}` | `CO-0000003` |
| `kugo2p__ChatMessage__c` | 9-digit sequence | `000000165` |
| `kugo2p__SalesQuoteProductLine__c` | sequence | `0098308` |
| `kugo2p__SalesQuoteServiceLine__c` | sequence | `0043468` |
| `kugo2p__SalesQuoteAdditionalChargeCredit__c` | sequence | `0009725` |
| `kugo2p__SalesQuoteOptionalLine__c` | sequence | `0003452` |
| `kugo2p__SalesOrderProductLine__c` | sequence | `0167453` |
| `kugo2p__SalesOrderServiceLine__c` | sequence | `0098030` |
| `kugo2p__SalesOrderAdditionalChargeCredit__c` | sequence | `0050596` |
| `kugo2p__KugamonInvoiceLine__c` | sequence | `0186531` |
| `kugo2p__KugamonInvoiceAdditionalChargeCredit__c` | sequence | `0008382` |
| `kugo2p__OrderInvoiceRelationship__c` | sequence | `0077818` |
| `kugo2p__ShipmentLine__c` | sequence | |
| `kugo2p__AppliedPayment__c` | sequence | `0031357` |
| `kugo2p__AdditionalProductDetail__c` | sequence | `0024014` |
| `kugo2p__ProductCost__c` | sequence | `0000000` |

### Group C: Text set by Kugamon — don't set Name

The field is technically writable, but Kugamon overwrites it on insert and update:

| Object | What Kugamon writes | Example |
|---|---|---|
| `kugo2p__PaymentX__c` | `Payment for Order <SO#>` or `Payment for Invoice <INV#>`; a refund adds ` (Refund)` | `Payment for Order SO-260325-0113546` |
| `kugo2p__Payment_Method__c` | `<Card Brand> (<last 4>)` from the tokenized card | `Visa (4242)` |
| `kugo2p__AdditionalAccountDetail__c` | The Account's name (up to 80 characters); follows renames | `Starbucks Corporation` |

### Group D: Text you supply — set a meaningful Name

Line groups, invoice schedules, recurring charges, reusable additional charges/credits, product catalogs and categories, tiered pricing, carriers, warehouses, tax locations, VAT/GST, processor connections. Match the conventions already in the org.

### Mistakes to avoid

- Inventing a prefix (`Q-…`, `QT-…`, `ORD-…`, `IN-…`, `INV2-…`, or `SQ-0001` without a date). The Quote, Order and Invoice prefixes are fixed — `SQ`, `SO`, `INV` — the date is the real `YYMMDD` of creation, and the sequence is 7 digits.
- Supplying `Name` on Auto Number objects — it's dropped.
- Supplying `Name` on Group C objects and expecting it to stick — it's overwritten.
- Not checking the Name type with `get_object_fields` first.
