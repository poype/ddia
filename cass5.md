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



# 事务

## 轻量级事务

Cassandra的轻量级事务（lightweight transaction， LWT）机制使用了 **Paxos** 算法。它支持以下语义：

- 在一个INSERT上，增加`IF NOT EXISTS`子句将确保不会覆盖相同主键的已存在的行。可以用于幂等性保证。
- 在一个UPDATE上，增加`IF EXISTS`子句确保只会更新已存在的行。限制upsert行为。
- **乐观锁**。在一个UPDATE上，增加`IF <conditions>`子句会检查一个或多个指定的条件，只有当所有条件都满足时，才会执行UPDATE操作。
  多个条件分别用一个 AND 分隔。

**如果批处理包含轻量级事务，那所有写操作仅限于在单个分区**。

## 批处理

Cassandra还提供了一种批处理（batch）机制，允许把**多个修改操作分组到一个语句**，而不论它们要处理的是同一个分区还是不同的分区。

```CQL
BEGIN BATCH
INSERT INTO table1(xxx, xxx) VALUES ();
INSERT INTO table2(xxx, xxx) VALUES ();
INSERT INTO table3(xxx, xxx) VALUES ();
APPLY BATCH;
```

将三个INSERT语句打包到一个batch中，确保写操作的原子性。

批处理可以分为有日志（logged）和 无日志（unlogged）两种类型。

有日志的批处理（logged batches）具有原子性，即只要批处理被接受，那么批处理中的所有语句最终都会成功。所以有日志的批处理有时也被称为原子批处理（atomic batches）。

在底层，批处理的工作如下：

1. coordinator节点将批处理的请求发送给集群中另外两个节点，那两个节点会把批处理请求存储在`system.batchlog`表中。当coordinator节点执行完批处理后，会从那两个节点上删除对应的batchlog。
2. 如果coordinator节点未能成功完成批处理，由于有另外两个节点也存储了批处理的信息，所以能够重放这个批处理。每个节点每分钟检查一次自己的batchlog，查看是否有需要重放的批处理。用额外两个节点保存批处理请求的信息更能确保批处理的可靠性。

在无日志批处理中，就不涉及上述batchlog的步骤。

所以无日志的批处理不能保证对不同分区的所有写操作都能成功完成，这可能会让数据库出于一种不一致状态。

但无日志的批处理仍然可以确保对单一分区修改的原子性。

当你请求一个有日志批处理，如果其中只包含对一个分区的修改，那么Cassandra在执行时实际上会把它作为一个无日志批处理，以此来提供额外的速度提升。

### 与传统事务的比较

批处理与传统关系型数据中的事务并不相同，批处理的原子性只能保证最终一致性，它并不能提供隔离性等特性，所以在批处理执行过程中，可能会查询到“脏数据”，或者在某个时间窗口内存在数据不一致。其实有日志批处理的实现原理就可以简单理解成事务补偿。

但对于**同一个分区中的所有修改操作**，无日志批处理也可以保证其原子性。

#### Cassandra中的写操作原子性

**为什么无日志批处理也能保证单个分区内所有写操作的原子性？**

- Cassandra在行级别支持原子性和隔离性。
- 写操作在分区级别上是原子的（所以分区级别只能保证最终一致性）。

这意味着，对于同一分区中的多行写操作，Cassandra会将它们视为一个单一的写操作来处理。如果这些写操作中的任何一个失败，整个写操作都会被视为失败，并且不会部分提交。

注意：The commit log is shared among tables. commit log是针对分区维度的，所有写操作要么被同时加入commit log，要么一个都无法被加入到commit log



如果在批处理中（无论是有日志还是无日志）包含轻量级事务语句，那么一个批处理中的多个轻量级事务必须应用到同一个分区。



Cassandra的有日志批处理在处理跨多个分区的写操作时，可以确保最终一致性。即使存在一个数据不一致的时间窗口，也能在机制保障下确保所有写操作最终成功完成。Cassandra可以通过副本机制、时间戳机制和读修复机制等确保所有写操作最终都能成功完成，并达到最终一致性。

但感觉最好还是尽量不要有跨分区的事务级操作，跨分区的事务级操作比较复杂。





​				
