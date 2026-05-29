# Revenue Recognition Logic — Detailed Specification

## Core principle

Revenue is recognized **proportionally over the service period** of each paid invoice. We don't recognize money on the day it's charged — we recognize it on the days the service is delivered.

Source data: every paid Stripe invoice has three fields we care about:
- `amount_paid` — money received
- `period_start` — when service begins
- `period_end` — when service ends (Stripe's convention: this is the next billing date, so the period is `[period_start, period_end)` — inclusive start, exclusive end)

## Core formula

For any paid invoice:

```txt
total_days  = days between subscription_startDate and subscription_endDate
daily_rate  = amount_paid / total_days

For each calendar month M that the period 
[subscription_startDate, subscription_endDate) touches:

    overlap_start = max(subscription_startDate, first_day_of(M))
    overlap_end   = min(subscription_endDate, first_day_of(M + 1))

    overlap_days  = number of days in 
                    [overlap_start, overlap_end)

    recognized_M  = overlap_days × daily_rate

    Write one revenue_recognition row:
        invoice_id,
        therapist_id,
        year,
        month,
        amount = recognized_M
```

The sum of all `recognized_M` values must equal `amount_paid`
(with any sub-cent rounding residual added to the final month).

### Explicit Formula

Given invoice with amount `P`, service window:

```txt
[subscription_startDate, subscription_endDate)
```

```txt
T = subscription_endDate − subscription_startDate

daily_rate = P / T
```

For each calendar month `M`:

```txt
overlap_start = max(subscription_startDate, first_day_of(M))

overlap_end   = min(
                    subscription_endDate,
                    first_day_of(M + 1)
                )

overlap_days  = max(0, overlap_end − overlap_start)

recognized_M  = overlap_days × daily_rate
```

Constraint:

```txt
Σ recognized_M over all months = P
```

(enforced by adding any sub-cent rounding residual to the last month's row)

### Example

Subscription window:

```txt
subscription_startDate = March 15
subscription_endDate   = April 15
amount_paid            = $100
```

```txt
T = 31
daily_rate = 100 / 31 = 3.2258
```

March:

```txt
overlap_start = Mar 15
overlap_end   = Apr 1

17 days × 3.2258 = $54.84
```

April:

```txt
overlap_start = Apr 1
overlap_end   = Apr 15

14 days × 3.2258 = $45.16
```

```txt
Total = $100.00 ✓
```


## Storage

New table `revenue_recognition`:

| Column | Type | Why it exists |
|---|---|---|
| `id` | uuid | Primary key. Needed for safe upserts and row-level audit operations (e.g. marking voided). Without a stable PK, idempotent re-processing would require composite deletes. |
| `stripe_invoice_id` | text | Ties this recognition row back to the source invoice. Required for the reconciliation check (`SUM(amount) == invoice.amount_paid`) and for voiding rows when an invoice is reversed. Also part of the unique constraint that makes re-processing safe. |
| `therapist_id` | text | Enables per-user revenue queries without joining back through the invoice. Every dashboard metric that breaks down by user (avg revenue per user, super-user detection) filters or groups on this column. Indexed for that reason. |
| `organization_id` | text? | Nullable — not all therapists belong to an org. Stored here so org-level revenue rollups never need to join through the invoice or subscription table. Indexed for org-scoped queries. |
| `year` | int | The calendar year this slice of revenue belongs to. Stored as a plain integer (not a timestamp) so annual rollup queries are a simple `WHERE year = 2026` with no date arithmetic. Indexed. |
| `month` | int (1–12) | The calendar month this slice belongs to. Together with `year`, this is the grain of the table — one row per (invoice, month). Storing month as an integer (not a date) keeps group-by and range queries simple and readable. Indexed. |
| `amount` | decimal(12,4) | The recognized revenue for this specific (invoice, month) slice. 4 decimal places because the daily rate produces sub-cent fractions; we store full precision here and round only at display time. This is the core value every dashboard metric sums over. |
| `voided_at` | timestamptz? | Nullable. Set when the source invoice is voided or marked uncollectible. Dashboards filter `WHERE voided_at IS NULL`. Storing a timestamp rather than a boolean keeps the audit trail — you can see exactly when the reversal happened. Rows are never hard-deleted so reconciliation queries always see the full history. |
| `created_at` | timestamptz | When this row was generated. Useful for debugging re-processing runs and auditing when recognition was first recorded. |

**Unique constraint**: `(stripe_invoice_id, year, month)` — one row per invoice per month. Idempotent on re-runs.

---

## Case 1 — Annual plan bought mid-March

**Setup**: Therapist buys $1200 annual plan on **March 15, 2026**.

- `period_start = 2026-03-15`
- `period_end   = 2027-03-15`
- `total_days   = 365`
- `daily_rate   = $1200 / 365 = $3.2877/day`

| Month | Overlap days | Recognized |
|---|---|---|
| Mar 2026 | 17 (Mar 15–31) | $55.89 |
| Apr 2026 | 30 | $98.63 |
| May 2026 | 31 | $101.92 |
| Jun 2026 | 30 | $98.63 |
| Jul 2026 | 31 | $101.92 |
| Aug 2026 | 31 | $101.92 |
| Sep 2026 | 30 | $98.63 |
| Oct 2026 | 31 | $101.92 |
| Nov 2026 | 30 | $98.63 |
| Dec 2026 | 31 | $101.92 |
| Jan 2027 | 31 | $101.92 |
| Feb 2027 | 28 | $92.06 |
| Mar 2027 | 14 (Mar 1–14) | $46.03 |
| **Total** | **365** | **$1200.00** |

**Dashboard impact for various ranges**:

| Query | Result |
|---|---|
| March 2026 alone | $55.89 |
| Apr 2026 → Mar 2027 (fiscal year ending Mar 31) | next FY gets $1144.11 |
| Last 1 month (queried Apr 30, 2026) | $98.63 |
| Last 6 months (queried Oct 1, 2026) | ≈ $497.65 |
| Last 1 year (queried Mar 14, 2027) | ≈ $1153.97 |
| Calendar year 2026 | $1108.94 |

The "fiscal year ending Mar 31" vs "last 1 year" question you raised resolves cleanly: same table, different date filter.

**DB rows written** (`stripe_invoice_id = 'inv_ann_001'`, `therapist_id = 'th_case1'`):

```sql
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-101', 'inv_ann_001', 'th_case1', 2026,  3,   55.8904),  -- 17 days
  ('rr-102', 'inv_ann_001', 'th_case1', 2026,  4,   98.6301),  -- 30 days
  ('rr-103', 'inv_ann_001', 'th_case1', 2026,  5,  101.9178),  -- 31 days
  ('rr-104', 'inv_ann_001', 'th_case1', 2026,  6,   98.6301),  -- 30 days
  ('rr-105', 'inv_ann_001', 'th_case1', 2026,  7,  101.9178),  -- 31 days
  ('rr-106', 'inv_ann_001', 'th_case1', 2026,  8,  101.9178),  -- 31 days
  ('rr-107', 'inv_ann_001', 'th_case1', 2026,  9,   98.6301),  -- 30 days
  ('rr-108', 'inv_ann_001', 'th_case1', 2026, 10,  101.9178),  -- 31 days
  ('rr-109', 'inv_ann_001', 'th_case1', 2026, 11,   98.6301),  -- 30 days
  ('rr-110', 'inv_ann_001', 'th_case1', 2026, 12,  101.9178),  -- 31 days
  ('rr-111', 'inv_ann_001', 'th_case1', 2027,  1,  101.9178),  -- 31 days
  ('rr-112', 'inv_ann_001', 'th_case1', 2027,  2,   92.0548),  -- 28 days
  ('rr-113', 'inv_ann_001', 'th_case1', 2027,  3,   46.0276);  -- 14 days, absorbs rounding residual
-- SUM = 1200.0000
```

**Validation:**
```sql
SELECT SUM(amount) FROM revenue_recognition
WHERE stripe_invoice_id = 'inv_ann_001' AND voided_at IS NULL;
-- Expected: 1200.0000
```

---

## Case 2 — Annual plan bought mid-April

**Setup**: Therapist buys $1200 annual plan on **April 15, 2026**.

- `period_start = 2026-04-15`
- `period_end   = 2027-04-15`
- `total_days   = 365`
- `daily_rate   = $3.2877/day`

| Month | Overlap days | Recognized |
|---|---|---|
| Apr 2026 | 16 (Apr 15–30) | $52.60 |
| May 2026 | 31 | $101.92 |
| Jun 2026 | 30 | $98.63 |
| Jul 2026 | 31 | $101.92 |
| Aug 2026 | 31 | $101.92 |
| Sep 2026 | 30 | $98.63 |
| Oct 2026 | 31 | $101.92 |
| Nov 2026 | 30 | $98.63 |
| Dec 2026 | 31 | $101.92 |
| Jan 2027 | 31 | $101.92 |
| Feb 2027 | 28 | $92.06 |
| Mar 2027 | 31 | $101.92 |
| Apr 2027 | 14 (Apr 1–14) | $46.03 |
| **Total** | **365** | **$1200.00** |

**Note**: This invoice is fully inside fiscal year 2026/27 (Apr 1 2026 → Mar 31 2027), so $1153.97 of it goes in that FY and only $46.03 spills into the next FY. Compare to Case 1, which split nearly evenly across two FYs. Same product, same price — but timing of purchase determines which fiscal year gets the revenue.

**DB rows written** (`stripe_invoice_id = 'inv_ann_002'`, `therapist_id = 'th_case2'`):

```sql
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-201', 'inv_ann_002', 'th_case2', 2026,  4,   52.6027),  -- 16 days (Apr 15–30)
  ('rr-202', 'inv_ann_002', 'th_case2', 2026,  5,  101.9178),
  ('rr-203', 'inv_ann_002', 'th_case2', 2026,  6,   98.6301),
  ('rr-204', 'inv_ann_002', 'th_case2', 2026,  7,  101.9178),
  ('rr-205', 'inv_ann_002', 'th_case2', 2026,  8,  101.9178),
  ('rr-206', 'inv_ann_002', 'th_case2', 2026,  9,   98.6301),
  ('rr-207', 'inv_ann_002', 'th_case2', 2026, 10,  101.9178),
  ('rr-208', 'inv_ann_002', 'th_case2', 2026, 11,   98.6301),
  ('rr-209', 'inv_ann_002', 'th_case2', 2026, 12,  101.9178),
  ('rr-210', 'inv_ann_002', 'th_case2', 2027,  1,  101.9178),
  ('rr-211', 'inv_ann_002', 'th_case2', 2027,  2,   92.0548),
  ('rr-212', 'inv_ann_002', 'th_case2', 2027,  3,  101.9178),
  ('rr-213', 'inv_ann_002', 'th_case2', 2027,  4,   46.0274);  -- 14 days (Apr 1–14), absorbs residual
-- SUM = 1200.0000
```

---

## Case 3 — Monthly plan bought mid-March

**Setup**: Therapist buys $100/month plan on **March 15, 2026**.

### Initial invoice (paid Mar 15)

- `period_start = 2026-03-15`
- `period_end   = 2026-04-15`
- `total_days   = 31`
- `daily_rate   = $100 / 31 = $3.2258/day`

| Month | Overlap days | Recognized |
|---|---|---|
| Mar 2026 | 17 | $54.84 |
| Apr 2026 | 14 | $45.16 |
| **Total** | **31** | **$100.00** |

**DB rows written** (`stripe_invoice_id = 'inv_mo_001'`, `therapist_id = 'th_case3'`):

```sql
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-301', 'inv_mo_001', 'th_case3', 2026, 3, 54.8387),  -- 17 days
  ('rr-302', 'inv_mo_001', 'th_case3', 2026, 4, 45.1613);  -- 14 days
-- SUM = 100.0000
```

### Renewal invoice (paid May 15)

- `period_start = 2026-05-15`
- `period_end   = 2026-06-15`
- `total_days   = 31`
- `daily_rate   = $100 / 31 = $3.2258/day`

| Month | Overlap days | Recognized |
|---|---|---|
| May 2026 | 17 (May 15–31) | $54.84 |
| Jun 2026 | 14 (Jun 1–14) | $45.16 |
| **Total** | **31** | **$100.00** |

**DB rows written** (`stripe_invoice_id = 'inv_mo_002'`, `therapist_id = 'th_case3'`):

```sql
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-303', 'inv_mo_002', 'th_case3', 2026, 5, 54.8387),  -- 17 days
  ('rr-304', 'inv_mo_002', 'th_case3', 2026, 6, 45.1613);  -- 14 days
-- SUM = 100.0000
```

**Validation — May total for th_case3:**
```sql
SELECT SUM(amount) FROM revenue_recognition
WHERE therapist_id = 'th_case3' AND year = 2026 AND month = 5 AND voided_at IS NULL;
-- Expected: 54.8387 (only the renewal invoice contributes to May)
```

### What May actually shows

May 2026 recognized revenue from this therapist:
- From original Mar 15 invoice (Apr 15 tail carries into Apr, not May)
- From May 15 renewal invoice: $54.84
- **Total May: $54.84**

Note: the billing period varies by calendar month — March and May have 31 days, April has 30. The formula handles this automatically; `daily_rate` is always divided by the actual number of days in the billing window so the total always reconciles to exactly $100.00 regardless of period length. This is why monthly subscribers' recognized revenue is roughly flat each month — each renewal naturally fills the partial month left by the previous invoice.

---

## Case 4 — Cancellation mid-period (no refund)

**Setup**: Therapist on monthly $100 plan paid Mar 15. Cancels Mar 25.

Stripe behavior:
- `cancel_at_period_end = true` set on subscription.
- Service continues through `period_end` (Apr 15).
- **No refund issued** (confirmed in `paymentController.ts`).
- No new invoice fires on Apr 15.

Our handling: **no change**. The original $100 invoice continues to amortize through its `period_end` as in Case 3 initial invoice:

| Month | Recognized |
|---|---|
| Mar 2026 | $54.84 |
| Apr 2026 | $45.16 |

**DB rows written** — identical to Case 3 initial invoice. No rows are added, modified, or deleted when the cancellation fires.

```sql
-- Already exists from invoice.paid. Cancellation writes nothing.
SELECT * FROM revenue_recognition WHERE stripe_invoice_id = 'inv_mo_003';
-- Returns 2 rows (Mar + Apr), both unchanged, voided_at = NULL
```

**Rationale**: The therapist had access through Apr 15, paid for that service window, and didn't get refunded. We delivered (or made available) the service. GAAP says we recognize over the contracted period regardless of usage.

---

## Case 5 — Upgrade mid-period (e.g., Plus → Pro)

**Setup**: Therapist on monthly Plus ($30) paid Mar 15. Upgrades to monthly Pro ($60) on Apr 5.

Stripe's behavior (with `proration_behavior: 'always_invoice'`, which is what the upgrade code does):
1. Original Plus invoice already exists: $30 for Mar 15 → Apr 15.
2. A **proration invoice** fires on Apr 5 with two line items:
   - Credit for unused Plus time (Apr 5 → Apr 15, 10 days at Plus rate): **−$10**
   - Charge for prorated Pro time (Apr 5 → Apr 15, 10 days at Pro rate): **+$20**
   - **Net invoice total: $10**, period roughly Apr 5 → Apr 15.
3. Next renewal on Apr 15 fires as a full Pro monthly: $60 for Apr 15 → May 15.

### Our handling: recognize each invoice on its own

**Invoice A — original Plus** ($30, Mar 15 → Apr 15):
| Month | Recognized |
|---|---|
| Mar 2026 | $16.45 |
| Apr 2026 | $13.55 |

**Invoice B — proration** ($10, Apr 5 → Apr 15):
| Month | Recognized |
|---|---|
| Apr 2026 | $10.00 |

**Invoice C — full Pro renewal** ($60, Apr 15 → May 15):
| Month | Recognized |
|---|---|
| Apr 2026 | $32.00 |
| May 2026 | $28.00 |

**April total recognized from this therapist**: $13.55 + $10.00 + $32.00 = **$55.55**.

**DB rows written** (three invoices, same `therapist_id = 'th_case5'`):

```sql
-- Invoice A: original Plus ($30, Mar 15 → Apr 15)
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-501', 'inv_plus_001', 'th_case5', 2026, 3, 16.4516),  -- 17 days
  ('rr-502', 'inv_plus_001', 'th_case5', 2026, 4, 13.5484);  -- 14 days

-- Invoice B: proration net charge ($10, Apr 5 → Apr 15)
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-503', 'inv_prorate_001', 'th_case5', 2026, 4, 10.0000);  -- 10 days, single month

-- Invoice C: full Pro renewal ($60, Apr 15 → May 15)
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-504', 'inv_pro_001', 'th_case5', 2026, 4, 32.0000),  -- 16 days
  ('rr-505', 'inv_pro_001', 'th_case5', 2026, 5, 28.0000);  -- 14 days
```

**Validation — April total for th_case5:**
```sql
SELECT SUM(amount) FROM revenue_recognition
WHERE therapist_id = 'th_case5' AND year = 2026 AND month = 4 AND voided_at IS NULL;
-- Expected: 13.5484 + 10.0000 + 32.0000 = 55.5484
```

**Why this is right enough**: Stripe's proration math ensures the *net* invoice amounts are correct. We let each invoice amortize over its own `period_start → period_end`. There can be tiny per-month rounding noise vs. strict daily-rate accounting because Stripe uses 30-day-month proration assumptions while we use actual day counts, but the totals always reconcile. We do **not** try to back out the credit line and reassign it to the original Plus invoice — that's needlessly complex for sub-dollar accuracy.

---

## Case 6 — Downgrade mid-period (Pro → Plus)

**Setup**: Therapist on monthly Pro ($60) paid Mar 15. Downgrades to Plus on Apr 5.

Stripe's behavior (with `proration_behavior: 'none'`, which is what the downgrade code does):
- **No invoice fires today.** No charge, no refund.
- The subscription's price ID is updated immediately in Stripe, but the user keeps Pro access through Apr 15.
- On Apr 15, a new Plus monthly invoice fires at $30 for Apr 15 → May 15.

### Our handling

**Original Pro invoice** ($60, Mar 15 → Apr 15) continues to amortize as planned:
| Month | Recognized |
|---|---|
| Mar 2026 | $32.90 |
| Apr 2026 | $27.10 |

**New Plus invoice** ($30, Apr 15 → May 15):
| Month | Recognized |
|---|---|
| Apr 2026 | $16.00 |
| May 2026 | $14.00 |

**April total recognized**: $27.10 + $16.00 = **$43.10**.

**DB rows written** (two invoices, `therapist_id = 'th_case6'`):

```sql
-- Original Pro invoice ($60, Mar 15 → Apr 15) — written on Mar 15, untouched by downgrade
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-601', 'inv_pro_002', 'th_case6', 2026, 3, 32.9032),  -- 17 days
  ('rr-602', 'inv_pro_002', 'th_case6', 2026, 4, 27.0968);  -- 14 days

-- New Plus renewal ($30, Apr 15 → May 15) — written on Apr 15
INSERT INTO revenue_recognition (id, stripe_invoice_id, therapist_id, year, month, amount)
VALUES
  ('rr-603', 'inv_plus_002', 'th_case6', 2026, 4, 16.0000),  -- 16 days
  ('rr-604', 'inv_plus_002', 'th_case6', 2026, 5, 14.0000);  -- 14 days
```

**Validation — April total for th_case6:**
```sql
SELECT SUM(amount) FROM revenue_recognition
WHERE therapist_id = 'th_case6' AND year = 2026 AND month = 4 AND voided_at IS NULL;
-- Expected: 27.0968 + 16.0000 = 43.0968
```

No reconciliation gymnastics needed — downgrades just produce a smaller renewal invoice.

---

## Case 7 — Billing period switch (Monthly ↔ Yearly)

**Setup**: Therapist on Pro Monthly ($60) paid Mar 15. Switches to Pro Yearly ($600) on Apr 5.

Per the billing docs, this is **scheduled**, not charged today. The user finishes their current month (through Apr 15), then on Apr 15 the new yearly plan is billed at $600.

### Our handling

**Original Monthly Pro invoice** ($60, Mar 15 → Apr 15) — amortizes normally:
| Month | Recognized |
|---|---|
| Mar 2026 | $32.90 |
| Apr 2026 | $27.10 |

**Scheduled yearly invoice** fires Apr 15 ($600, Apr 15 → Apr 15 2027):

`daily_rate = $600 / 365 = $1.6438`

| Month | Days | Recognized |
|---|---|---|
| Apr 2026 | 16 | $26.30 |
| May 2026 | 31 | $50.96 |
| Jun 2026 | 30 | $49.32 |
| ... | ... | ... |
| Apr 2027 | 14 | $23.01 |
| **Total** | **365** | **$600.00** |

**No special logic needed**. The scheduled change just produces a new invoice on the future date, which is then amortized.

---

## Case 8 — Yearly to Monthly switch mid-period

**Setup**: Therapist on Yearly Pro ($600) paid Mar 15, 2026. Switches to Pro Monthly on Aug 1, 2026.

Per the billing docs:
- **No refund.**
- User keeps yearly access through Mar 15, 2027.
- On Mar 15, 2027 the new monthly invoice fires at $60.

### Our handling

**Original Yearly invoice** ($600, Mar 15 2026 → Mar 15 2027) amortizes in full as in Case 1.

**Monthly invoice** ($60, Mar 15 2027 → Apr 15 2027) starts the new cadence.

The switch on Aug 1 has **zero effect** on recognition until Mar 15 2027.

---

## Case 9 — Payment failed

**Setup**: Renewal attempt fails. `invoice.payment_failed` fires.

**Our handling**: **Do nothing for recognition.** Only `invoice.paid` triggers the recognition schedule. The `stripe_invoice` row gets updated to `status = open` with the failure recorded, but no `revenue_recognition` rows are created.

If Stripe retries and eventually succeeds → `invoice.paid` fires → recognition schedule generated at that point, using the invoice's actual `period_start`/`period_end` (which is still the original service window).

If Stripe gives up and voids → invoice never paid → recognition never created. Correct.

---

## Case 10 — Refund (future / out of scope)

Per the current code, the product **does not issue refunds**. There's no `charge.refunded` handler, no refund endpoint, and the downgrade flow explicitly states no refund.

When refund support eventually ships, the handler will reverse recognition rows from the refund date forward, while leaving already-recognized (already-delivered service) periods intact. We'll spec that when it's needed.

---

## Edge cases and decisions

### Rounding

The daily rate produces fractional cents. Two rules:

- Store `revenue_recognition.amount` with **4 decimal places** (matches the precision needed without overflow risk). Display rounds to 2.
- **Largest-remainder reconciliation**: after computing all months for an invoice, if the sum differs from `amount_paid` by less than 1 cent due to rounding, adjust the **last month's** amount to make the sum exact. Guarantees `SUM(recognized) = amount_paid` per invoice.

### Leap years

If a 365-day period crosses Feb 29, total_days becomes 366. The formula handles this automatically — `daily_rate` divides by 366, and February gets 29 days of overlap. No special-case code needed.

### Month-end edge cases

Stripe handles "Jan 31 → Feb 28" anchor-shifting at the subscription level. We just consume `period_start` and `period_end` as given. The formula doesn't care if the period is 28, 29, 30, 31, 365, or 366 days.

### Timezone

All recognition is computed in **UTC**. The `period_start` and `period_end` from Stripe are UTC timestamps. Year/month assignment uses UTC. (This matches the existing `getAdminAnalytics` code.)

### Idempotency

The `revenue_recognition` table has `UNIQUE(stripe_invoice_id, year, month)`. The generator uses `UPSERT`, so reprocessing the same `invoice.paid` event (which Stripe will sometimes do) produces zero net changes.

### Reconciliation check

A periodic job verifies, for every paid invoice:

```
SUM(revenue_recognition.amount WHERE stripe_invoice_id = X) == stripe_invoice.amount_paid
```

Any mismatch beyond 1 cent gets flagged for a human. This catches generator bugs early.

### Invoice voided after payment (rare)

If Stripe voids a previously-paid invoice (e.g., chargeback), the `stripe_invoice` row goes to `status = void` and all its `revenue_recognition` rows get a `voided_at` timestamp. Dashboards filter `WHERE voided_at IS NULL`. Don't hard-delete — keep the audit trail.

**Example** — `inv_mo_001` (Case 3 initial invoice) gets voided via chargeback:

```sql
-- Before: two rows, voided_at = NULL
-- After invoice.voided fires:
UPDATE revenue_recognition
SET voided_at = '2026-05-10 14:23:00Z'
WHERE stripe_invoice_id = 'inv_mo_001' AND voided_at IS NULL;
-- 2 rows updated (Mar + Apr slices)

-- Dashboard query now returns nothing for this invoice:
SELECT SUM(amount) FROM revenue_recognition
WHERE stripe_invoice_id = 'inv_mo_001' AND voided_at IS NULL;
-- Returns NULL (no rows pass the filter)

-- Audit query still sees the full history:
SELECT id, year, month, amount, voided_at FROM revenue_recognition
WHERE stripe_invoice_id = 'inv_mo_001';
-- rr-301 | 2026 | 3 | 54.8387 | 2026-05-10 14:23:00Z
-- rr-302 | 2026 | 4 | 45.1613 | 2026-05-10 14:23:00Z
```

**What sets `voided_at`:** Two webhook events trigger the update, handled in `stripeWebhookController.ts`:

- `invoice.voided` — Stripe marks the invoice void (e.g., manually voided or chargeback settled). Handler runs:
  ```sql
  UPDATE revenue_recognition
  SET voided_at = NOW()
  WHERE stripe_invoice_id = :invoiceId AND voided_at IS NULL;
  ```
- `invoice.marked_uncollectible` — Stripe gives up collecting (final retry failed). Same UPDATE as above.

The `AND voided_at IS NULL` guard makes both handlers fully idempotent — if the webhook fires twice, the second execution matches zero rows and produces no side effects.

### Duplicate webhooks

Stripe guarantees at-least-once delivery. Any event can fire more than once. Protection per event type:

| Event | Protection mechanism |
|---|---|
| `invoice.paid` | `UPSERT` on `(stripe_invoice_id, year, month)` — second fire overwrites with identical values, net change = zero |
| `invoice.voided` | `UPDATE … WHERE voided_at IS NULL` — second fire matches zero rows |
| `invoice.marked_uncollectible` | Same as voided |
| `invoice.payment_failed` | Only updates `stripe_invoice.status` and `attempt_count`; no recognition rows touched |
| `invoice.finalized` | Only upserts `stripe_invoice` row — idempotent by PK |

No distributed lock or deduplication table is needed. The unique constraint on `revenue_recognition` and the null-guarded UPDATE on `voided_at` are sufficient.

---

## What we generate per Stripe event

| Stripe event | stripe_invoice action | revenue_recognition action |
|---|---|---|
| `invoice.paid` | Upsert row, status = paid, amount_paid set | Generate all month rows |
| `invoice.payment_failed` | Upsert row, status = open, attempt_count++ | None |
| `invoice.voided` | Update status = void | Mark all rows voided_at = now |
| `invoice.marked_uncollectible` | Update status = uncollectible | Mark all rows voided_at = now |
| `invoice.finalized` (not yet paid) | Upsert row, status = open | None |
| `charge.refunded` (future) | Update amount_refunded | Future: reverse forward-dated rows |

---

## Dashboard queries

### A note on time window filters

The existing analytics dashboard supports rolling windows (last 30, 60, 90 days). These work perfectly for the `cost` table which has a precise `created_at` timestamp. The `revenue_recognition` table is bucketed by `(year, month)`, so rolling-window filters align to monthly boundaries — "last 30 days" maps to the current and previous calendar month. This is intentional: recognized revenue is a monthly accounting concept and sub-monthly slicing would produce misleading partial-month figures.

For all queries below, replace the date filter with whatever window applies:
- **Last N months**: `WHERE (year * 12 + month) >= ((EXTRACT(YEAR FROM NOW()) * 12 + EXTRACT(MONTH FROM NOW())) - N)`
- **Calendar year**: `WHERE year = 2026`
- **Fiscal year (Apr–Mar)**: `WHERE (year = 2026 AND month >= 4) OR (year = 2027 AND month <= 3)`

---

### 1. Recognized revenue vs AI cost incurred (monthly breakdown)

**What it answers:** "Are we making or losing money each month, and by how much?"

This is the core P&L chart. It compares how much revenue we recognized (from `revenue_recognition`) against how much we actually spent on AI (from `cost`) for the same month. The two sources are independent — revenue comes from Stripe invoices spread over time, cost comes from real-time OpenAI/transcription spend — so they need to be joined by calendar month.

The FULL OUTER JOIN is important: if a month has AI costs but no recognized revenue (e.g., free users were active before any paid invoices landed), it still shows up with `recognized_revenue = 0`. Same the other way. You get a complete picture with no gaps.

Output columns: `year`, `month`, `recognized_revenue`, `ai_cost`, `gross_profit`. A negative `gross_profit` means that month cost more to operate than it earned.

```sql
WITH monthly_revenue AS (
  SELECT
    year,
    month,
    SUM(amount)::float AS revenue
  FROM revenue_recognition
  WHERE voided_at IS NULL
    -- replace with your window filter
    AND year = 2026
  GROUP BY year, month
),
monthly_cost AS (
  SELECT
    EXTRACT(YEAR  FROM c.created_at)::int AS year,
    EXTRACT(MONTH FROM c.created_at)::int AS month,
    SUM(c.cost)::float                    AS cost
  FROM cost c
  JOIN session s ON s.id = c.session_id
  WHERE s.deleted = false
    AND c.created_at >= '2026-01-01'
  GROUP BY 1, 2
)
SELECT
  COALESCE(r.year,  c.year)  AS year,
  COALESCE(r.month, c.month) AS month,
  COALESCE(r.revenue, 0)     AS recognized_revenue,
  COALESCE(c.cost,    0)     AS ai_cost,
  COALESCE(r.revenue, 0) - COALESCE(c.cost, 0) AS gross_profit
FROM monthly_revenue r
FULL OUTER JOIN monthly_cost c USING (year, month)
ORDER BY year, month;
```

---

### 2. Super users — paying $40+ but costing $40+ in AI

**What it answers:** "Which specific users are costing us as much or more than they pay us?"

A super user is a paying therapist whose AI usage bill (transcription, summaries, chat, etc.) is so high that it eats into or exceeds what they pay in subscription revenue. These users are individually unprofitable. The query surfaces them ranked by worst net loss first so you can investigate — are they running unusually long sessions, using a feature excessively, or is this a pricing problem?

The `revenue >= 40` filter excludes free users intentionally. A free user costing $5 in AI is expected and priced in. A Pro user ($60/month) who costs $80 in AI is the problem this query is designed to catch.

Output columns: `therapist_id`, `revenue` (what they paid this month), `cost` (what their AI usage cost us), `net_loss` (how much we lost on them — higher = worse).

```sql
WITH period_revenue AS (
  SELECT therapist_id, SUM(amount)::float AS revenue
  FROM revenue_recognition
  WHERE voided_at IS NULL
    AND year = 2026 AND month = 5   -- replace with target period
  GROUP BY therapist_id
),
period_cost AS (
  SELECT
    s.therapist_id,
    SUM(c.cost)::float AS cost
  FROM cost c
  JOIN session s ON s.id = c.session_id
  WHERE s.deleted = false
    AND DATE_TRUNC('month', c.created_at) = '2026-05-01'
  GROUP BY s.therapist_id
)
SELECT
  r.therapist_id,
  r.revenue,
  c.cost,
  (c.cost - r.revenue)::float AS net_loss
FROM period_revenue r
JOIN period_cost c ON r.therapist_id = c.therapist_id
WHERE r.revenue >= 40        -- only paid users above threshold
  AND c.cost    >= r.revenue -- cost equals or exceeds what they pay
ORDER BY net_loss DESC;
```

---

### 3. Average AI cost per user — monthly and yearly

**What it answers:** "On average, how much are we spending on AI per active user?"

This tells you the unit economics of your AI spend. It uses active users (those who actually incurred a cost) as the denominator rather than total registered users — if you used total users, months with low activity would look artificially cheap because idle users drag the average down without contributing any cost.

Track this over time to spot if AI costs per user are growing faster than revenue per user. If this number trends up while query 4 (avg subscription value) stays flat, your margins are compressing.

Output columns: `year`, `month` (monthly variant) or `year` (yearly variant), `avg_cost_per_user` in dollars.

```sql
-- Monthly
SELECT
  EXTRACT(YEAR  FROM c.created_at)::int                       AS year,
  EXTRACT(MONTH FROM c.created_at)::int                       AS month,
  SUM(c.cost)::float / COUNT(DISTINCT s.therapist_id)::float  AS avg_cost_per_user
FROM cost c
JOIN session s ON s.id = c.session_id
WHERE s.deleted = false
GROUP BY 1, 2
ORDER BY 1, 2;

-- Yearly
SELECT
  EXTRACT(YEAR FROM c.created_at)::int                        AS year,
  SUM(c.cost)::float / COUNT(DISTINCT s.therapist_id)::float  AS avg_cost_per_user
FROM cost c
JOIN session s ON s.id = c.session_id
WHERE s.deleted = false
GROUP BY 1
ORDER BY 1;
```

---

### 4. Average subscription revenue per paying user — monthly and yearly

**What it answers:** "What is our average revenue per paying user (ARPU)?"

This is the revenue side of the unit economics picture. It counts only users who have recognition rows — meaning only users with at least one paid invoice. Free users never generate recognition rows so they're excluded from the denominator automatically, keeping this a true ARPU figure rather than a blended number that includes non-paying users.

Pair this with query 3 (avg AI cost per user) to compute implied gross margin per user: if ARPU is $45 and avg AI cost is $8, your per-user gross margin is roughly 82%. Watch for the gap narrowing over time.

Output columns: `year`, `month` (monthly) or `year` (yearly), `avg_subscription_value` in dollars.

```sql
-- Monthly
SELECT
  year,
  month,
  SUM(amount)::float / COUNT(DISTINCT therapist_id)::float AS avg_subscription_value
FROM revenue_recognition
WHERE voided_at IS NULL
GROUP BY year, month
ORDER BY year, month;

-- Yearly
SELECT
  year,
  SUM(amount)::float / COUNT(DISTINCT therapist_id)::float AS avg_subscription_value
FROM revenue_recognition
WHERE voided_at IS NULL
GROUP BY year
ORDER BY year;
```

---

### 5. Average profit per user — global blended, monthly and yearly

**What it answers:** "Across all users — paying and free — how much profit do we make per user on average?"

Unlike query 4 which only looks at paying users, this is a fully blended number. It computes profit (revenue minus AI cost) per user first, then averages across everyone — including free users who have $0 revenue but real AI costs. Those users drag the average down, which is intentional: they represent a genuine cost of running the product and should show up in the profitability picture.

This is the number to watch at the business level. If it's negative, you're losing money on average across your entire user base. If it's positive and growing, the business is healthy. A sudden drop here might mean free usage spiked, a paid cohort churned, or AI costs jumped.

The FULL OUTER JOIN between revenue and cost subqueries ensures users who appear in only one table (e.g., a free user with AI cost but no revenue) still get included with the missing side defaulting to zero.

Output columns: `year`, `month` (monthly) or `year` (yearly), `avg_profit_per_user` in dollars (can be negative).

```sql
-- Monthly
WITH user_monthly AS (
  SELECT
    COALESCE(r.year,         EXTRACT(YEAR  FROM c.created_at)::int)  AS year,
    COALESCE(r.month,        EXTRACT(MONTH FROM c.created_at)::int)  AS month,
    COALESCE(r.therapist_id, s.therapist_id)                          AS therapist_id,
    COALESCE(r.revenue, 0) - COALESCE(c.cost, 0)                     AS profit
  FROM (
    SELECT year, month, therapist_id, SUM(amount)::float AS revenue
    FROM revenue_recognition
    WHERE voided_at IS NULL
    GROUP BY year, month, therapist_id
  ) r
  FULL OUTER JOIN (
    SELECT
      EXTRACT(YEAR  FROM c.created_at)::int AS year,
      EXTRACT(MONTH FROM c.created_at)::int AS month,
      s.therapist_id,
      SUM(c.cost)::float                    AS cost
    FROM cost c
    JOIN session s ON s.id = c.session_id
    WHERE s.deleted = false
    GROUP BY 1, 2, s.therapist_id
  ) c ON r.year = c.year AND r.month = c.month AND r.therapist_id = s.therapist_id
)
SELECT
  year,
  month,
  AVG(profit)::float AS avg_profit_per_user
FROM user_monthly
GROUP BY year, month
ORDER BY year, month;

-- Yearly (same structure, drop the month grain)
WITH user_yearly AS (
  SELECT
    COALESCE(r.year, c.year)                 AS year,
    COALESCE(r.therapist_id, c.therapist_id) AS therapist_id,
    COALESCE(r.revenue, 0) - COALESCE(c.cost, 0) AS profit
  FROM (
    SELECT year, therapist_id, SUM(amount)::float AS revenue
    FROM revenue_recognition
    WHERE voided_at IS NULL
    GROUP BY year, therapist_id
  ) r
  FULL OUTER JOIN (
    SELECT
      EXTRACT(YEAR FROM c.created_at)::int AS year,
      s.therapist_id,
      SUM(c.cost)::float AS cost
    FROM cost c
    JOIN session s ON s.id = c.session_id
    WHERE s.deleted = false
    GROUP BY 1, s.therapist_id
  ) c USING (year, therapist_id)
)
SELECT
  year,
  AVG(profit)::float AS avg_profit_per_user
FROM user_yearly
GROUP BY year
ORDER BY year;
```

---

### 6. Average AI cost per user broken down by service type — monthly and yearly

**What it answers:** "Which features are driving AI spend, and how much does each cost us per user?"

Instead of a single blended AI cost number, this breaks it down by what the cost was actually for — transcription, session summaries, chat responses, clinical note generation, or vector store operations. This is useful for identifying which features are disproportionately expensive and informing decisions like rate-limiting, feature gating by plan tier, or switching providers for a specific service type.

For example, if `TRANSCRIPTION` costs $6/user/month but `CHAT` costs $0.50, and you're considering a price increase, transcription is the lever to focus on — not chat.

Service types map to the `CostType` enum: `TRANSCRIPTION`, `SUMMARY`, `CHAT`, `CLINICAL_SUMMARY`, `VECTOR_STORE`.

Output columns: `year`, `month` (monthly) or `year` (yearly), `service` (the cost type), `avg_cost_per_user` in dollars. One row per (period, service type) combination.

```sql
-- Monthly — one row per (year, month, service type)
SELECT
  EXTRACT(YEAR  FROM c.created_at)::int                      AS year,
  EXTRACT(MONTH FROM c.created_at)::int                      AS month,
  c.type                                                      AS service,
  SUM(c.cost)::float / COUNT(DISTINCT s.therapist_id)::float AS avg_cost_per_user
FROM cost c
JOIN session s ON s.id = c.session_id
WHERE s.deleted = false
GROUP BY 1, 2, c.type
ORDER BY 1, 2, c.type;

-- Yearly
SELECT
  EXTRACT(YEAR FROM c.created_at)::int                       AS year,
  c.type                                                      AS service,
  SUM(c.cost)::float / COUNT(DISTINCT s.therapist_id)::float AS avg_cost_per_user
FROM cost c
JOIN session s ON s.id = c.session_id
WHERE s.deleted = false
GROUP BY 1, c.type
ORDER BY 1, c.type;
```

---

## Reconciliation queries

```sql
-- Last calendar month
SELECT SUM(amount) FROM revenue_recognition
WHERE (year, month) = (2026, 4) AND voided_at IS NULL;

-- Fiscal year Apr 2026 → Mar 2027
SELECT SUM(amount) FROM revenue_recognition
WHERE ((year = 2026 AND month >= 4) OR (year = 2027 AND month <= 3))
  AND voided_at IS NULL;

-- Last 6 months ending Oct 2026
SELECT SUM(amount) FROM revenue_recognition
WHERE (year, month) IN ((2026,5),(2026,6),(2026,7),(2026,8),(2026,9),(2026,10))
  AND voided_at IS NULL;

-- Per-therapist revenue this fiscal year
SELECT therapist_id, SUM(amount) FROM revenue_recognition
WHERE ((year = 2026 AND month >= 4) OR (year = 2027 AND month <= 3))
  AND voided_at IS NULL
GROUP BY therapist_id ORDER BY SUM(amount) DESC;
```

---

## Summary table — what gets recognized in April 2026

For perspective, if all 10 cases above happened to different therapists in the same period:

| Case | April recognition |
|---|---|
| 1. Annual $1200, bought Mar 15 | $98.63 |
| 2. Annual $1200, bought Apr 15 | $52.60 |
| 3. Monthly $100, bought Mar 15, renewed Apr 15 | $98.49 |
| 4. Monthly $100, bought Mar 15, cancelled Mar 25 | $45.16 |
| 5. Upgrade Plus → Pro on Apr 5 | $55.55 |
| 6. Downgrade Pro → Plus on Apr 5 | $43.10 |
| 7. Switch to yearly scheduled Apr 5, billed Apr 15 | $53.40 |
| 8. Switch yearly → monthly, scheduled, no Apr impact | $49.32 (yearly continues) |
| 9. Payment failed | $0 |
| 10. Refund — N/A | — |

Total April recognized = $496.25. Compare to Stripe `amount_paid` in April (cash collected) which would be very different — that's the whole point of deferred revenue.
