# Hive 实战笔记

> 2026.06.14 | Spark SQL + Hive 动手记录

---

## 一、Hive 是什么

Hive 是 SQL 到 MapReduce/Spark 的翻译层。

- 数据存在 HDFS，不是 Hive 自己的存储
- 计算由 Spark 或 MapReduce 执行，不是 Hive 自己的引擎
- MetaStore 存的是「表名→HDFS 路径」的映射，相当于档案室

类比：Hive 是个翻译官——把 SQL 翻译成 Spark 能执行的代码，自己既不存数据也不算数据。

---

## 二、内部表 vs 外部表

| | 内部表 (MANAGED TABLE) | 外部表 (EXTERNAL TABLE) |
|------|------|------|
| 数据生命周期 | Hive 管理 | 用户自己管理 |
| DROP TABLE | 元数据 + HDFS 数据一起删 | 只删元数据，HDFS 文件不动 |
| 数据默认路径 | `/user/hive/warehouse/` | 任意路径 |
| 适用场景 | Hive 自己的中间表 | 已有数据文件，只需 SQL 查询入口 |

**验证实验**：用管道产出的 Parquet 建了 EXTERNAL TABLE → DROP TABLE → `SHOW TABLES` 空了 → `hdfs dfs -ls` 下 16 个 Parquet 文件完好。

**为什么用外部表**：Parquet 是 PySpark 管道生成的，Hive 只是查询入口。DROP TABLE 不应该删数据。

---

## 三、管道数据对应数仓分层

| 层 | 全称 | 在我的管道里 |
|------|------|-------------|
| ODS | Operational Data Store | Kaggle 下载的原始图片，未经处理 |
| **DWD** | **Data Warehouse Detail** | **清洗后的 495 条记录（清晰度→去重→224×224）** |
| DWS | Data Warehouse Summary | 按 label 聚合（`GROUP BY label` 统计猫狗数量） |
| ADS | Application Data Service | 下游模型训练直接取用的最终数据 |

管道输出 = DWD 层。

---

## 四、Hive 和 Spark SQL 的关系

- Spark SQL 通过 `enableHiveSupport()` 连接 Hive 的 MetaStore，可以建表、查表
- Hive 的查询引擎可替换为 Spark（Hive on Spark 模式）
- 本项目的分工：Spark 负责管道的计算和数据生成，Hive MetaStore 负责表结构的登记和 SQL 查询

---

## 五、数据观察

| 类别 | 数量 | 平均清晰度 |
|------|------|-----------|
| Cat | 248 | 973 |
| Dog | 247 | 1443 |

狗的平均清晰度比猫高约 50%。推测原因：此数据集中狗的照片更多在户外拍摄，光线和对焦条件更好。
