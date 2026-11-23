# 在结账时预测电商订单配送延误

[English](README.md) | 简体中文

SDSC5001 统计机器学习 I · Unsupervised Human Sleeping 小组

如果平台只能在结账时检查十分之一的订单，应该优先检查哪些？本项目使用巴西 Olist 的 96,470 笔已完成配送订单，以及下单时可获得的 19 个特征。**实际送达日期晚于承诺日期**即记为延误。按此口径，共有 6,534 笔延误，占 6.77%。[数据来源与字段字典](data/README.zh-CN.md)

## 项目回答了什么

| 提案中的研究问题 | 对应证据 |
| --- | --- |
| 结账信息能多准确地预测延误？树模型比正则化逻辑回归提高了多少？ | 七个模型中，普通逻辑回归测试集平均精确率（AP）最高，为 0.0798。Ridge 为 0.0731。三个课程树模型和 CatBoost 的测试 AP 均低于 Ridge。 |
| 哪些因素影响预测？它们如何交互？ | 承诺期、州与跨州情况、运距对风险排序最重要。承诺期不超过 21 天时，跨州普通件和大件的实际延误率分别为 12.7% 和 11.7%。大件是否带来额外风险仍不确定。 |
| 检查风险最高的 10% 能发现多少真实延误？ | 最初由验证集选出的普通逻辑回归在 14,485 笔测试订单中标记了 1,448 笔，命中 620 笔延误中的 122 笔（19.7%）。扩展实验比较了其他预算，以及每笔订单结账时可直接使用的固定阈值。 |

三个问题的具体答案、分母、比较对象和适用范围见[研究发现](docs/findings.zh-CN.md)。

## 运行 Notebook

在仓库根目录使用 Python 3.11。使用 Anaconda 时，可运行以下命令。

```bash
conda create -n dsc5001-olist python=3.11 -y
conda activate dsc5001-olist
python -m pip install -r requirements.txt
python -m jupyterlab
```

在 JupyterLab 中按下列顺序运行每一本的全部单元格。01 首次运行会下载约 45 MB 的固定版本 Olist 压缩包，也支持读取本地数据。[离线数据准备](data/README.zh-CN.md)

| 顺序 | Notebook | 主要产出 |
| --- | --- | --- |
| 01 | [构建结账时点的订单表](notebooks/01_build_order_table.ipynb) | 核验源文件哈希和连接粒度，构建标签与特征，按完整购买日划分训练、验证和测试窗口。 |
| 02 | [探索训练期订单](notebooks/02_explore_training_orders.ipynb) | 检查特征覆盖率、月度变化、运距、承诺时长和包裹特征。 |
| 03 | [比较六个课程模型](notebooks/03_compare_models.ipynb) | 用三折扩展窗口调参，按验证 AP 选模型，再评估较晚测试期。保存供 05 使用的本地逐单预测。 |
| 04 | [解释风险和漏报](notebooks/04_explain_predictions.ipynb) | 分析特征重要性、多因素组合风险，以及在验证期统一预警预算下的误报和漏报。 |
| 05 | [比较 CatBoost 和预警策略](notebooks/05_catboost_and_screening.ipynb) | 复用原有时间折，按周配对重抽样比较七个模型，检验不同预算和结账时固定阈值的表现。 |

## 结果与复现

- PNG 图存放在[图件目录](results/figures/)，汇总 CSV 和记录原模型选择的 JSON 文件存放在[表格目录](results/tables/)。
- 训练、验证、测试三个购买窗口分别有 67,379、14,606、14,485 笔订单。拟合和选模时只使用截至当时已完成配送的历史标签。测试期为 2018 年 6 月 21 日至 8 月 29 日。[切分边界](results/tables/purchase_splits.csv) · [可用标签数](results/tables/label_availability.csv)
- 主模型在首次查看测试结果前已经由验证集选定。后加的 CatBoost 和预警策略复用这段已公布结果的测试窗口。逐单数据、模型缓存和逐单分数在本地的 `data/processed/` 下生成。[七模型比较表](results/tables/all_model_comparison.csv)
