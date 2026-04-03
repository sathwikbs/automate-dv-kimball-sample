{% docs __overview__ %}

# DV2 to Kimball Sample Project

A complete end-to-end pipeline demonstrating how to build Kimball dimensional models on top of Data Vault 2.0.

## Pipeline Layers

| Layer | Models | Description |
|-------|--------|-------------|
| 1_raw_source | 13 | Synthetic transactional data (customers, orders, products, payments, etc.) |
| 2_staging | 15 | Hashing, derived columns, record source assignment |
| 3_raw_vault | 26 | Hubs, Links, Satellites, Effectivity Satellites, Multi-Active Satellites, PIT, Bridge, XTS, Reference Tables |
| 4_business_vault | 3 | Computed satellites, deduplication, pre-resolved milestones |
| 5_kimball | 36 | Dimensions (SCD1, SCD2, pivoted, PIT-based, audit), Facts (transaction, periodic, accumulating, factless), Bridges |

## Raw Vault Assets (Layer 3)

| Asset Type | Models |
|------------|--------|
| Hub | hub_customer, hub_product, hub_order, hub_account |
| Link | link_order, link_customer_account, link_promotion_coverage |
| Satellite | sat_customer_details, sat_product_details, sat_order_details, sat_order_context, sat_order_placed, sat_order_shipped, sat_order_delivered, sat_inventory_level, sat_customer_address |
| Effectivity Satellite | eff_sat_customer_account |
| Multi-Active Satellite | ma_sat_customer_phones |
| Non-Historized Link | nh_link_payment |
| Point-In-Time | pit_customer |
| Bridge | bridge_dv2_order, bridge_cust_acct |
| XTS | xts_customer |
| Reference Table | ref_country |

## Kimball Assets (Layer 5)

### Dimensions

| Model | Pattern | DV2 Source |
|-------|---------|------------|
| dim_customer | SCD1 | hub + sat |
| dim_product | SCD2 | hub + sat + hashdiff |
| dim_customer_int_sk | Integer surrogate key | hub + sat |
| dim_customer_multi_sat | Multi-satellite | hub + 2 sats |
| dim_customer_phone | Pivoted MA-Sat | hub + ma_sat |
| dim_account_effectivity | Effectivity SCD2 | hub + eff_sat |
| dim_customer_pit | Point-in-time | hub + PIT + sats |
| dim_customer_satellite_audit | Audit | hub + XTS |
| dim_customer_with_bv | BV-enriched | hub + sat + BV computed sat |
| dim_customer_deduped | Resolved | BV deduped hub |
| dim_date | Static | seed data |
| dim_country | Reference | ref_table |

### Facts

| Model | Pattern | DV2 Source |
|-------|---------|------------|
| fact_order | Transaction | link + sats |
| fact_payment | Transaction (NH-Link) | nh_link |
| fact_inventory_daily | Periodic snapshot | hub + periodic sat |
| fact_order_fulfillment | Accumulating snapshot (2 milestones) | link + milestone sats |
| fact_order_fulfillment_3ms | Accumulating snapshot (3 milestones) | link + 3 milestone sats |
| fact_promotion_coverage | Factless | link (no measures) |
| fact_customer_account_coverage | Factless | link |

### Bridges

| Model | Pattern |
|-------|---------|
| bridge_customer_account | Simple weighted bridge |
| bridge_customer_account_simple | Unweighted bridge |
| bridge_customer_account_timevar | Time-varying bridge |

## Semantic Layer (MetricFlow)

Requires dbt >= 1.6 and dbt-metricflow.

### Semantic Models

| Model | Source | Key Measures |
|-------|--------|-------------|
| sem_customers | dim_customer | -- |
| sem_products | dim_product | unit_price |
| sem_dates | dim_date | -- |
| sem_orders | fact_order | order_amount, order_quantity, order_count |
| sem_order_fulfillment | fact_order_fulfillment | fulfillment_order_amount, order_to_ship_days |
| sem_inventory | fact_inventory_daily | quantity_on_hand (semi-additive) |
| sem_payments | fact_payment | payment_amount, payment_count |
| sem_promotions | fact_promotion_coverage | promotion_count |

### Metrics

| Metric | Type | Description |
|--------|------|-------------|
| total_revenue | simple | Sum of order amounts |
| total_orders | simple | Count of orders |
| average_order_value | derived | Revenue per order |
| total_quantity_sold | simple | Total items sold |
| revenue_per_unit | derived | Revenue per unit |
| total_payments | simple | Sum of payments |
| avg_fulfillment_days | simple | Average order-to-ship days |
| current_inventory | simple | Latest inventory on hand |
| promotion_coverage_count | simple | Count of promotion events |

## Tested on

Databricks (dbt-core 1.10.20, dbt-databricks 1.11.0)

{% enddocs %}
