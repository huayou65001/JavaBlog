# Clickhouse

## 引擎
### 引擎的作用
1. 数据的存储位置
2. 数据组织结构
3. 是否分块、是否索引、是否持久化
4. 是否可以并发读写
5. 是否支持副本、是否支持索引
6. 是否支持分布式

### clickhouse 为什么快
1. C++可以利用硬件优势
2. 摒弃了Hadoop生态
3. 数据底层以列式数据存储
4. 利用单节点的多核并行处理
5. 为数据建立索引 一级索引、二级索引 稀疏索引
6. 使用大量算法处理数据
7. 支持向量化处理
8. 预先设计运算模型，预先计算
9. 分布式处理数据