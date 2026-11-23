# 数据来源与预测口径

[English](README.md) | 简体中文

本项目使用 Olist 发布的 **Brazilian E-Commerce Public Dataset by Olist** 第 2 版，许可协议为 CC BY-NC-SA 4.0。数据记录了 2016 至 2018 年巴西电商平台的匿名交易。我们使用其中七张 CSV 表生成订单级数据和汇总结果。[Olist 数据说明](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) · [固定版本下载](https://www.kaggle.com/api/v1/datasets/download/olistbr/brazilian-ecommerce?datasetVersionNumber=2) · [数据许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)

## 获取数据

联网运行 [01 · 构建订单表](../notebooks/01_build_order_table.ipynb)。首次运行会把压缩包下载到 `data/raw/olist-v2.zip`。离线时，可把下载好的 ZIP 放在该位置，或把下列七个未经修改的 CSV 文件直接放入 `data/raw/`。Notebook 会按[源文件清单](source_files.json)逐一校验 SHA-256。订单明细、中间表和模型缓存保存在本地 `data/processed/`。

| 源表 | 每行代表 | 用途 |
| --- | --- | --- |
| orders | 一笔订单 | 订单状态用来筛选已完成配送的样本。购买时间和承诺日期生成结账特征。实际送达时间用于标签及其可用时间。 |
| order_items | 订单中的一个商品序号 | 关联商品、卖家、商品价格和运费。 |
| products | 一种商品 | 商品目录中的品类、重量和尺寸。 |
| customers | 一个订单关联的客户 ID | 买家所在州和邮编前缀。 |
| sellers | 一名卖家 | 卖家所在州和邮编前缀。 |
| geolocation | 某邮编前缀下的一条坐标观测 | 按前缀计算有效纬度和经度的中位数。 |
| product_category_name_translation | 一个葡萄牙语品类 | 提供英文品类名。缺少译名时沿用原品类。 |

## 如何定义延误

保留已送达且购买、承诺与实际到货日期齐全的订单。实际到货时间须在购买时刻或之后，承诺日期须在购买日或之后。按数据记录的当地时钟，**实际到货自然日大于承诺自然日**即为延误。在承诺日的任何时刻送达均算准时。样本中所有承诺时间都记录为零点，若用精确时间戳作截止，会额外把 1,292 笔当日送达的订单算作延误。[标签口径审计](../results/tables/label_definitions.csv)

结论适用于最终完成配送的订单。取消、始终未送达和客户评价需要另行定义结果。发布的数据是历史快照。我们假设商品、卖家、买家和邮编参照信息能够近似反映购买时已知的资料。

## 19 个结账特征

所有分类器都使用同一份[特征清单](checkout_features.json)。ID 用于连接表和处理预测分数并列。购买与送达时间戳决定历史样本及标签何时可用。最终状态、实际到货、发货事件、发货截止时间、付款和评价均不作为模型输入。

| 字段 | 定义 |
| --- | --- |
| `item_count`、`product_count`、`seller_count` | 商品明细行数、不同商品数和不同卖家数。 |
| `merchandise_brl`、`freight_brl`、`max_item_price_brl` | 各项商品价格之和、各项运费之和，以及最高商品单价。单位为巴西雷亚尔。 |
| `weight_kg`、`volume_l` | 商品重量之和（千克），以及按长方体计算的包裹体积之和（升）。非正的重量或尺寸视为缺失。 |
| `distance_km` | 所有卖家到买家邮编前缀之间最长的直线距离。每个前缀使用地理范围校验后经纬度各自的中位数。 |
| `promise_days` | 从购买自然日到承诺送达自然日的天数。 |
| `interstate_item_share` | 卖家与买家位于不同州的商品行占比。 |
| `purchase_month_sin`、`purchase_month_cos` | 以 12 个月为周期的购买月份正弦和余弦编码。 |
| `purchase_hour_sin`、`purchase_hour_cos` | 以 24 小时为周期的结账时间编码，包含分钟。 |
| `buyer_state`、`primary_seller_state`、`primary_category` | 买家州，以及价格最高商品的卖家州和品类。多个商品价格相同时取最小的订单内商品序号。 |
| `purchase_weekday` | 购买时是星期几。 |

如果任一商品缺少参与计算的物理量或坐标，相关总量或最长运距保留为缺失。按商品聚合后，符合条件的订单全部保留。Notebook 01 核对行数，并对账商品价格与运费总额。六个课程模型在各自的拟合样本内学习填补值、缩放参数和类别编码。CatBoost 原生处理数值缺失，对类别缺失使用固定占位值，类别统计只取自当前拟合样本。

Notebook 04 的包裹和承诺时长分组用于解释模型行为。重量和体积的门槛来自训练集四分位数。七个分类器的输入仍为同样的 19 个结账特征。[研究发现](../docs/findings.zh-CN.md) · [项目首页](../README.zh-CN.md)
