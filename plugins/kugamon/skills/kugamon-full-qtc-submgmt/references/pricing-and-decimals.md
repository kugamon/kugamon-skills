# Pricing and Decimal Prices

Reference for the `kugamon-full-qtc-submgmt` skill. Package versions are listed only in SKILL.md (Package Detection → Package versions); Price Scale and 6-decimal prices need the kugo2p version listed there.

## How a line's starting price is found

When a Product is added in the Add Lines or Edit Lines screens, the most specific source wins:

| # | Price | Applies when |
|---|---|---|
| 1 | Included ($0) | A Configuration Option's Price Behavior is Included, or the kit pricing makes the line $0 |
| 2 | Cost price | The APD's `kugo2p__PricingMethod__c` is Cost. Account Pricing doesn't apply |
| 3 | Purchase price | An Asset or Subscription exists for the Product (with the same Contract, when kuga_sub is installed) |
| 4 | Account price | Active Account Pricing for the Account or its parent, in the Quote's currency |
| 5 | Option price | A Configuration Option's Price Behavior is Discounted, and no Account Pricing applies |
| 6 | Tier Price | Active Tiered Pricing for the Product and the Quote's Price Book |
| 7 | List Price | The Price Book Entry's Unit Price |

- **Only the line editor applies this order.** A line saved with a blank Sales Price through Import Lines, Apex, Flow or the API takes the List Price — no Account Pricing, no Tier Price. Set `kugo2p__SalesPrice__c` yourself when it should be something else.
- **Price Book Entries are required.** A Product with no entry in the Quote's Price Book and currency can't be added ("One or more of the products added to the quote do not belong to this Pricebook…").
- The line's **Pricing Source** shows where the price came from.

## Account Pricing (`kugo2p__AccountPricing__c`)

Customer-specific prices or discounts, so a negotiated rate applies without a separate Price Book.

- **One rule per row.** A row gives a **Price** (`kugo2p__Price__c`) or a **Discount** % (`kugo2p__Discount__c`), not both, and targets either one Product/Service (`kugo2p__ProductService__c`) or a **filter** (`kugo2p__FilterObject__c` Product or APD, `kugo2p__FilterField__c`, `kugo2p__Operator__c`, `kugo2p__FilterValue__c` — e.g. Family equals Hardware), not both. A row can instead only **Hide** the Product for that Account (`kugo2p__Hide__c`).
- **Dates and Contract.** It applies between `kugo2p__DateStart__c` and `kugo2p__DateEnd__c`, and can be tied to `kugo2p__Contract__c`.
- **Parent Accounts.** Rows on a parent Account apply to its children; the closest Account wins. The AAD's `kugo2p__IgnoreParentAccountPricing__c` turns this off for an Account.
- **Ties.** The Account's own row beats the parent's; a Product/Service row beats a filter row.
- **Currency.** In multi-currency orgs, create one row per currency.
- **Overrides.** `kugo2p__LockSalesPrice__c`, `kugo2p__LockLineDiscount__c` and `kugo2p__Hide__c` on the row override the Product's own settings for that Account.
- Validation: "You must enter either Price or Discount", "You must enter either Product/Service or Filter Information", "You must enter Filter Object, Filter Field and Operator fields", "Date End must be greater than or equal to Date Start."

When the agreed price didn't apply, check in order: the dates cover today, the currency matches, the Product or filter matches, the Product isn't cost-priced or an included option, and no more specific row wins.

## Tiered Pricing (`kugo2p__TieredPricing__c`, `kugo2p__Tier__c`)

- A Tiered Pricing ties one Product2 (`kugo2p__Product__c`) to one Price Book and has a **Type**: Volume or Slab (blank behaves as Volume). Only one can be active per Product per Price Book, and it applies only to List-priced Products.
- **Tiers** set a starting quantity (`kugo2p__QuantityStart__c`) or starting term (`kugo2p__TermStart__c`) and a `kugo2p__TierPrice__c`.
- **Volume:** the whole quantity is priced at the tier it reaches. **Slab:** each band is priced at its own tier and the line gets the average unit price, carried at the Product's Price Scale. Example with tiers from 1 (100), 10 (80) and 50 (60): Volume prices 10 units at 80; Slab prices 10 units at 98 (9 × 100 + 1 × 80, divided by 10).
- Tier Prices hold up to 6 decimals and are rounded to the Product's Price Scale.

## Kits and Bundles (`kugo2p__KitBundleMember__c`)

- **Setup.** On the header's APD: Configuration Method Kit/Bundle (`kugo2p__KitBundle__c`) and a `kugo2p__KitBundlePricing__c`. Members are children of the kit's APD (`kugo2p__ReferenceKitBundle__c`), each with a member (`kugo2p__ReferenceMemberProductService__c`), `kugo2p__Quantity__c`, `kugo2p__Required__c`, `kugo2p__AllowQuantityChange__c` and `kugo2p__HideKitBundleMember__c`.
- **On a Quote** (by any route, the API included): adding the header adds its members. Member quantity = header quantity × member quantity, and changing the header quantity updates the members. Required members can't be deleted; deleting the header deletes its members. A service kit can contain only services.

| Kit/Bundle Pricing | Header line | Member lines |
|---|---|---|
| Kit/Bundle Price Only | Its Price Book price | $0 (Charge Reason "Kit/Bundle") |
| Member Prices Only | $0 (Charge Reason "Kit/Bundle") | Their own price (or Tier Price) |
| Kit/Bundle Price + Member Prices | Its price | Their own price |

