# DiskANN与Vamana：图、压缩和SSD怎样组合

[Wiki首页](../README.md) · 前置：[HNSW](HNSW.md)、[量化与重排序](量化与重排序.md)、[磁盘与内存](../数据库/磁盘与内存.md)

DiskANN的经典磁盘方案把图搜索、压缩表示、缓存和SSD读取一起设计；Vamana是其中的图算法。DiskANN也是一个实现库的名字，不是数据库。

## 为什么不能只把图写成文件

如果下一步去哪个节点，必须先等当前节点从磁盘读回，搜索路径就会串起很多等待。只减少距离计算次数，还不足以让SSD上的搜索快。

经典方案可以先用三部分建立直觉：

|位置/组件|作用|
|---|---|
|内存中的压缩表示|便宜地估计候选距离，帮助判断值得访问谁|
|SSD上的详细节点数据|保存图邻接信息及更完整的向量信息，布局依版本而定|
|缓存和读取调度|复用常见节点，安排多个读取请求，减少等待影响|

压缩表示本身仍然占内存，不能认为用了DiskANN就不再受内存限制。

## Vamana先理解到什么程度

它组织一张可导航的近邻图。构图时需要寻找邻居候选，再进行选边/剪枝；目标不仅是连近点，还要保留能覆盖不同方向的导航能力，并控制度数。

可以把它理解成修路：一堆非常近、方向又重复的道路，未必比少量覆盖不同方向的道路更好。具体的RobustPrune规则、alpha参数和搜索队列，留到手推及源码专题展开。

它与HNSW都属于图方法，但不能当作“去掉几层的HNSW”直接替换全部构图和搜索逻辑。

## 一次查询的概念流程

1. 从入口及缓存可用信息开始。
2. 利用压缩表示对候选排序或估计。
3. 选择有希望的节点，读取所需详细数据。
4. 用邻接信息扩展候选，继续搜索。
5. 根据可用的准确表示形成结果。

这是架构直觉，实际算法不应直接照此五行实现。读取批次、停止条件、量化和最终评分方式都要锁定版本研究。

## 三个相似名字不能混用

- 微软DiskANN库包含磁盘与内存相关实现，必须看实际调用接口。
- pgvectorscale的StreamingDiskANN是数据库扩展里的实现，不是微软库的任意版本直接改名。
- VIBE公开Vamana适配调用内存索引，因此相关成绩不能当SSD测试成绩。

## 与本题的关系

这是重要候选研究方向，但独立库的图搜索能力不自动提供PG事务、WAL、MVCC和规划器集成。实际技术路线要先确认硬要求覆盖，再测性能。

## 延伸阅读

[Vamana构图与RobustPrune手推](Vamana构图与RobustPrune手推.md)：对比α=1与1.5的选边，区分HNSW与Vamana的构建和搜索。

[DiskANN内存与SSD查询手推](DiskANN内存与SSD查询手推.md)：沿小图追踪短码估距、缓存、并行读盘与准确评分。

[PostgreSQL与索引扩展](../数据库/PostgreSQL与索引扩展.md) · [MVCC](../数据库/MVCC.md) · [WAL](../数据库/WAL.md) · [公开基准现状](../../公开基准现状.md)

版本提醒：2026-10-21核对时，官方main已是新的Rust架构，旧C++在cpp_main分支且标为非活跃维护。本Wiki手推明确区分2019论文、经典C++路径和当前项目，不把教学原理当作最新实现的逐行说明。

来源：[DiskANN原论文](https://papers.nips.cc/paper_files/paper/2019/hash/09853c7fb1d3f8ee67a61b6bf4a7f8e6-Abstract.html)、[实现库](https://github.com/microsoft/DiskANN)、[VIBE内存版适配](https://github.com/vector-index-bench/vibe/blob/main/vibe/algorithms/diskann/module.py)、[pgvectorscale](https://github.com/timescale/pgvectorscale)。
