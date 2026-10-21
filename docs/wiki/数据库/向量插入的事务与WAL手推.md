# 向量插入的事务与WAL手推

[Wiki首页](../README.md) · 前置：[向量索引的PG页面与缓存](向量索引的PG页面与缓存.md)、[事务](事务.md)、[WAL](WAL.md) · 下一篇：[向量查询的索引与MVCC手推](向量查询的索引与MVCC手推.md)

本页用普通、持久化的表与索引说明一次插入。布局和事务编号是教学示例；SQL、并发与恢复实验尚未运行，不表示项目已经实现DiskANN扩展。

## 1. 会话A插入一条向量

假设documents已有id和vector(3)列，以及要研究的向量索引。id=42此前不存在：

```sql
BEGIN;
INSERT INTO documents (id, embedding)
VALUES (42, '[0.1,0.2,0.3]'::vector);
-- 先不提交，后面分别讨论COMMIT和ROLLBACK
```

假定事务编号为900，PG先把物理行版本放到表中，得到heap TID=(37,5)，并管理其事务信息；然后通过索引的aminsert等接口维护对应索引。这不是先独立写一个ANN文件、再希望数据库事务自动接上。

## 2. 索引插入需要做什么

扩展收到向量值和heap TID后，可进行以下概念步骤：

1. 在已有图中搜索邻居候选，执行所选构图/剪枝策略。
2. 分配索引节点空间，例如位置(82,3)，保存到表TID=(37,5)的关联、导航表示及本例所需的原向量信息。
3. 在受保护的页上写好节点，更新邻接关系和必要元信息。
4. 给每类持久修改记录所需WAL；处理缓存和结构发布。

必须防止其他查询跟随一条边读到未初始化记录。具体的页分配、先初始化后发布、并发锁顺序与跨页恢复协议需要设计；PG不知道一张图怎样才算完整。

## 3. 复用Generic WAL，不是重写整个日志系统

PG有供扩展使用的通用页变更日志机制。一个受保护的修改单元可概括为：

```text
pin目标缓冲页并取得适当排他内容锁
  → GenericXLogStart
  → GenericXLogRegisterBuffer，取得临时页副本
  → 修改注册得到的副本
  → GenericXLogFinish，应用页变化并产生WAL记录
  → 按协议释放锁与pin
```

走Generic WAL时要改注册得到的临时副本，不能绕过它直接修改原缓冲内容。Finish负责相关标脏/LSN等处理，但调用者仍负责保护与协议。

**GenericXLogFinish不是SQL COMMIT，也不意味着此刻已经强制把日志全部写入稳定存储。** 生成日志记录与持久化日志是两个动作。

一条Generic WAL记录能注册的页数有上限。图修改涉及多个独立记录时，不能认定任意日志前缀都天然构成合法图；恢复过程的结构一致性仍是扩展责任。其他WAL设计也可研究，但不能省略保障。

## 4. 此时另一个会话能不能看到42

节点可能已在索引物理结构中。普通查询若找到它，仍需回表，按当前查询快照检查事务900是否对自己可见。

- A在之后的一条语句中通常能看到自己已插入的记录。
- B在A尚未提交时的普通查询不能把42作为已提交结果返回。
- 内部导航是否经过这个物理节点，与最终是否返回其表记录，是不同问题。

更完整的扫描过程见[查询手推](向量查询的索引与MVCC手推.md)。

## 5. 分支一：A提交

正常同步提交与可靠持久化配置下，PG等待所需WAL，包括提交记录，持久化后确认成功。相关表页、索引页可以稍后写回；若之前写回，也必须满足对应WAL先持久化的约束。

提交后，B新开始的Read Committed普通查询可在MVCC上看到42；已经取得旧快照的查询不会因此中途换成新快照。能看见只表示资格，ANN是否找回它还取决于搜索。

崩溃时若某些已提交索引页尚未写回，恢复可以依据完整WAL重做。前提是扩展确实正确记录了节点、边和元信息等修改，并设计了合法恢复状态。

## 6. 分支二：A回滚

如果改成ROLLBACK，事务900不会成为已提交事务，42不能作为有效插入结果返回。**这不要求把每条已写入的图边都逐条物理倒回。**

普通PG索引中，失败插入留下物理条目并不必然错误：行版本可见性负责拒绝非法结果，后续清理负责安全回收。但图不能因此出现损坏、悬空引用，或因随意删除桥梁严重影响其他合法节点。

GenericXLogAbort只取消尚未Finish的临时页变更；它不是SQL ROLLBACK对所有既有图修改的统一undo。重启重做了某个物理节点，也不等于它关联的事务已经提交。

## 7. 哪些事情PG不会代替我们完成

PG负责表行版本、事务状态、提交、WAL存储与重做基础。扩展负责图节点与TID的对应、页操作的日志记录、并发发布、量化/邻接缓存一致性，以及后续图维护。

VACUUM提供判断哪些表版本可安全回收的机制和索引回调，但扩展仍须安全清理索引引用、必要时修补图，并遵循扫描与回收的协作规则。heap位置以后可能复用，不能把TID当永久业务编号缓存。

## 8. 可以从现有代码学什么

pgvector的[hnswinsert.c](https://github.com/pgvector/pgvector/blob/master/src/hnswinsert.c)可用于理解PG页、buffer与Generic WAL如何围绕索引插入配合；不是让我们把HNSW算法直接改名为DiskANN。

pgvectorscale的[build.rs](https://github.com/timescale/pgvectorscale/blob/main/pgvectorscale/src/access_method/build.rs)能追踪aminsert_internal、创建节点与graph.insert；[page.rs](https://github.com/timescale/pgvectorscale/blob/main/pgvectorscale/src/util/page.rs)能追踪页副本与Generic WAL包装。

其中WritablePage::commit只是完成页修改，不是提交SQL事务。此次所读插入入口还使用事务级advisory锁做串行化保护，不能凭“有PG集成”就认定已验证高写入并发。

来源：[PG索引插入接口](https://www.postgresql.org/docs/18/index-functions.html)、[表与索引的更新次序](https://www.postgresql.org/docs/18/index-locking.html)、[Generic WAL](https://www.postgresql.org/docs/18/generic-wal.html)、[Generic WAL实现](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/access/transam/generic_xlog.c)、[WAL原理](https://www.postgresql.org/docs/18/wal-intro.html)。代码阅读核查日期2026-10-21，移动分支不等于最终锁定版本，尚未做编译或恢复验证。