- **Hidden members** don't show in Edit Lines but are saved, priced and **counted in totals** — hide only $0 members. They print on PDFs only when the header's APD has `kugo2p__PrintKitBundleMembersonPDFs__c`.
- No nested kits; a member must be active, have a quantity above 0, and not share the kit's name.

## Configurable Products

- **Setup.** On the APD: `kugo2p__ConfigurationMethod__c` Configure, `kugo2p__ConfigurationType__c` (Required, Allowed, Disabled) and `kugo2p__ConfigurationAction__c` (Add, Edit, Always). **Configuration Groups** (`kugo2p__ConfigurationGroup__c`, children of the APD) set `kugo2p__MinimumOption__c` and `kugo2p__MaximumOption__c`. **Configuration Options** (`kugo2p__ConfigurationOption__c`) each have an `kugo2p__OptionProduct__c`, `kugo2p__Required__c`, `kugo2p__Selected__c` (selected by default), quantity rules and a `kugo2p__PriceBehavior__c`:
  - **Included** — the line is $0;
  - **Discounted** — reduced by `kugo2p__LineDiscount__c` % **or** `kugo2p__PriceDiscount__c`, not both, and only when no Account Pricing applies;
  - **Related** — the line takes its normal price.
- `kugo2p__PrintConfigurationOption__c` decides whether option lines print; `kugo2p__HideConfigurationOption__c` hides the line in Edit Lines only — it still counts in totals.
- Groups, options and kit members are maintained in the **Product Setup Configurator**, opened from the APD.

## Header discount

The Quote's (or Order's) `kugo2p__DiscountPercent__c` applies to every Product and Service Line on top of each line's `kugo2p__LineDiscountPercent__c`. **The two add, they don't compound**: 10% + 5% takes 15% off. It defaults from the AAD. Additional Charges/Credits aren't discounted by it.

## Price Scale: up to 6 decimals

The APD's `kugo2p__PriceScale__c` (2 to 6; blank = 2) sets how many decimals a Product's prices round to and display at. Different Products on one Quote can use different scales. `kugo2p__QuantityScale__c` and `kugo2p__ServiceTermScale__c` do the same for quantity and service term.

### Stored fields that hold up to 6 decimals (`Currency(12, 6)`)

| Object | Fields |
|---|---|
| `kugo2p__SalesQuoteProductLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__SalesQuoteServiceLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__NonUpliftSalesPrice__c`, `kugo2p__UpliftPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__SalesQuoteOptionalLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c` |
| `kugo2p__SalesOrderProductLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__SalesOrderServiceLine__c` | `kugo2p__ListPrice__c`, `kugo2p__SalesPrice__c`, `kugo2p__NonUpliftSalesPrice__c`, `kugo2p__UpliftPrice__c`, `kugo2p__TierPrice__c` |
| `kugo2p__AccountPricing__c` | `kugo2p__Price__c` |
| `kugo2p__Tier__c` | `kugo2p__TierPrice__c` |
| `Asset` | `kugo2p__PurchasePrice__c` |
| `kuga_sub__Subscription__c` | `kuga_sub__PurchasePrice__c` (see Package versions) |
| `OpportunityLineItem` | `kuga_sub__NonUpliftSalesPrice__c` (kuga_sub) |

### Formula fields that carry line prices

| Object | Field | Precision |
|---|---|---|
| Quote and Order lines | `kugo2p__DiscountSalesPrice__c` ("Effective Price") | Sales Price less the header and line discounts, rounded to the Product's Price Scale |
| `kugo2p__KugamonInvoiceLine__c` | `kugo2p__DiscountSalesPrice__c` ("Effective Price") | Follows the billed Order line's price and discounts |
| `kugo2p__KugamonInvoiceLine__c` | `kugo2p__SalesPrice__c` | Reads the Order line's Sales Price |

### What stays at 2 decimals

- **Amounts and totals** — line amounts (Line, Net, Total, Tax, VAT/GST, Discount, Balance Due), Quote/Order/Invoice header totals, and every Invoice Line amount.
- Payments and Applied Payments, Charges/Credits, the APD's `kugo2p__StandardPrice__c`.
- `PricebookEntry.UnitPrice` (standard).
- Subscription `kuga_sub__MRR__c`, `kuga_sub__ARR__c`, `kuga_sub__NetAmount__c`, `kuga_sub__TotalAmount__c`, and Opportunity roll-ups such as `kuga_sub__Amount__c`.
- `OpportunityLineItem.UnitPrice` **displays** 2 decimals but stores the Quote line's price unrounded, as written by the sync.

### Display

- Kugamon's own screens — Add Lines, Edit Lines, the configurators — and its PDFs and online pages show each line at its Product's Price Scale.
- Standard record pages, list views and reports show the 6-decimal fields with 6 decimals for every Product, because a field has one fixed scale. That's expected.

### Reading and writing

1. Read 6-decimal fields without rounding them unless the user asks — the extra decimals carry pricing meaning.
2. A price written through the API with more decimals than the Product's Price Scale is **stored as written**; Kugamon's screens and documents round it. Round to the Price Scale yourself if the stored value must match what the customer sees.
3. Don't truncate prices in custom display logic; use the Product's Price Scale.
4. Opportunity amount roll-ups stay at their own 2-decimal precision — line-level precision doesn't roll up.
