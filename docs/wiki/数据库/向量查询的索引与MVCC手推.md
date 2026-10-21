# 向量查询的索引与MVCC手推

[Wiki首页](../README.md) · 前置：[向量插入的事务与WAL手推](向量插入的事务与WAL手推.md)、[MVCC](MVCC.md)

本页接着事务900插入id=42的例子，假设规划器选择了研究中的ANN索引，以普通回表索引扫描说明查询。以下是教学过程，未运行SQL、索引实现或比赛基准。

## 1. 数据与查询

表中已有已提交记录7和9；A新插入42但先不提交：

|业务id|向量|到q=(0,0,0)的L2平方|L2距离|
|---|---|---|---|
|42|(0.1,0.2,0.3)|0.14|约0.374|
|9|(0.5,0.5,0.5)|0.75|约0.866|
|7|(1,1,1)|3|约1.732|

```sql
SELECT id
FROM documents
ORDER BY embedding <-> '[0,0,0]'::vector
LIMIT 2;
```

为了只观察MVCC，本次假定ANN覆盖这3个候选，扩展能取得用于准确评分的向量，并以42、9、7的距离顺序提供候选。它不意味着所有ANN查询都覆盖全部真近邻，也不表示PG能无条件修正任意PQ估值。

## 2. 一次普通索引扫描分工

```text
SQL与规划器
  → 扩展初始化搜索，读入口和导航页
  → 图搜索、压缩估价、必要的准确重排
  → amgettuple提供下一个heap TID
  → PG表访问路径按本次快照检查行版本
  → 执行器检查其余条件，满足后输出
  → 需要更多行时继续向索引要候选
```

heap TID=(37,5)在documents表里定位42对应的行版本，不是业务id，也不是索引节点的(82,3)。PG获得TID后才能定位需要检查的表记录。

## 3. 索引读页时，缓存与页面保护怎样参与

扩展访问图节点时，通过PG缓冲接口读取节点所在页，命中shared_buffers可直接用缓存内容；否则读取关系文件。按协议pin、取得内容锁，读到邻接引用和编码后继续探索。

本教学路线的索引持久状态由PG页面管理，并不保证全数据PQ码像某些独立DiskANN配置一样始终驻内存。要实现更专门的共享码表、预取或批量I/O，需另行设计与测量。

普通扫描返回索引项并回表期间，还需与VACUUM回收协作。不能只复制TID后就忽略其关联项及位置复用的安全性；PG接口规定了相关pin与清理约束，具体AM必须遵守。

## 4. B先遇到42，但不能返回它

B使用默认Read Committed普通SELECT，其快照取得时事务900未提交：

1. 图搜索发现42对应节点；索引可计算其候选距离。
2. 扩展给出对应heap TID。
3. PG回表发现该行由尚未对本快照可见的事务900插入，排除它。
4. 再请求候选9，可见，输出第1条。
5. 再请求候选7，可见，输出第2条，满足LIMIT 2。

因此图上较近的物理节点与SQL最终结果不同。B得到9、7；并不是把未提交42当作合法近邻。

如果扩展只准备了42、9两个候选，然后就宣布结束，B可能只能得到一条。PG能继续请求，但不会代替扩展自动扩大图搜索；继续扫描或补充候选的能力必须由扩展提供，并受预算约束。

## 5. 提交与快照变化

在其他数据不变、仍覆盖这三个候选的教学条件下：

|情况|42是否可作为可见结果|示例结果|
|---|---|---|
|A插入后，在自身后续语句查询|可以看到自己的修改|42、9|
|B本条查询快照早于A提交|不可以|9、7|
|A中途提交，B原查询继续|仍按原快照，不可以|9、7|
|A提交后，B新开始一条Read Committed查询|可以|42、9|
|A改为回滚，B之后查询|不可以|9、7|

这里“可以”是可见性资格，不是ANN召回保证。Repeatable Read等隔离级别还会改变快照范围，见[MVCC](MVCC.md)。

不可见节点有时仍可作为内部导航点，不能因此把内容作为结果返回；清理它时又要避免其他边变成悬空引用，必要时维护图的可导航性。

## 6. 谁负责准确距离和返回顺序

图搜索可利用PQ/SBQ等估计距离，但用户SQL中的距离定义是另一层要求。扩展需要安排足够的候选、取得所需真实向量并按约定评分/排序；普通执行器的重检机制不是“任意估值都能自动恢复正确顺序”的开关。

尤其是ORDER BY重检接口有自己的约束，不能只设置一个标志就假定完成了量化重排。实际采用索引向量副本还是回表重排、怎样扩大候选与保持返回顺序，都需要实现验证。

ANN未找回某个合法近邻、候选内部距离算错、输出了不可见记录，是三类不同问题，必须分开定位。

## 7. PG会给哪些接口，我们需要填什么

|接口/机制|扩展要提供的内容|
|---|---|
|ambuild|扫描既有表数据，建立索引持久结构|
|aminsert|把一次插入加入索引，正确修改页面与记录日志|
|ambeginscan/amrescan/amgettuple/amendscan|初始化、推进与释放图查询状态，交出合法TID|
|amcostestimate与操作符类|声明适用查询与合理成本，帮助规划器选择|
|ambulkdelete/amvacuumcleanup|与表版本回收协作，清理或维护图|

能安装扩展不代表已经实现所有接口；实现接口也不代表正确性、并发和性能已经通过。实际查询路径需用EXPLAIN检查，不能仅因索引存在就认定查询走了它。

## 8. 现有代码怎样帮助我们入门

以下是2026-10-21静态阅读入口，不是完整能力或性能认证：

- [pgvectorscale access_method/mod.rs](https://github.com/timescale/pgvectorscale/blob/main/pgvectorscale/src/access_method/mod.rs)：amhandler如何挂接接口。
- [scan.rs](https://github.com/timescale/pgvectorscale/blob/main/pgvectorscale/src/access_method/scan.rs)：查询迭代、resort、xs_heaptid及返回项的buffer pin。
- [storage.rs](https://github.com/timescale/pgvectorscale/blob/main/pgvectorscale/src/access_method/storage.rs)：准确距离与邻接存储的抽象边界。
- [vacuum.rs](https://github.com/timescale/pgvectorscale/blob/main/pgvectorscale/src/access_method/vacuum.rs)：怎样利用PG提供的可回收判断协作清理。

可以研究复用其PG集成基础，但仍需核对PG18.4、全部类型距离、高维、AND过滤、资源与动态负载，不能将已有StreamingDiskANN名称等同本题完成。

来源：[普通索引扫描职责](https://www.postgresql.org/docs/18/index-scanning.html)、[索引AM函数](https://www.postgresql.org/docs/18/index-functions.html)、[索引与VACUUM并发规则](https://www.postgresql.org/docs/18/index-locking.html)、[PG实际TID与回表调用链](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/access/index/indexam.c)、[执行器排序重检](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/executor/nodeIndexscan.c)。
