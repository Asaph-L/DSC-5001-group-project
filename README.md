# Predicting e-commerce delivery delays at checkout

SDSC5001 Statistical Machine Learning I · Unsupervised Human Sleeping

我们用 Olist 的巴西电商订单研究下单时能多准确地预测延误，哪些结账信息影响风险排序，以及只检查风险最高的 10% 订单时能发现多少真实延误。四本 Jupyter Notebook 保留了可直接查看的汇总表和图，也可以在新环境中按顺序运行。[数据来源与字段定义](data/README.md)

## 从这里开始

| 顺序 | Notebook | 可以复现或修改的内容 |
| --- | --- | --- |
| 01 | [清洗与订单级特征](notebooks/01_build_order_table.ipynb) | 校验下载、定义延误、处理一单多件和多卖家、计算运距，输出一单一行的建模表。 |
| 02 | [训练期探索](notebooks/02_explore_training_orders.ipynb) | 检查缺失与月份变化，比较运距、承诺时长、包裹和跨州路线的延误率与样本量。 |
| 03 | [时间验证与模型比较](notebooks/03_compare_models.ipynb) | 在三折前向交叉验证中调参，比较六个分类器，并按固定 10% 预算回测。 |
| 04 | [风险因素与错误分析](notebooks/04_explain_predictions.ipynb) | 查看正则逻辑回归系数、分组置换重要性、距离与承诺时长的部分依赖，以及分组误报和漏报。 |

在仓库根目录使用 Python 3.11。使用 Anaconda 的同学可以运行以下命令。

```bash
conda create -n dsc5001-olist python=3.11 -y
conda activate dsc5001-olist
python -m pip install -r requirements.txt
python -m jupyterlab
```

在 JupyterLab 依次对 **01 → 02 → 03 → 04** 选择 **Run All Cells**。已配置 Python 3.11 虚拟环境时，从安装依赖这一步开始即可。01 首次运行会下载约 45 MB 的 Kaggle 数据。离线运行方式见[数据说明](data/README.md)。01 生成 `data/processed/orders.parquet`，03 生成 04 需要的本地预测结果和模型缓存。可共享的图表和指标保存在 [results](results/)。

## 首轮回测

99,441 笔原始订单中，96,470 笔已完成配送且日期有效。主标签按自然日判断，承诺日当天送达算准时，共有 **6,534 笔延误（6.77%）**。若以承诺日零点作截止，会多计 1,292 笔当日送达订单。[清洗记录](results/cohort_audit.csv) · [标签敏感性](results/label_definitions.csv)

按购买日期整日切分，训练、验证、测试窗口分别有 67,379、14,606、14,485 单。拟合模型和选择超参数时，只使用在当时已送达、已经知道结果的历史订单。验证集上用来选模型的订单有 12,675 笔。测试窗口是 **2018-06-21 至 2018-08-29**，共有 620 笔延误，发生率为 4.28%。[切分边界](results/purchase_splits.csv) · [标签可用性](results/label_availability.csv)

| 模型 | 验证 AP | 测试 ROC-AUC | 测试 AP | 测试 Recall@10% |
| --- | ---: | ---: | ---: | ---: |
| 普通逻辑回归 | 0.124 | 0.642 | 0.080 | 19.7% |
| Lasso 逻辑回归 | 0.119 | 0.607 | 0.071 | 16.6% |
| Ridge 逻辑回归 | 0.121 | 0.621 | 0.073 | 17.9% |
| 剪枝决策树 | 0.061 | 0.579 | 0.055 | 11.0% |
| 随机森林 | 0.099 | 0.485 | 0.054 | 15.2% |
| 梯度提升树 | 0.111 | 0.584 | 0.057 | 14.0% |

用平均精确率（AP）汇总精确率–召回率曲线，普通逻辑回归在验证集上得分最高。它在测试期将 **1,448 笔**订单列为高风险，覆盖 **122 / 620** 笔真实延误。预警命中率为 **8.43%**，是测试期整体延误率的 **1.97 倍**。三种树模型在这次时间切分上的测试 AP 均低于 Ridge。[验证指标](results/validation_metrics.csv) · [测试指标](results/test_metrics.csv) · [调参结果](results/cv_sensitivity.csv)

## 如何解读

- **筛查预算。** 每个评估窗口统一选择风险最高的 `floor(N / 10)` 笔，分数相同时按匿名订单 ID 排序。Recall@10% 的分母是窗口内全部真实延误。每日限额属于另一种运营设定。
- **预测因素。** 在验证集上分别打乱承诺时长、州与跨州信息、运距，逻辑回归的 AP 平均下降 0.058、0.049、0.033。这些字段彼此相关，降幅不能相加。训练期全部商品均跨州的 41,850 单有 8.18% 延误，同州的 22,404 单有 4.25% 延误。[分组置换结果](results/validation_group_importance.csv) · [分组样本量与延误率](results/training_package_and_route_groups.csv)
- **适用人群。** 当前结论适用于最终完成配送、能够核对日期的订单。验证时只使用截止日前已经产生的标签，临近截止日的慢单可能代表不足。模型在验证与较晚测试窗口的表现有差异。
