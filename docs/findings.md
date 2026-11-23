# What can a marketplace predict at checkout?

English | [简体中文](findings.zh-CN.md) · [Project home](../README.md)

We studied 96,470 Brazilian Olist orders that ultimately reached the customer. An order is late if its actual delivery date falls after the promised calendar day. Across the study, 6,534 orders were late (6.77%). The later test period ran from 21 June to 29 August 2018 and contained 620 delays among 14,485 orders (4.28%). Its lower delay rate matters when interpreting precision. [Cohort audit](../results/tables/cohort_audit.csv) · [Data definitions](../data/README.md)

## 1. How well can we predict delay, and do tree models improve on regularized logistic regression?

**Plain logistic regression achieved the highest test average precision (AP), 0.0798, and a ROC-AUC of 0.642.** Its validation AP had selected it from the original six models. Ridge was the stronger regularized comparator on validation. Every tree model, including the additional CatBoost experiment, scored below Ridge on test AP. The test delay prevalence of 0.0428 gives a reference level for AP.

| Model | Validation AP | Test ROC-AUC | Test AP | Delays caught among 620 at a 10% budget |
| --- | ---: | ---: | ---: | ---: |
| Plain logistic regression | 0.1236 | 0.642 | 0.0798 | 122 (19.7%) |
| Lasso logistic regression (L1) | 0.1193 | 0.607 | 0.0710 | 103 (16.6%) |
| Ridge logistic regression (L2) | 0.1208 | 0.621 | 0.0731 | 111 (17.9%) |
| Pruned decision tree | 0.0610 | 0.579 | 0.0548 | 68 (11.0%) |
| Random forest | 0.0990 | 0.485 | 0.0544 | 94 (15.2%) |
| Histogram gradient boosting | 0.1112 | 0.584 | 0.0570 | 87 (14.0%) |
| CatBoost with native categories | 0.0859 | 0.497 | 0.0503 | 89 (14.4%) |

All seven models received the same 19 checkout features. We tuned within three expanding training folds, selected the original model family on 12,675 validation orders with known outcomes, and refit on the 80,043 orders with outcomes known before the first test purchase. CatBoost let us test whether ordered boosting with native state and product categories added value. Training-fold AP favored depth 4 over depth 6 (0.1408 vs 0.1395). This extension reused the previously reported test period. [Full comparison](../results/tables/all_model_comparison.csv) · [Course model tuning](../results/tables/cv_sensitivity.csv) · [CatBoost tuning](../results/tables/catboost_cv.csv)

Plain logistic regression gained 0.0068 AP over Ridge. The 95% paired interval from resampling purchase weeks was −0.0003 to +0.0383, so this sample leaves the size and direction of that small gain uncertain. CatBoost's difference was −0.0228 AP (−0.0320 to −0.0084). These intervals describe variation across ten observed purchase weeks with the fits held fixed. Plain logistic regression remains the primary model for the project. [Paired intervals](../results/tables/paired_weekly_intervals.csv) · [Model comparison figure](../results/figures/test_paired_ap_differences.png)

## 2. Which checkout factors matter, and how do they interact?

**Promise length, states and border crossings, and shipping distance carried the most predictive information for the selected model.** On validation, shuffling each group reduced AP by 0.0577, 0.0489 and 0.0331, respectively. Shuffling weight and volume together changed AP by −0.0022 on this measure, providing little evidence of additional ranking value for those two fields in this fit. [Grouped permutation results](../results/tables/validation_group_importance.csv) · [Logistic coefficients](../results/tables/logistic_coefficients.csv)

Distance mattered more under shorter promises in the fitted gradient booster. With a 20-day promise, average predicted delay risk rose from 3.9% for a 79 km route to 5.7% for 1,063 km. With a 38-day promise, it rose from 2.7% to 3.4%. These are model predictions averaged over the same validation orders. [Partial dependence figure](../results/figures/validation_partial_dependence.png) · [Values and training support](../results/tables/validation_partial_dependence.csv)

The observed rates also allow a direct check of the proposal's bulky interstate shipment example. A large basket weighs at least 2.133 kg **or** occupies at least 20.02 L, using the training 75th percentile of each field. A short promise is at most 21 days. Each cell below shows late orders / eligible validation orders.

| Route and promise | Standard basket | Large basket |
| --- | ---: | ---: |
| Within state, ≤21 days | 87 / 2,613 (3.3%) | 39 / 1,007 (3.9%) |
| Within state, >21 days | 4 / 1,072 (0.4%) | 3 / 565 (0.5%) |
| Any interstate item, ≤21 days | 119 / 934 (12.7%) | 43 / 366 (11.7%) |
| Any interstate item, >21 days | 176 / 4,124 (4.3%) | 71 / 1,994 (3.6%) |

Interstate orders with short promises had the highest observed rates for both sizes. Among interstate orders, the large-minus-standard rate gap was 0.29 percentage points lower for short promises than for longer ones (95% weekly-block interval −7.15 to +6.32 points). The evidence for an additional bulky-package penalty is inconclusive. Differences in routes, products and how promises are assigned also limit causal interpretation. [Eight-group figure](../results/figures/validation_package_route_promise.png) · [Counts and rates](../results/tables/validation_package_route_promise.csv) · [Size contrasts](../results/tables/validation_package_interaction.csv)

## 3. What fraction of actual delays can a top-10% review queue catch?

**The refitted primary model caught 122 of 620 delays (19.7%) by flagging 1,448 of 14,485 test orders.** Of those alerts, 8.43% were true delays, about 1.97 times the overall test delay rate. The queue generated 1,326 false alerts and missed 498 delays. The 95% weekly-block interval for recall was 13.4% to 33.5%. [Test metrics](../results/tables/test_metrics.csv) · [Recall intervals](../results/tables/paired_weekly_intervals.csv)

| Review budget | Orders reviewed | Delays caught | Recall | Precision |
| --- | ---: | ---: | ---: | ---: |
| 5% | 724 | 88 | 14.2% | 12.2% |
| 10% | 1,448 | 122 | 19.7% | 8.4% |
| 20% | 2,897 | 193 | 31.1% | 6.7% |

Doubling the budget from 10% to 20% meant reviewing 1,449 additional orders to find 71 more delays. All seven budgets use exact whole-period quotas, with order ID breaking tied scores. [Budget curve](../results/figures/test_capture_by_budget.png) · [All model and budget combinations](../results/tables/test_screening_budgets.csv)

A separate experiment asked what a reviewer could do **as each order arrived**. We froze an earlier logistic fit before validation, scored all 14,606 validation purchases without using their outcomes, and set a cutoff to flag the top tenth of those scores. All three rows below use that same earlier model on the same test orders, so differences between rows come from the alert rule.

| When a decision is available | Orders flagged | Delays caught | Recall |
| --- | ---: | ---: | ---: |
| After the entire test window, rank all scores | 1,448 (10.0%) | 110 | 17.7% |
| At the end of each day, rank that day's scores | 1,414 (9.8%) | 96 | 15.5% |
| At checkout, apply the frozen cutoff | 1,910 (13.2%) | 130 | 21.0% |

The at-checkout cutoff (0.08078) found more delays and used more review capacity as the score distribution changed. The daily rule stayed within each day's quota but required waiting until day-end. These results measure detection and workload. The dataset cannot establish how many delays an intervention would prevent. [Screening policies and confusion counts](../results/tables/test_screening_policies.csv)
