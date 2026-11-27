# Subscriptions and Contracts: Cancel, Merge, Refresh

Reference for the `kugamon-full-qtc-submgmt` skill (Workflows 4a–4d), for orgs with `HAS_KUGA_SUB = true`. Read the whole section for the action before you cancel, merge or refresh. Package versions are listed only in SKILL.md (Package Detection → Package versions). Queries: `queries.md` (Cancellations, Merge Contract).

## Workflow 4a: Cancel a Subscription

**How.** The Cancel Subscription button sets `kuga_sub__Subscription__c.kuga_sub__Status__c = 'Cancelled'` and nothing else; everything that follows runs in the Subscription trigger, so updating the field through the API has the same effect. `kuga_sub__Status__c` is a restricted picklist: `Purchased`, `Provisioned`, `Cancelled`.

**Refused when** the Subscription is already `Cancelled`, or `kuga_sub__EndDate__c` is before today — "Expired Subscriptions cannot be Cancelled." (enforced in the trigger, so an API update fails too).

**What happens** (the Order Service Line steps need the kuga_sub version in Package versions), for a Subscription whose Order Service Line is renewable (`kuga_sub__Renew__c = true`):

1. **The Order Service Line is cancelled.** `kugo2p__CancelDate__c` = today, `kugo2p__CancelledQuantity__c` = the Subscription's `kuga_sub__Quantity__c`, `kugo2p__CancelReason__c` = `Subscription Cancelled`.
2. **Its scheduled billing stops.** When an Order Service Line's Cancel Reason goes from blank to a value, its invoice lines and service delivery schedules with `kugo2p__RecordStatus__c = 'Scheduled'` become `Cancelled`, and the Order's service tax, discount and VAT totals are recalculated. Invoices already generated are not touched — credit or refund those separately if the user wants money back.
3. **The renewal opportunity shrinks.** On the Contract's Current Renewal Opportunity (`Contract.kuga_sub__RenewalOpportunity__c`), the matching product is reduced by the cancelled quantity, and deleted at zero or below. A product matches only if all of these agree: the renewal product (the product's `kuga_sub__RenewalProduct__c`, or the product itself); the line's sales price against the renewal product's `kuga_sub__NonUpliftSalesPrice__c` (to 6 decimals); line plus order discount; line description; configuration option. If anything differs, **nothing is reduced and no error is raised** — check the renewal opportunity and correct it by hand, or run Refresh Products (Workflow 4d).
4. **Contract totals catch up.** The subscription count drops at once. ARR and MRR roll up only subscriptions flagged `kuga_sub__IsActive__c`, which is resynced from the `kuga_sub__Active__c` formula by the scheduled Subscription job (or the next save) — so ARR and MRR drop then.

If the Order Service Line is **not** renewable, only the Subscription's Status changes: the line is not stamped, its scheduled invoice lines keep billing, and there is no renewal product to reduce. Tell the user, and check the line's scheduled invoice lines. Below the kuga_sub version in Package versions, steps 1 and 2 never happen.

Also watch for:

- **A Subscription with no Order Service Line** (`kuga_sub__OrderServiceLine__c` blank — usually created by hand or migrated): cancelling it fails with an Apex error. Report it; do not retry.
- **An Order Service Line that already has a Cancel Reason**: it is overwritten, but scheduled billing is cancelled only when Cancel Reason goes from blank to a value — so those invoice lines keep billing.
- **Child Subscriptions** (`kuga_sub__ParentSubscription__c` pointing at this one, e.g. kit or bundle members) are not cancelled with their parent. Cancel each one.

**Alternative the package itself suggests:** end a subscription with an **Expansion Order** carrying a negative quantity of the subscription's service. Releasing it reduces or removes the service on the renewal opportunity.

**Bulk cancellations.** The cancellation logic runs once per transaction. Cancel at most 200 Subscriptions per update call, and don't cancel in two separate updates inside one transaction. Records past the first 200, or in the second update, end up `Cancelled` without the line being stamped or the renewal reduced. The allowance is shared with cancelling Contracts or an Expansion Order (Workflow 4b). After any bulk cancellation, run the check query in `queries.md` (Cancellations).

## Workflow 4b: Cancel a Contract

**How.** The Cancel Contract button asks for confirmation, then sets `Contract.Status = 'Cancelled'` and moves `EndDate` back to today if it is in the future. Through the API, set the same two fields.

**Refused when** the Contract is already Cancelled, or its Status is anything other than blank, `Draft` or `Activated` — "Contract Status must be Draft or Activated to Cancel."

**What happens**

1. The Current Renewal Opportunity is set to `Closed Lost` with today's Close Date.
2. Every Subscription on the Contract with an **End Date after today** that is not already Cancelled is set to `Cancelled`, so each one runs Workflow 4a. Subscriptions **ending today** or with **blank dates** are left as they are (the selection also needs `kuga_sub__Active__c`, which needs an Effective Contract); cancel those individually if wanted.
3. The Contract can no longer be used for New Expansion Opportunity, New Expansion Order, or Refresh Products.

The 200-per-transaction limit in Workflow 4a applies here too: a Contract with more than 200 live Subscriptions needs the check afterwards.

**Cancelling an Order also cancels Subscriptions.** When an Order goes from `Approved` or `Released` to `Cancelled`, kuga_sub sets every Subscription on that Order to `Cancelled`.

