# Amount Fields and the Renew Field

Reference for the `kugamon-full-qtc-submgmt` skill. Package versions are listed only in SKILL.md (Package Detection → Package versions).

## 🔑 Pipeline forecasting rule (read first)

**When `HAS_KUGA_SUB = true`, use `kuga_sub__Amount__c` for all Opportunity pipeline forecasting. DO NOT use the standard `Amount` field.** It is the Roll-Up SUM maintained by Kugamon Subscription Management for forecasting and pipeline reporting.

## The Amount field problem

In orgs with subscription management, the standard `Amount` field on Opportunity can hold Monthly Recurring Revenue (MRR), Annual Recurring Revenue (ARR), Total Contract Value (TCV) or Annual Contract Value (ACV), depending on configuration — and the Primary Quote or Order can also write it (SKILL.md, Quote and Opportunity Sync). That makes it unsuitable for pipeline forecasting and confusing to compare with quote totals.

## Amount fields

### On Opportunity

| Field | Meaning | Example |
|---|---|---|
| `Amount` (standard) | **Don't use for forecasting when `HAS_KUGA_SUB = true`.** May be MRR, ACV or TCV — varies by org | $10,000 (monthly or annual?) |
| `kuga_sub__Amount__c` | **✅ Use for pipeline forecasting.** Roll-Up SUM maintained by Kugamon | $120,000 |
| `kuga_sub__MonthlyRecurringRevenue__c` | Monthly recurring revenue (roll-up of line MRR) | $10,000/month |
| `kuga_sub__AnnualRecurringRevenueCommitted__c` | Annual recurring revenue (roll-up of line ARR) | $120,000/year |
| `kuga_sub__NonRecurringRevenue__c` | One-time fees (roll-up) | $5,000 |
| `kuga_sub__AnnualContractValueInitial__c` | ACV formula = NonRecurring + ARR | $125,000 |
| `kuga_sub__TotalContractValue__c` | TCV formula | $120,000 (1-year) or $360,000 (3-year) |
| `kuga_sub__AnnualRecurringRevenueForecast__c` | MRR × 12 | |
| `kuga_sub__ExpectedRevenue__c` | Amount × Probability | |

### On Quote and Order

| Field | Meaning |
|---|---|
| `kugo2p__TotalAmount__c` | Total amount (formula) |
| `kugo2p__SubtotalAmount__c` | Subtotal before discounts and taxes (formula) |
| `kugo2p__NetAmount__c` | Net amount (formula) — one of the bases Kugamon can write to the Opportunity Amount |

Kugamon Settings `kugo2p__UpdateOpportunityAmountBasis__c` chooses which total the Primary Quote or Order writes to an Opportunity that has no Products: `Net Amount`, `Total Amount`, or `Total Amount + Additional Charges/Credit Amount`.

## Comparison rules

**When subscription fields exist** (`HAS_KUGA_SUB = true`):

1. Don't compare `kugo2p__TotalAmount__c` with `Amount`.
2. Compare `kugo2p__TotalAmount__c` with `kuga_sub__AnnualContractValueInitial__c` (preferred) or `kuga_sub__TotalContractValue__c`.
3. Why: `Amount` is often configured to show MRR, while quotes show annual or total values.

**When they don't exist** (`HAS_KUGA_SUB = false`): compare `kugo2p__TotalAmount__c` with `Amount` — without subscription management, `Amount` is the total opportunity value. Expected differences come from lines the sync skips (excluded, zero quantity, no Price Book Entry) and Quantity rounding; price decimals don't cause them.

### Examples

1. **Subscription org, MRR in Amount** — `Amount` $10,000; `kuga_sub__MonthlyRecurringRevenue__c` $10,000; `kuga_sub__AnnualContractValueInitial__c` $120,000; Quote `kugo2p__TotalAmount__c` $120,000. The Quote matches ACV, not Amount.
2. **Subscription org, ACV in Amount** — `Amount` and ACV both $120,000; Quote $120,000. Matches both.
3. **Non-subscription org** — `Amount` $120,000; Quote $120,000. Matches Amount.

