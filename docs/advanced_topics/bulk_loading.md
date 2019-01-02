
34、 批量加载
---
有许多配置选项和工具可以更有效地将大量图形数据提取到JanusGraph中。与通过单个事务添加少量数据的默认事务加载相比，这种提取被称为批量加载。
有许多用例将数据批量加载到JanusGraph中，包括：
- 将JanusGraph引入具有现有数据的现有环境，并将此数据迁移或复制到新的JanusGraph集群中。
- 使用JanusGraph作为ETL过程的终点。
- 将现有或外部图形数据集（例如，公开可用的RDF数据集）添加到正在运行的JanusGraph集群。
- 使用图表分析作业的结果更新JanusGraph图表。

本页介绍了在JanusGraph中使批量加载更高效的配置选项和工具。在继续操作之前，请仔细观察每个选项的限制和假设，以避免数据丢失或数据损坏。

本文档重点介绍JanusGraph的具体优化。此外，考虑改进所选的存储后端和（可选）索引后端以实现高写入性能。有关更多信息，请参阅相应后端的文档。

### 34.1 配置选项
#### 34.1.1 批量加载

storage.batch-loading对大多数应用程序而言，启用配置选项将对批量加载时间产生最大的积极影响。启用批量加载会在许多地方禁用JanusGraph内部一致性检查。最重要的是，它禁用锁定。换句话说，JanusGraph假设要加载到JanusGraph中的数据与图形一致，因此为了提高性能而禁用自己的检查。

在许多批量加载方案中，在加载数据之前确保数据一致性然后在将数据加载到数据库中时确保数据一致性要便宜得多。storage.batch-loading由于此观察，配置选项存在。

在许多批量加载场景中，在加载数据之前确保数据一致性要比在将数据加载到数据库时确保数据一致性便宜得多。由于此观察storage.batch-loading，配置选项存在。

例如，考虑将现有用户配置文件批量加载到JanusGraph中的用例。此外，假设用户名属性键具有在其上定义的唯一复合索引，即用户名在整个图形中必须是唯一的。如果从其他数据库导入用户配置文件，则可能已保证用户名唯一性。如果没有，则按名称对配置文件进行排序并过滤掉重复项或编写执行此类过滤的Hadoop作业很简单。现在，我们可以启用storage.batch-loading哪个显着减少批量加载时间，因为JanusGraph不必检查每个添加的用户是否该名称已存在于数据库中。

要点：启用storage.batch-loading要求用户确保加载的数据在内部一致且与图中已有的任何数据一致。特别是，在启用批量加载时，并发类型创建可能会导致严重的数据完整性问题。因此，我们强烈建议您通过schema.default = none在图表配置中进行设置来禁用自动类型创建。

#### 34.1.2 优化ID分配
##### 34.1.2.1 ID块大小
每个新添加的顶点或边缘都被分配一个惟一的id。JanusGraph的id池管理器为特定的JanusGraph实例获取块中的id。id块获取过程代价高昂，因为它需要保证块的全局惟一分配。增加id块大小减少了获取的数量，但可能会留下许多id未分配，从而造成浪费。对于事务性工作负载，默认的块大小是合理的，但在批量加载期间，顶点和边的添加频率要高得多，而且速度也快得多。因此，通常建议根据每台机器要添加的顶点数量将块大小增加10倍或更多。

经验法则：设置ids.block-size为每小时每个JanusGraph实例添加的顶点数。

重要提示：所有JanusGraph实例必须配置相同的ids.block-size值以确保正确的id分配。因此，在更改此值之前，请务必关闭所有JanusGraph实例。

##### 34.1.2.2 身份识别获取流程
当许多JanusGraph实例经常并行分配id块时，实例之间的分配冲突将不可避免地出现并减慢分配过程。此外，由于批量加载而增加的写入负载可能会进一步减慢进程，使得JanusGraph认为它失败并抛出异常。有三种配置选项可以调整以避免这种情况。

1）ids.authority.wait-time配置id池管理器等待存储后端确认id块应用程序的时间（以毫秒为单位）。此时间越短，应用程序在拥塞的存储群集上失败的可能性就越大。

经验法则：将此值设置为加载时存储后端群集上测量的95%读取和写入时间的总和。 重要提示：所有JanusGraph实例的值都应相同。

2）ids.renew-timeout配置JanusGraph的id池管理器在尝试获取失败前的新id块时将等待的毫秒数。

经验法则：将此值设置为尽可能大，以便不必等待太长时间以应对不可恢复的故障。增加它的唯一缺点是JanusGraph将在不可用的存储后端群集上尝试很长时间。

#### 34.1.3 优化写入和读取
##### 34.1.3.1 缓冲区大小
JanusGraph缓冲写入并以小批量执行它们以减少针对存储后端的请求数。这些批次的大小由storage.buffer-size。在短时间内执行大量写入时，存储后端可能会因写入请求而过载。在这种情况下，增加storage.buffer-size可以通过增加每个请求的写入次数来避免失败，从而减少请求的数量。

但是，增加缓冲区大小会增加写请求的延迟及其失败的可能性。因此，不建议为事务性负载增加此设置，并且应在批量加载期间仔细尝试此设置。

##### 34.1.3.2 读写稳健性
在批量加载期间，群集上的负载通常会增加，从而使读取和写入操作更有可能失败（特别是如果缓冲区大小如上所述增加）。 storage.read-attempts并storage.write-attempts配置JanusGraph在放弃之前尝试对存储后端执行读取或写入操作的次数。如果预计在批量加载期间后端有高负载，通常建议增加这些配置选项。

storage.attempt-wait指定JanusGraph在重新尝试失败的后端操作之前将等待的毫秒数。较高的值可以确保操作重试不会进一步增加后端的负载。

### 34.2 策略
#### 34.2.1 并行化负载
如果JanusGraph的存储后端群集足够大以满足其他请求,通过并行处理跨多台计算机的批量加载，则可以大大减少加载时间。这基本上是方法[第37章，JanusGraph和TinkerPop的Hadoop-Gremlin](https://docs.janusgraph.org/latest/hadoop-tp3.html)使用MapReduce将数据批量加载到JanusGraph中。

如果Hadoop不能用于并行化批量加载过程，那么以下是一些有效并行化加载过程的高级指南：

- 在某些情况下，图表数据可以分解为多个断开连接的子图。这些子图可以在多台机器上并行独立加载（例如，如上所述使用BatchGraph）。
- 如果无法分解图形，则在多个步骤中加载通常是有益的，其中最后两个步骤可以跨多个机器并行化：
    1. 确保顶点和边缘数据集是重复数据删除且一致的。
    1. 设置batch-loading=true。可能优化上述附加配置设置。
    1. 将所有顶点及其属性添加到图形中（但没有边缘）。维护一个（分布式）映射，从顶点id（由加载的数据定义）到JanusGraph的内部顶点id（即vertex.getId()），这是一个64位长的id。
    1. 使用地图添加所有边以查找JanusGraph的顶点id并使用该id检索顶点。

34.3 问与答

- 在批量加载期间，我该怎么做才能避免以下异常： java.io.IOException: ID renewal thread on partition [X] did not complete in time.？此异常很可能是由于高度压缩的存储后端在id分配阶段重复超时造成的。请参阅上面的 ID分配优化部分。