- **New or Renewal Order:** the Order's renewal opportunity goes to Closed Lost, and each Subscription runs Workflow 4a.
- **Expansion Order:** the renewal opportunity is reduced by the expansion-order logic instead, and Workflow 4a is switched off for the rest of that transaction — don't cancel other Subscriptions in the same transaction.
- **Either kind:** if any Subscription on the Order has already expired, the Order can't be cancelled ("Expired Subscriptions cannot be Cancelled.").

## Workflow 4c: Merge Contracts

Needs the kuga_sub version in Package versions. Folds one Contract (the **current** Contract, the one the button is pressed on) into another Contract for the same Account (the **primary** Contract), so the subscriptions renew together under one Contract and one renewal opportunity. The current Contract is **deleted**. Get the user's explicit confirmation before they run it, and say which Contract will be deleted.

**Where.** Merge Contract button on the current Contract: `/apex/kuga_sub__MergeContract?id={Contract Id}`. If it isn't on the page layout, an admin has to add it. The page lists the eligible primary Contracts; the user picks one and confirms. **UI only** — there is no API route, and the merge must not be imitated with DML: it moves a dozen record types and deletes in a specific order inside one transaction.

**The current Contract can be merged only if**

- it has an Account, a Price Book (`kuga_sub__Pricebook2Id__c`), and a Current Renewal Opportunity with a Date Required (`kuga_sub__DateRequired__c`, the earliest product start date on the renewal);
- its Current Renewal Opportunity has **no Quote and no Order at all** — not even ones the user can't see. Otherwise: "This Contract cannot be merged. The Current Renewal Opportunity has at least one Quote and/or Order present."

**A Contract is an eligible primary only if**

1. it is Effective (`kuga_sub__Effective__c = true`);
2. it has the same Price Book and the same Account;
3. the current Contract's renewal Date Required falls inside the primary's renewal service window: primary Date Required ≤ current Date Required ≤ primary `kuga_sub__ServiceDateExpires__c`;
4. it is not the current Contract.

One more check runs only when the user confirms: the primary's Current Renewal Opportunity must have record type `Renewal` and point back to the primary as its Parent Contract (`kuga_sub__ParentContract__c`). If not, the merge stops before changing anything — so check it too. Eligibility queries (read-only): `queries.md`, Merge Contract.

**What the merge does** — one transaction; any failure rolls everything back ("Merge Contract failed. No changes were saved."):

1. **Re-points to the primary Contract:** Opportunities (`ContractId`, `kuga_sub__ParentContract__c`) other than the current Contract's renewal opportunity; Quotes (`kuga_sub__ContractNumber__c`); Orders (`kuga_sub__ContractNumber__c`, `kuga_sub__ParentContract__c`, and `kuga_sub__RenewalOpportunity__c` from the current renewal opportunity to the primary's); Order and Quote Service Lines and Account Pricing (`kugo2p__Contract__c`); Assets (`kuga_sub__ContractNumber__c`); Subscriptions (`kuga_sub__ContractNumber__c`). Records the running user can't see are moved too.
2. **Brings over the current Contract's own records:** Contract Contact Roles are re-created (a Contact + Role the primary already has is skipped; the primary keeps its own primary role); Notes and Attachments are re-created (new Ids and created dates); Files are linked to the primary; Tasks and Events are moved, archived ones included. Attachments over 4 MB in total block the merge — convert them to Files first. Any other lookup to the current Contract, such as an org's own custom fields, is cleared when it is deleted.
3. **Deletes** the current Contract, then its Current Renewal Opportunity.
4. **Runs Refresh Products** on the primary's Current Renewal Opportunity (Workflow 4d). This step respects the running user's sharing: Orders they can't see are left out of the rebuilt renewal products. Have someone who can see all of the Account's Orders run the merge.

No Quote or Order is ever deleted. The counts of moved records appear briefly in a toast, so ask the user to note them, then check the primary's renewal opportunity products yourself.

Also: the page lists at most 200 eligible Contracts; the current Contract's own Status is not checked, so a Cancelled or expired Contract can be merged in. Its Order Service Lines cancelled before the kuga_sub version in Package versions have no Cancel Date, so those services would come back onto the primary's renewal. Flag this before the user merges a Cancelled Contract.

## Workflow 4d: Refresh Products on a Renewal Opportunity

Deletes every product on the renewal opportunity and rebuilds them from the Parent Contract's Orders that point to this opportunity (`kuga_sub__RenewalOpportunity__c`), are Released (or Approved, when Kugamon Settings start subscription management on Approve/Release), and have `kuga_sub__GenerateRenewalOpportunity__c = true`. Only renewable lines (`kuga_sub__Renew__c = true`) are used. Products added by hand are lost — say so before the user runs it.

- An Order's product lines are used only if the Order also has at least one renewable, uncancelled service line. **If every service line has been cancelled, the renewal opportunity ends up with no products at all** — renewable product lines included.
- A renewal product with no price book entry in the Order's price book is skipped without an error.
- Order Service Lines with a `kugo2p__CancelDate__c` are left out, line by line, so cancelled subscriptions don't come back (see Package versions). Order Release applies the same rule when it builds renewal products. Order Product Lines have no such filter.

**Refused when** the opportunity's record type isn't `Renewal`, it has no Parent Contract, the Parent Contract is Cancelled, or it isn't the Parent Contract's Current Renewal Opportunity.

UI only — Refresh Products button on the Opportunity (or the Opportunity Products related list).
