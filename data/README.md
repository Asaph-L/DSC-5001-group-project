# Data and prediction contract

English | [简体中文](README.zh-CN.md)

Use version 2 of the **Brazilian E-Commerce Public Dataset by Olist**, published by Olist under CC BY-NC-SA 4.0. It records anonymized Brazilian marketplace transactions from 2016–2018. This project derives an order-level table and aggregate results from seven CSVs. [Olist data card](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) · [Versioned ZIP](https://www.kaggle.com/api/v1/datasets/download/olistbr/brazilian-ecommerce?datasetVersionNumber=2) · [Data license](https://creativecommons.org/licenses/by-nc-sa/4.0/)

## Get the files

Run [01 · Build the checkout order table](../notebooks/01_build_order_table.ipynb) with internet access. It downloads the ZIP to `data/raw/olist-v2.zip` on the first run. For offline work, place the downloaded ZIP at that path, or put the seven unchanged CSVs directly in `data/raw/`. Each CSV is verified against the [source manifest](source_files.json). All intermediate order records and fitted-model caches live in `data/processed/`.

| Source table | Grain | Use |
| --- | --- | --- |
| orders | One row per order | Order status for cohort eligibility. Purchase time and promised date for checkout features. Actual delivery for the target and label-availability cutoff. |
| order_items | One row per order/item sequence | Product and seller references, item price and freight. |
| products | One row per product | Catalog category, weight and dimensions. |
| customers | One row per order-specific customer ID | Buyer state and postal prefix. |
| sellers | One row per seller | Seller state and postal prefix. |
| geolocation | Multiple coordinate observations per postal prefix | Median valid latitude and longitude for each prefix. |
| product_category_name_translation | One row per Portuguese category | English name, falling back to the source category when missing. |

## Define a late order

Keep completed deliveries with purchase, promise and arrival dates, arrival at or after purchase, and a promise date on or after the purchase day. Late means **arrival calendar day > promised calendar day**, using the timestamps as recorded. Arrival at any hour on the promised date counts as on time. All promise timestamps in the eligible cohort are at midnight. A timestamp deadline labels 1,292 extra same-day deliveries late. [Definition audit](../results/tables/label_definitions.csv)

This cohort supports comparisons among ultimately delivered orders. Cancellation, non-delivery and customer reviews would require separate outcomes. Static product, seller, buyer and postal-reference fields are assumed to approximate the information available at checkout. The release provides a historical snapshot of these attributes.

## Build the 19 predictors

The [feature allowlist](checkout_features.json) is the input contract for every classifier. IDs link tables and break tied scores. Purchase and delivery timestamps control temporal eligibility. Final status, actual arrival or shipping events, shipping deadlines, payments and reviews are outside the feature list.

| Fields | Definition |
| --- | --- |
| `item_count`, `product_count`, `seller_count` | Item lines, distinct products and distinct sellers in the order. |
| `merchandise_brl`, `freight_brl`, `max_item_price_brl` | Sum of item prices, sum of item freight charges and maximum unit price, in Brazilian reais. |
| `weight_kg`, `volume_l` | Item weights and rectangular package volumes summed across the basket. Nonpositive measurements are treated as missing. |
| `distance_km` | Longest straight-line route among all seller-to-buyer postal-prefix pairs. Each prefix uses coordinate-wise medians after the geographic check. |
| `promise_days` | Promised calendar date minus purchase calendar date, in days. |
| `interstate_item_share` | Fraction of item lines for which seller and buyer states differ. |
| `purchase_month_sin`, `purchase_month_cos` | Month of purchase on a 12-month cycle. |
| `purchase_hour_sin`, `purchase_hour_cos` | Checkout time on a 24-hour cycle, including minutes. |
| `buyer_state`, `primary_seller_state`, `primary_category` | Buyer state plus seller state and category of the highest-priced item. Lowest item sequence breaks equal-price ties. |
| `purchase_weekday` | Weekday of purchase. |

A physical total or route maximum with an unknown contributing item stays missing. All retained orders survive the item aggregation. Notebook 01 checks row counts and reconciles price and freight totals. The six course pipelines learn imputation, scaling and categorical encodings within each fit. CatBoost uses native numeric missing values and a fixed missing-category label. Its categorical statistics are learned only from its fitting rows.

The package and promise groups used in notebook 04 describe model behavior. The weight and volume cutoffs are training-set quartiles. All seven classifiers use the same 19 checkout inputs. [Research findings](../docs/findings.md) · [Project home](../README.md)
