# Predicting e-commerce delivery delays at checkout

English | [简体中文](README.zh-CN.md)

SDSC5001 Statistical Machine Learning I · Unsupervised Human Sleeping

Which orders should a marketplace review if it can check only one in ten at checkout? We study 96,470 completed Brazilian Olist deliveries using 19 features available when the order was placed. An order is late when it arrives **after the promised calendar day**. This labels 6,534 deliveries late (6.77%). [Data source and feature dictionary](data/README.md)

## What the project answers

| Proposal question | Evidence |
| --- | --- |
| How well can checkout features predict delay, and do trees improve on regularized logistic regression? | Across seven models, plain logistic regression had the highest test average precision (AP), 0.0798. Ridge reached 0.0731. The three course tree models and CatBoost all scored below Ridge. |
| Which risk factors matter, and how do they interact? | The promised window, states and shipping distance carried the most ranking information. Under promises of 21 days or less, 12.7% of standard interstate orders and 11.7% of large interstate orders arrived late. The additional effect of package size is uncertain. |
| How many actual delays can a 10% alert budget catch? | The original validation-selected logistic model flagged 1,448 of 14,485 test orders and caught 122 of 620 delays (19.7%). The follow-up compares other budgets and a cutoff that can be applied as each order arrives. |

Read the answers with their denominators, comparison models and limits. [Findings by research question](docs/findings.md)

## Run the notebooks

Run these commands from the repository root to create a Python 3.11 environment with Anaconda.

```bash
conda create -n dsc5001-olist python=3.11 -y
conda activate dsc5001-olist
python -m pip install -r requirements.txt
python -m jupyterlab
```

In JupyterLab, run all cells in each notebook in this order. Notebook 01 downloads the versioned Olist ZIP (about 45 MB) the first time. It can also read a local copy. [Offline data setup](data/README.md)

| Order | Notebook | Main output |
| --- | --- | --- |
| 01 | [Build the checkout order table](notebooks/01_build_order_table.ipynb) | Checks the input hashes and join grain, builds the target and features, and assigns whole purchase days to train, validation and test. |
| 02 | [Explore training orders](notebooks/02_explore_training_orders.ipynb) | Examines coverage, monthly variation, route distance, promise length and package characteristics. |
| 03 | [Compare the six course models](notebooks/03_compare_models.ipynb) | Tunes each family in three expanding folds, selects on validation AP and measures later test performance. Saves local scores for notebook 05. |
| 04 | [Explain risk and missed delays](notebooks/04_explain_predictions.ipynb) | Measures feature importance, joint risk patterns and errors at the validation-wide alert budget. |
| 05 | [Compare CatBoost and screening policies](notebooks/05_catboost_and_screening.ipynb) | Reuses the original folds, compares all seven models with paired week resampling, and tests budget curves and a checkout cutoff. |

## Results and reproducibility

- [Figures](results/figures/) contains the PNG charts. [Tables](results/tables/) contains aggregate CSVs and the JSON record of the original model selection.
- The chronological windows contain 67,379 training, 14,606 validation and 14,485 test purchases. At each fitting or selection cutoff, only completed historical outcomes are eligible. The test window runs from 21 June to 29 August 2018. [Split boundaries](results/tables/purchase_splits.csv) · [Available labels](results/tables/label_availability.csv)
- The primary model was selected before the original test readout. The CatBoost and alert-policy extensions reuse that already reported test period. Individual orders, fitted models and order-level scores are generated locally under `data/processed/`. [Seven-model comparison](results/tables/all_model_comparison.csv)
