# BIGANN与评测框架

[Wiki首页](../README.md) · 前置：[概念地图](概念地图.md)、[召回率与性能评测](召回率与性能评测.md)

数据集是考题材料，评测框架是运行测试和统计成绩的工具。BIGANN与big-ann-benchmarks的名字相近，但角色不同。

## 一个检索数据集通常包含什么

- base：被搜索的向量集合。
- queries：查询向量。
- ground truth，简称GT或真值：按规定距离得到的精确邻居/距离，用来评估答案。
- 有时还有学习索引参数的样本、标签或动态操作序列。

这里的“训练样本”可能用来学习聚类中心或PQ码本，不自动意味着要训练文本embedding模型。

## 本题中的几个名字

|名字|用途|
|---|---|
|BIGANN / SIFT1B相关数据|核心ANN的10M、100M、1B规模，128维L2|
|simplewiki-openai-3072|高维搜索，具体文件版本和条数待核对|
|YFCC-10M|带1—2标签AND条件的过滤搜索|
|MSTuring-30M-clustered|按规定插入、删除、查询的动态负载|
|ann-benchmarks|ANN数据、算法适配和曲线评测工具|
|VIBE|现代embedding的算法基准|
|big-ann-benchmarks|大规模、过滤和动态等任务的评测框架|

runbook是操作手册，可看成一份YAML任务序列：“插入这些点→删除那些点→在此处查询并计分”。同一数据配不同runbook，不一定产生可比成绩。

## 本题当前的边界

公开数据能用于提前验证，正式PG适配器、调度脚本、VM配置及框架提交仍待官方提供。上游能跑通不代表已通过本题正式评测。

VIBE当前目录的normalized simplewiki变体为260,372条，而题面写1M；本次读取的公开结果K=100，而题面要求recall@10。具体差异与原始选点见[公开基准现状](../../公开基准现状.md)。

## 容易混淆的地方

不要把数据集名当数据库名；不要把框架默认参数当比赛要求；不要将数据同名、不同版本或不同编码的成绩直接比较。

## 延伸阅读

[第14题全貌](../第14题全貌.md) · [待确认事项](../../待确认事项.md)

来源：[Big-ANN数据定义](https://github.com/harsha-simhadri/big-ann-benchmarks/blob/main/benchmark/datasets.py)、[NeurIPS23说明](https://github.com/harsha-simhadri/big-ann-benchmarks/blob/main/neurips23/README.md)、[VIBE](https://github.com/vector-index-bench/vibe)。
