# automate-dv-kimball-sample

A complete working example that builds a **Kimball star schema** and **semantic layer** on top of a **Data Vault 2.0** raw vault using the [automate-dv-kimball](https://github.com/sathwikbs/automate-dv-kimball) dbt package.

93 models, 269 tests, 8 semantic models, 11 metrics — the full pipeline from raw source to queryable business KPIs.

## Architecture

```
1_raw_source/       13 models   Synthetic transactional data
       |
2_staging/          15 models   Hashing, derived columns, record source
       |
3_raw_vault/        26 models   Hubs, Links, Satellites, PIT, Bridge, XTS, Ref Tables
       |
4_business_vault/    3 models   Computed satellites, deduplication, milestones
       |
5_kimball/          36 models   Dimensions (SCD1, SCD2, pivoted, PIT-based, audit)
       |                        Facts (transaction, periodic, accumulating, factless)
       |                        Bridges (simple, time-varying, weighted)
       |
semantic_layer/      4 files    MetricFlow semantic models and metrics (dbt >= 1.6)
```

## What's covered

### DV2 Asset Types (Layer 3)

| Asset | Models | Example |
|-------|--------|---------|
| Hub | 4 | hub_customer, hub_product, hub_order, hub_account |
| Link | 3 | link_order, link_customer_account, link_promotion_coverage |
| Satellite | 9 | sat_customer_details, sat_order_details, sat_order_context, ... |
| Effectivity Satellite | 1 | eff_sat_customer_account |
| Multi-Active Satellite | 1 | ma_sat_customer_phones |
| Non-Historized Link | 1 | nh_link_payment |
| Point-In-Time | 1 | pit_customer |
| Bridge (DV2) | 2 | bridge_dv2_order, bridge_cust_acct |
| XTS | 1 | xts_customer |
| Reference Table | 1 | ref_country |

### Kimball Patterns (Layer 5)

| Pattern | Models | DV2 Source |
|---------|--------|------------|
| SCD1 Dimension | dim_customer, dim_customer_no_sat, dim_customer_multi_sat | Hub + Sat(s) |
| SCD2 Dimension | dim_product, dim_product_scd2_int_sk | Hub + Sat + hashdiff |
| Integer Surrogate Key | dim_customer_int_sk | Hub + Sat |
| Pivoted MA-Sat Dim | dim_customer_phone | Hub + MA-Sat |
| Effectivity Dim | dim_account_effectivity | Hub + Eff-Sat |
| PIT-based Dim | dim_customer_pit | Hub + PIT + Sats |
| XTS Audit Dim | dim_customer_satellite_audit | Hub + XTS |
| BV-enriched Dim | dim_customer_with_bv | Hub + Sat + BV computed sat |
| Deduped Dim | dim_customer_deduped | BV same-as-link |
| Static Dim | dim_date, dim_country | Seed / Ref table |
| Transaction Fact | fact_order, fact_order_multi_sat, fact_payment | Link + Sats / NH-Link |
| Periodic Snapshot | fact_inventory_daily, fact_inventory_monthly | Hub + periodic Sat |
| Accumulating Snapshot | fact_order_fulfillment (2ms, 3ms, timespan, BV, no-sat variants) | Link + milestone Sats |
| Factless Fact | fact_promotion_coverage, fact_customer_account_coverage | Link (no measures) |
| Kimball Bridge | bridge_customer_account (simple, time-varying, weighted) | DV2 bridge |
| Bus Matrix Tests | test_bus_matrix, test_validate_star, test_generate_star | Config validation |

### Semantic Layer (MetricFlow)

| Semantic Model | Source | Measures |
|---|---|---|
| sem_customers | dim_customer | -- |
| sem_products | dim_product | unit_price |
| sem_dates | dim_date | -- |
| sem_orders | fact_order | order_amount, order_quantity, order_count |
| sem_order_fulfillment | fact_order_fulfillment | fulfillment_order_amount, order_to_ship_days |
| sem_inventory | fact_inventory_daily | quantity_on_hand (semi-additive) |
| sem_payments | fact_payment | payment_amount, payment_count |
| sem_promotions | fact_promotion_coverage | promotion_count |

**Metrics:** total_revenue, total_orders, average_order_value, total_quantity_sold, revenue_per_unit, total_payments, total_payment_count, avg_fulfillment_days, total_fulfillment_orders, current_inventory, promotion_coverage_count

## Prerequisites

- Python 3.9+
- dbt-core >= 1.0 with your adapter
- A warehouse with a schema you can write to
- For semantic layer: dbt >= 1.6 and dbt-metricflow

## Setup

### 1. Clone this repo

```bash
git clone https://github.com/sathwikbs/automate-dv-kimball-sample.git
cd automate-dv-kimball-sample
```

### 2. Configure your connection

```bash
cp profiles.yml.example ~/.dbt/profiles.yml
# Edit with your warehouse credentials
```

### 3. Install packages

```bash
dbt deps
```

### 4. Build layer by layer

```bash
dbt run --select 1_raw_source      # synthetic data
dbt run --select 2_staging         # hashing + derived columns
dbt run --select 3_raw_vault       # hubs, links, satellites
dbt run --select 4_business_vault  # derived attributes
dbt run --select 5_kimball         # star schema
```

Or build everything at once:

```bash
dbt run
```

### 5. Run tests

```bash
dbt test
```

### 6. Explore the lineage

```bash
dbt docs generate && dbt docs serve
```

### 7. Query the semantic layer (requires dbt >= 1.6)

```bash
pip install "dbt-metricflow[your-adapter]"
mf list metrics
mf query --metrics total_revenue --group-by customer__customer_name
```

## License

Apache License 2.0
