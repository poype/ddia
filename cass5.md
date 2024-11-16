# How Cassandra reads and writes data

## 写

Cassandra中写数据的速度非常快，因为它**不需要先做磁盘读或查找操作**。Cassandra中的所有写操作都**只是追加**。

### 写操作一致性级别

ANY、ONE、TWO、THREE、LOCAL_ONE、QUORUM、LOCAL_QUORUM、EACH_QUORUM、ALL

值得注意的是ANY，它确保要写的数据至少被写入到一个副本节点中，**但hint**也可以作为一个写操作。

### 写路径

#### 节点间交互流程

coordinator 利用 partitioner 根据 **keyspace** 的副本因子确定集群中哪些节点是副本节点。

coordinator节点会**同时**向与它在同一个datacenter中的所有副本节点发出写请求。

如果集群跨多个datacenter，本地coordinator会在其它各个datacenter中分别选择一个Remote Coordinator，Remote Coordinator会负责自己所在datacenter的写操作，并向原Coordinator节点确认写操作。

Coordinator等待各个副本节点的response，一旦收到满足一致性级别的足够数量的节点的response，Coordinator就向客户端确认这个写操作。

![8dfe13147f3ae47b216f9b9c942bac14](.\image\8dfe13147f3ae47b216f9b9c942bac14.png)

#### 节点内部的交互流程

Write Path:

1. Logging data in the **commit log**.
2. Writing data to the **memtable**.
3. Flushing data from the **memtable**.
4. Storing data on disk in **SSTables**.

![dml_write-process_12](.\image\dml_write-process_12.png)

![edc6991b383491b15dd23de818bd91bf](.\image\edc6991b383491b15dd23de818bd91bf.png)

#### Commit log and memtable

When a write occurs, Cassandra stores the data in a **memory** structure called memtable.

it also appends writes to the commit log on **disk**, these durable writes survive permanently even if power fails on a node.

The memtable stores writes in **sorted order** until reaching a configurable limit, and then is flushed.

#### Flushing data from the memtable

To flush the data, Cassandra writes the data to disk, in the memtable-sorted order. 

A partition index is also created on the disk that maps the tokens to a location on disk. 

#### SSTables

Memtables and SSTables are maintained per table. The commit log is shared among tables.

SSTables are immutable, not written to again after the memtable is flushed. Consequently, a partition is typically stored across **multiple SSTable files**.

### How is data maintained?

The Cassandra write process stores data in files called SSTables. SSTables are immutable. 

Instead of overwriting existing rows with inserts or updates, Cassandra writes new **timestamped** versions of the inserted or updated data in **new** SSTables. 

Cassandra does not perform deletes by removing the deleted data: instead, Cassandra marks it with [tombstones](https://docs.datastax.com/en/glossary/doc/glossary/gloss_tombstone.html).

Over time, Cassandra may write many versions of a row in different SSTables. Each version may have a unique set of columns stored with a different timestamp. As SSTables accumulate, the distribution of data can require accessing more and more SSTables to retrieve a complete row.

To keep the database healthy, Cassandra periodically merges SSTables and discards old data. This process is called compaction.

一行中同一个列的值可能在多个SSTable文件中存在多个不同版本的值，可以合并那些SSTable文件中的数据，只保留最新版本的值即可。

Compaction works on a collection of SSTables. From these SSTables, compaction collects all versions of each unique row and assembles one complete row, using the most up-to-date version (by timestamp) of each of the row's columns.

The merge process does not use random I/O, because rows are sorted by partition key within each SSTable. 所以合并过程非常高效。

The new versions of each row is written to a **new SSTable**. 

The old versions, along with any rows that are ready for deletion, are left in the old SSTables, and are deleted as soon as pending reads are completed. As it completes, compaction frees up disk space occupied by old SSTables.

合并完成后，老的SSTables占用的磁盘空间就会被释放。这也会帮助提高读操作的性能。

### 删除

在Cassandra中，删除操作并不会真正删除数据，而是为要删除的数据加如一个墓碑标记。所以在执行删除操作后，数据库大小并不会立即缩减。

每个节点会跟踪它的所有墓碑的年龄，一旦达到`gc_grace_seconds`中配置的年龄（**默认为10天**），就会运行一个合并，将这些墓碑垃圾回收，并释放相应

的磁盘空间。

## 读

读操作通常比写操作要**慢**。

### 读一执行级别

ONE、TWO、THREE、LOCAL_ONE、QUORUM、LOCAL_QUORUM、EACH_QUORUM、ALL

如果一个节点在指定时间内没有响应一个查询，就认为这个节点读取失败。这个值默认为**5s**。

### 读路径

#### 节点间交互流程

![1738fee79e8c534cea85e116e05a3e84](.\image\1738fee79e8c534cea85e116e05a3e84.png)

注意每个datacenter中只有一个节点返回的是 full data，其它节点返回的都是数据的 digest。

coordinator会把读 full data 的请求发送给最快的副本（这个由snitch决定）。

#### 节点内交互流程

![5f03f77076b930e4824de9adc162fe88](.\image\5f03f77076b930e4824de9adc162fe88.png)

1. 先查询行缓存，如果行缓存中包含对应的数据，就直接返回。对于经常访问的行，行缓存可以提高读操作的性能。
2. 如果数据不在行缓存，就需要查询memtable 和 SSTable。
   memtable只有一个，**但一个表可能会对应多个SSTable，每个SSTable中可能包含所请求数据的一部分**。
   Cassandra实现了很多特性来优化SSTable搜索：键缓存、布隆过滤器等。
3. 一旦从所有SSTable中得到了数据，Cassandra会根据所请求各个列的最新时间戳选择值，来合并SSTable的数据和memtable数据。

### where条件要求

where子句的语法包括两个规则：

1. 必须包括分区键的所有元素。
2. 如果要限制一个给定的集群键，必须已经限制了它**之前的**所有集群键。

这是因为Cassandra基于集群列的排序顺序存储数据。

## 额外思考

### keyspace下不同的表

在Cassandra中，一个keyspace下可以包含多个不同的数据table。

对于一个键空间下的两个不同的数据表，尽管它们各自的主键或分区键不同，但如果某两行数据（分别属于不同的表）的分区键经分区算法计算出的token值相同，那么这两行数据就会被存储到相同的节点上。

注意Cassandra是基于keyspace配置的分区算法和副本因子来确定数据存储位置的，同一个keyspace下所有的数据都遵循该算法，所以不同数据表中具有相同token的行数据会被存储到同一个节点上。

### coordinator节点如何知道数据所在的目标节点

一个集群中可能会包含多个keyspace，集群中的所有节点并不一定知道所有keyspace的元数据（包括分区算法、副本因子、复制策略，查询数据表的分区键等信息）。

当一个coordinator节点接收到一个对某个keyspace数据的读写请求，如果它没有关于那个keyspace元数据信息，它会**先向集群中的其它节点询问关于那个keyspace的元数据信息**，在获取keyspace的元数据信息之后，coordinator节点就能够计算数据分区键的token值，并结合副本因子和复制策略等因素确定数据所在的目标节点。



