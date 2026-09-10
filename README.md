# Warehouse & Logistics Cost Optimization — Power BI

## Business Problem

An e-commerce logistics operation is missing on-time delivery targets and has no visibility into which levers — carrier, distance, weather, or warehouse handling — are actually driving the problem, or what it's costing in freight spend relative to order value. This project analyzes ~50,000 order-level shipping records to answer two questions:

1. **Why are deliveries late, and which factors can the business actually act on?**
2. **Is freight cost proportionate to what's being shipped, and does the worst-performing carrier at least cost less to offset its poor reliability?**

## Dataset

- Source: [E-Commerce Delivery and Shipping Data 2026](https://www.kaggle.com/datasets/datascikhan/e-commerce-delivery-and-shipping-data-2026) (Kaggle)
- ~50,000 order-level rows: carrier, shipping cost, order value, distance, delivery dates, weather condition, warehouse ID, package size, warehouse processing hours

## Data Model

- `orders` — flat fact table (all order-level fields)
- `dim_date` — built manually in Power Query (`List.Dates` / `Table.FromList`), with Year, Month, Quarter, Week, Weekday
- One active relationship: `dim_date[Date]` → `orders[order_date]`

A single flat fact table was sufficient here — the analysis didn't require a snowflaked carrier or warehouse dimension, since those attributes are already at order grain.

## Data Quality Issue: Shipping Cost Cap

While building the cost-analysis page, freight-to-value ratios spiked to nonsensical levels (3,000–5,000%) at high delivery distances. Investigation traced this to **658 orders** where `shipping_cost_usd` is capped at exactly $500 regardless of actual order value or distance — a synthetic ceiling in the source data, not real shipping economics.

**Fix:** rather than removing these rows from the dataset (which would have altered delivery-performance metrics that have nothing to do with cost), the exclusion was scoped to the cost measure only, with matching filter context on both numerator and denominator:

```dax
Corrected_FtoV2 =
DIVIDE(
    CALCULATE(SUM(orders[shipping_cost_usd]), orders[shipping_cost_usd] < 500),
    CALCULATE(SUM(orders[order_value_usd]), orders[shipping_cost_usd] < 500)
)
```

Delivery-performance metrics (Page 1) reflect the full, unfiltered dataset. Cost-ratio metrics (Page 2) exclude the 658 capped rows only.

## Page 1 — Delivery Performance

- **Overall on-time rate: 43.68%** — the headline problem.
- **Carrier is a real driver**: SwiftShip leads at 52.66% on-time, GlobalExpress trails at 34.25% — an 18.4pp spread independent of order volume.
- **Distance matters**: on-time rate drops ~7.4pp per ~975km increase in distance (verified via Key Influencers), independent of weather.
- **Weather is the strongest single driver found**: on-time rate ranges from 65.03% (Clear) down to 6.49% (Snow), with solid sample sizes verified across all six categories.
- **Quarter and warehouse_id were tested and ruled out** — both show no meaningful effect on on-time rate. Reported as genuine null results, not omitted.

## Page 2 — Cost Analysis (Freight-to-Value)

- Freight-to-value ratio (shipping cost ÷ order value) is **~65% and essentially flat across all eight carriers** (63.97%–68.75%) — including GlobalExpress.
- **Implication**: GlobalExpress's poor on-time performance is not offset by lower shipping cost. There is no cost trade-off that would justify keeping GlobalExpress despite its reliability problem.
- Freight-to-value rises with distance, consistent with normal shipping economics rather than a data anomaly (once the $500 cap rows are excluded from this measure — see above).

## Page 3 — Warehouse Processing Hours

Tested whether warehouse handling time compounds with poor carrier performance (i.e., does GlobalExpress also get slower internal handoffs) or is independent of it.

- Average processing time is **flat at 14.1–14.8 hours across all eight carriers**, with GlobalExpress sitting mid-pack (14.4h) rather than at the high end.
- Also flat across **warehouse ID** (14.1–14.6h) and **package size** (14–15h across Small/Medium/Large/Oversized).
- An initial cut by `order_value_usd` (binned) appeared to show rising variance at high order values, but was discarded after checking row counts per bin: high-value bins contained as few as 1–2 orders, meaning single outliers were driving the appearance of a trend. This is flagged here as a methodology check performed, not as a finding.
- **Conclusion (real null result, consistently confirmed across three independent cuts)**: warehouse processing time is not a driver of GlobalExpress's poor performance. The problem is isolated to carrier transit, not internal handling — which simplifies the recommendation below.

## Recommendation

Move volume away from GlobalExpress toward SwiftShip where feasible. The case is threefold:
1. GlobalExpress has the lowest on-time rate of any carrier (34.25% vs. SwiftShip's 52.66%).
2. It is not cheaper to compensate — freight-to-value is statistically the same as every other carrier.
3. The delay is not a warehouse-side handling problem that switching carriers would fail to fix — it's specific to GlobalExpress's transit performance.

## Tools

Power BI (Power Query, DAX), Kaggle dataset, manual date dimension construction.

## Notes on Methodology

This project intentionally reports several **null results** (quarter, warehouse_id on delivery performance; carrier/warehouse/package-size on processing hours) rather than omitting them. Each was tested with the same rigor as the positive findings, including sample-size checks before drawing conclusions — most notably discarding a misleading order-value-binned chart once bin sizes were found to be as small as 1–2 orders at the high end.
