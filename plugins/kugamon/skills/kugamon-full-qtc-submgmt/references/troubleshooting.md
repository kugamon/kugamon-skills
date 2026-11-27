# Troubleshooting: Common Issues

Reference for the `kugamon-full-qtc-submgmt` skill. Symptom first, then the usual cause and fix. Workflow numbers refer to SKILL.md; package versions are listed only there (Package Detection → Package versions).

## Quotes and Opportunities

- **The Opportunity's products disappeared or were replaced**: expected on a Primary Quote's first sync, and whenever its Price Book or currency differs from the Opportunity's. An empty Primary Quote's first sync deletes them all.
- **The Opportunity's products aren't updating**: check that the sync setting lists the Quote's current status, that the Quote is Primary with an Opportunity, then (after a short wait) the Sync Task on the Quote. "pricebook entry is inactive" in that Task means one line stopped the whole sync — reactivate the entry or remove the line.
- **Quote total vs. Amount mismatch**: `HAS_KUGA_SUB = true` → `Amount` may be MRR; compare with `kuga_sub__AnnualContractValueInitial__c`. `HAS_KUGA_SUB = false` → excluded lines, zero quantities, Quantity rounding and lines without a Price Book Entry explain differences; price decimals don't.
- **ACV double-counting**: lines with `kuga_sub__Renew__c = false` but non-zero ARR. Set Renew = true on recurring lines (`amounts-and-renew.md`).
- **Every new Quote expires today**: Kugamon Settings Quote Valid In Days is blank.
- **Wrong Contacts, terms or tax flags on a new Quote or Order**: they come from the AAD. Fix the AAD.
- **A Quote stayed Draft after it was emailed**: the subject must contain the Quote number, the email must be logged against the Quote, and the sender needs a Kugamon license.
- **Generate Order left the Quote Won with no Order**: the Quote is Restricted or its Price Book was renamed, deleted or deactivated. Fix it (Choose Price Book) and click Generate Order again.
- **The price isn't the List Price** (or should have been the customer's price): check the price lookup order in `pricing-and-decimals.md`. Lines created through the API with a blank Sales Price get only the List Price.
- **Line items not populating**: check both Product and Service line objects, and the APD's `kugo2p__Service__c`.
- **kuga_sub fields don't exist**: set `HAS_KUGA_SUB = false` and use the APD Service flag.

## Orders and Release

- **A status change or Release is refused**: read the message against the checks in Workflow 2 — most often a missing Warehouse or Contact, the AAD's payment fields, a Restricted Order, a Posted Invoice, or a Shipped Shipment.
- **Inserting several Orders fails with "The Payment Method cannot be blank."**: insert one Order per save.
- **Release took a long time and failed with a CPU limit**: the Order is too large for one Release (about 500 lines is the safe ceiling). Split it.
- **Nothing was built on Release**: the Application Settings switches Disable Invoices and Disable Shipments may be on (they can't be queried in a customer's org — ask the admin or Kugamon Support), or the Order already had an Invoice link.
- **Order Release not creating Contracts or Subscriptions**: check the Order's Generate checkboxes, the Product2 flags, and that the Order reached the status kuga_sub releases on (Released, or Approved in `Approve/Release` mode).
- **Order Release not creating Assets**: Assets come from kugo2p. Check that the Product Line's APD `kugo2p__CreateAsset__c` is `Per Line` or `Per Quantity` and that the Order entered Approved or Released. Assets are created only when the Order has none; after changing product setup, use Refresh Assets.
- **Release error about "Per Quantity" parent products** (kuga_sub): a parent product (kit/bundle or configured product) with `CreateAsset__c = Per Quantity` can't have a child product set to `Per Line` or a child service that creates a Subscription. Change the parent to `Per Line` or blank; if the Order was Approved, also run Refresh Assets.

## Subscriptions and Contracts

- **A cancelled Subscription is still billing**: check the Order Service Line's `kugo2p__CancelReason__c` and its invoice lines' `kugo2p__RecordStatus__c`. Usual causes: the kuga_sub version (Package versions); the line isn't renewable; or the Subscription was one of more than 200 cancelled in one transaction (Workflow 4a). Invoices generated before the cancellation never change.
- **A cancelled Subscription came back after Refresh Products**: it was cancelled before the org reached the kuga_sub version in Package versions, so its Order Service Line has no Cancel Date. Stamping the line (`kugo2p__CancelDate__c`, `kugo2p__CancelledQuantity__c`, `kugo2p__CancelReason__c = 'Subscription Cancelled'`) keeps it out — but also cancels the line's `Scheduled` invoice lines, so confirm with the user first.
- **Merge Contract shows no eligible Contracts**: compare the candidates with the four eligibility rules (and the merge-time renewal check) in Workflow 4c. The usual misses are a different Price Book and a renewal Date Required outside the other Contract's renewal service window.
- **Merge Contract is blocked**: the current Contract's Current Renewal Opportunity has a Quote or an Order (including ones the user can't see). Delete or re-home them first; the merge deletes that opportunity.

## Invoices and Payments

- **The payment shows but the Invoice is still unpaid**: the Payment isn't `Completed` (often Authorized, Pending, or a gateway call that didn't finish).
- **Paid on the Order, but the Invoices show unpaid**: check the Payment is Completed; Release re-applies Order Payments to the Invoices.
- **Balance Due is negative**: the customer overpaid; the extra sits on the first line.
- **Monthly invoicing produced one invoice**: the service's Unit of Term is Year, or the whole term falls inside one month.
- **Can't unrelease or cancel the Order**: a Shipment is Shipped or Delivered, an Invoice is Posted, or a Payment is applied to one of its Invoices.

## Quote Chat

- **Quote Chat panel says "Please contact your Administrator to enable Quote Chat."**: Enable Online Chat is off or the Sites URL is blank.
- **"Page Link not Configured"**: the Quote has no Online Approval Key — edit and save the Quote.
- **Messages from an integration show a blank preview**: it didn't set `kugo2p__BodyPreview__c`.
