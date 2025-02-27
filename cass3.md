## Cassandra数据库建模与RDBMS的区别

#### 1. Cassandra没有Join操作。

如果必须要完成某种join操作，就必须在客户端完成这个工作，或者另外创建一个反规范化的表存储join操作的结果。Cassandra数据建模更倾向于采用后一种做法。

#### 2. 没有外键

Cassandra的表没有引用完整性的概念，不支持诸如级联删除等操作。

#### 3. 反规范化

规范化的表结构对Cassandra并不是一个优点。在Cassandra中，反规范化相当正常。

即使关系型数据库往往也会采用反规范化的表结构，这通常有两个原因。一个是出于性能考虑，当数据量很大时join操作的性能很差，所以会根据已知查询的结果反规范化。另一个出于业务需求，例如订单表中要保存商品信息的一个快照。

#### 4. 查询优先设计

关系型建模要从概念领域入手，将领域中的模型用数据表来表示，并指定主键和外键表示模型间的关系。

对于关系模型，查询是次要的。一般认为，只要适当地建立的表，就总能得到想要的查询，尽管可能会使用很多复杂的查询SQL，但这一点总是成立的。

Cassandra与之相反，Cassandra的表设计并不是从领域模型开始。**要从查询模型入手**。

要先对查询建模，然后根据查询来组织数据。

先考虑你的应用最常使用的查询路径，然后创建支持这些查询所需的表。

#### 5. 设计最优存储

设计Cassandra的表结构时，一个关键目标是：**为满足一个给定查询所需搜索的分区要尽可能少**。

分区是一个不会跨节点划分的存储单位，搜索单个分区的查询性能最好。

#### 6. 排序会影响表的设计

在Cassandra中，不能使用`order by`对结果任意排序。查询中可用的排序顺序是固定的，这完全由集群列确定。

CQL SELECT语句确实也支持ORDER BY，不过**只能按集群列指定的顺序（升序或降序）排列**。

## 逻辑数据建模

先定义有哪些查询，每个查询由一个表来支持其数据来源。

1. 对各个表命名。先识别查询的主实体，以此作为表明的开头。再识别出查询条件中的属性，把这些属性名追加到表明后面，中间用"by"分隔。例如`hotels_by_poi`。
2. 识别主键，确定好分区键和集群键。一定要确保唯一性，否则就会有意外覆盖数据的风险。
   使用集群列存储区间查询中用到的那些属性。
3. 把查询结果中包含的所有其它属性都加到表中。
4. 如果某些属性对单个分区中的所有行都一样，则将那些属性定义为静态列。

## 二级索引

默认情况下，Cassandra是不支持根据非主键列进行查询的。

```CQL
CREATE INDEX ON hotels (name)
```

这行命令会创建名为`hotels_name_idx`的二级索引。

创建索引后，下面的查询就能正常工作了：

```CQL
SELECT * FROM hotels WHERE name = 'xxxx';
```

除了简单类型外，还可以基于用户自定义类型或存储在集合中的值创建索引。

删除一个索引的命令：

```CQL
DROP INDEX hotels_name_idx;
```

使用二级索引进行查询通常会涉及更多节点，这会大大增加这些查询的开销。

二级索引的原理可以简单理解成：

Cassandra为二级索引创建了一张隐藏表，在这张隐藏表中存储了二级索引到一级索引的映射关系。Cassandra会自动维护这个隐藏表中的数据。

## 物化视图

引入物化视图是为了解决二级索引的一些缺点。用二级索引作为条件查询可能需要查询环中的大多数甚至全部节点。

每个物化视图支持不属于原主键的**一个**列上的查询。Cassandra会负责更新物化视图，保证它们与基表一致。

物化视图会对基表写操作的性能造成一些影响，因为需要保持基表与物化视图间的一致性。不过，相对于在应用客户端管理多张反范式化表，物化视图更高效。

在Cassandra内部，会使用**批处理**实现物化视图的更新。

物化视图的PK必须包含基表主键中的所有列。由于有这个限制，所以Cassandra不允许把基表中的多行折叠为物化视图中的一行（即无法聚合），否则会大大增加管理更新的复杂性。

截至目前，物化视图的主键不能包含基表的多个非主键列，只能包含一个非主键列。