### Best practices

1. For pipeline forecasting in subscription orgs, always use `kuga_sub__Amount__c`.
2. Query all the amount fields before comparing.
3. Show all relevant amounts in user-facing summaries.
4. Never assume what the standard `Amount` represents in a subscription org — let the data guide you.
5. Say clearly when amounts don't match the expected pattern.

## The Renew field (`kuga_sub__Renew__c`)

`kuga_sub__Renew__c` on OpportunityLineItem (and Quote and Order lines) decides whether a line is recurring or one-time revenue. It does **not** create Subscriptions — that's `kuga_sub__Track__c` on Order Service Lines.

| Renew | Treated as | Revenue goes to |
|---|---|---|
| `true` | Recurring | `kuga_sub__MRR__c` and `kuga_sub__ARR__c`; `kuga_sub__NonRecurringRevenue__c` = 0 |
| `false` or blank | One-time | `kuga_sub__NonRecurringRevenue__c`; MRR and ARR = 0 |

Opportunity roll-ups: `kuga_sub__MonthlyRecurringRevenue__c` (line MRR), `kuga_sub__AnnualRecurringRevenueCommitted__c` (line ARR), `kuga_sub__NonRecurringRevenue__c` (line non-recurring). `kuga_sub__AnnualContractValueInitial__c` = NonRecurring + ARR — the formula that double-counts when Renew is wrong.

### Double-counting

When a recurring product has `Renew = false`, the line can carry both `kuga_sub__ARR__c` and `kuga_sub__NonRecurringRevenue__c`, and ACV counts it twice:

```
Wrong:   "Annual Support Contract", UnitPrice $2,000/month, kuga_sub__Renew__c = false
         ARR $24,000 + NonRecurringRevenue $24,000 → ACV $48,000 (double-counted)

Correct: kuga_sub__Renew__c = true
         ARR $24,000 + NonRecurringRevenue $0 → ACV $24,000
```

### Which products are recurring

| Product type | Renew | Examples |
|---|---|---|
| Subscriptions | `true` | SaaS licenses, recurring services |
| Support contracts | `true` | Standard Support, Premium Support |
| Recurring retainers | `true` | Monthly consulting retainers |
| Hardware | `false` | Servers, equipment, devices |
| Implementation | `false` | Setup fees, onboarding, training |
| Professional services | `false` | Consulting hours (unless a retainer) |
| One-time licenses | `false` | Perpetual software licenses |

### Troubleshooting ACV higher than expected

```sql
SELECT kuga_sub__NonRecurringRevenue__c, kuga_sub__AnnualRecurringRevenueCommitted__c,
       kuga_sub__AnnualContractValueInitial__c
FROM Opportunity WHERE Id = '<opp_id>'

SELECT Id, Product2.Name, kuga_sub__Renew__c, kuga_sub__MRR__c, kuga_sub__ARR__c,
       kuga_sub__NonRecurringRevenue__c
FROM OpportunityLineItem WHERE OpportunityId = '<opp_id>'
```

Look for lines with all of `kuga_sub__Renew__c = false`, `kuga_sub__ARR__c > 0` and `kuga_sub__NonRecurringRevenue__c > 0`, and set `kuga_sub__Renew__c = true` on them. When a Primary Quote syncs to the Opportunity, fix Renew on the **Quote lines** instead — the sync carries Renew to the Opportunity Products.

Kugamon calculates MRR and ARR from the price and term, then sets NonRecurringRevenue from Renew. Writing `kuga_sub__NonRecurringRevenue__c` directly is overridden — Renew is the source of truth.

### Field dependencies

1. Set `kuga_sub__Renew__c` correctly on each line.
2. For recurring lines, set `kuga_sub__ServiceTerm__c` and `kuga_sub__UnitofTerm__c`.
3. Kugamon fills MRR, ARR and NonRecurringRevenue, which roll up to the Opportunity.
4. ACV = NonRecurring + ARR.
