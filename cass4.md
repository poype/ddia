## 集群架构

Cassandra常用在**地理位置分散**的系统中。

Cassandra提供了两层分组来描述一个集群的拓扑：**数据中心（data center）和机架（rack）**。

<img src=".\image\datacenter.png" alt="datacenter" style="zoom: 80%;" />

## Gossip

Gossip是一种点对点（peer-to-peer）通信协议。Cassandra利用Gossip协议跟踪集群中所有**节点的状态信息**，以此来实现故障检测。

节点间通过Gossip协议周期性地交换信息，交换的信息既包括节点**自己的状态信息**，也包括它所了解到的**其它节点的状态信息**。

The gossip process runs **every second** and exchanges state messages with **up to three** other nodes in the cluster.

The nodes exchange information about **themselves** and about the **other nodes** that they have gossiped about, so **all nodes quickly learn about all other nodes in the cluster**. 

A gossip message has a **version** associated with it, so that during a gossip exchange, older information is **overwritten** with the most current state for a particular node.

### Failure detection and recovery

Failure detection is a method for locally determining **from gossip state and history** if another node in the system is down or has come back up. Cassandra uses this information to **avoid routing client requests to unreachable nodes** whenever possible（尽可能地）.

The gossip process tracks state from other nodes both **directly** (nodes gossiping directly to it) and **indirectly** (nodes communicated about secondhand, third-hand, and so on). 

Cassandra采用 Phi 累计型故障检测（Phi Accrual Failure Detection）判断一个节点是否存活。传统的故障检测只由简单的“心跳”机制实现，只是根据是否收到心跳确定节点是否死亡。

但在死亡和存活之间，应该还存在一种怀疑和**可信度级别**。可信度级别表示对一个节点有故障的相信程度。例如，只是发现一个连接有问题并不一定说明整个节点死亡。

Cassandra会考虑 network performance、workload、historical conditions 几个方面的因素来计算某个节点故障的可信度级别。

Configuring the [phi_convict_threshold](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/configuration/configCassandra_yaml.html#configCassandra_yaml__phi_convict_threshold) property adjusts the sensitivity of the failure detector. Lower values **increase** the likelihood that an unresponsive node will be marked as down, while higher values **decrease** the likelihood that transient failures causing node failure. Use the default value for most situations, but increase it to 10 or 12 for Amazon EC2 (due to frequently encountered network congestion). In unstable network environments, raising the value to 10 or 12 helps **prevent false failures**.

**Values higher than 12 and lower than 5 are not recommended.**

由于节点中断很少意味着永久离开群集，所以即使断定节点发生故障，也不会自动将故障节点永久从集群中删除。其他节点会定期尝试与故障节点重新建立联系，查看其是否恢复运行。

如果想要永久从集群中移除一个节点，administrators 必须通过`nodetool utility`明确将节点移除集群。

## Snitch

snitch（告密者）的任务是提供网络拓扑的有关信息，使Cassandra能高效的路由请求。

A snitch determines which datacenters and racks nodes belong to. They inform Cassandra about the network topology so that requests are routed efficiently and allows Cassandra to distribute replicas by grouping machines into datacenters and racks.

The **replication strategy** places the replicas **based on the information provided by the snitch**.

Cassandra执行一个读操作时，必须满足一致性级别要求的副本数。为了提高读取速度，Cassandra会选择一个节点查询完整的数据，并向其它节点请求数据的Hash值来实现数据的对比。snitch在这里的作用就是帮助识别能最快返回数据的节点，从而向那个节点查询完整的数据。

模式配置下，snitch的实现是`SimpleSnitch`，它与网络拓扑无关，无法感知集群中的datacenter和rack，所以不适合在多数据中心部署时使用。

Cassandra为不同的网络拓扑和云环境提供了多个snitch，如 Amazon EC2、Google Cloud 等。

- [SimpleSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchSimple.html)

  The SimpleSnitch is used only for single-datacenter deployments.

- [Dynamic snitching](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchDynamic.html)

  Monitors the performance of reads from the various replicas and chooses the best replica based on this history.

- [RackInferringSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchRackInf.html)

  Determines the location of nodes by rack and datacenter corresponding to the IP addresses.

- [PropertyFileSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchPFSnitch.html)

  Determines the location of nodes by rack and datacenter.

- [GossipingPropertyFileSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archsnitchGossipPF.html)

  Automatically updates all nodes using gossip when adding new nodes and is **recommended for production**.

- [Ec2Snitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchEC2.html)

  Use the Ec2Snitch with Amazon EC2 in a single region.

- [Ec2MultiRegionSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchEC2MultiRegion.html)

  Use the Ec2MultiRegionSnitch for deployments on Amazon EC2 where the cluster spans multiple regions.

- [GoogleCloudSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchGoogle.html)

  Use the GoogleCloudSnitch for Cassandra deployments on Google Cloud Platform across one or more regions.

- [CloudstackSnitch](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archSnitchCloudStack.html)

  Use the CloudstackSnitch for Apache Cloudstack environments.

## Data distribution and replication

### 环和令牌

Cassandra将集群管理的数据空间抽象为一个环，集群中的每个节点会被映射到环中的某个位置，每个节点负责管理环上的一段数据区间。

Cassandra采用的就是`一致性哈希算法`，[一致性哈希算法的介绍可以参考Here](https://developer.aliyun.com/article/1082388)。

#### Consistent hashing

Consistent hashing allows distribution of data across a cluster to minimize reorganization when nodes are added or removed.

Each node in the cluster is responsible for **a range of data** based on the hash value.

![arc_hashValueRange](.\image\arc_hashValueRange.png)

Cassandra places the data on each node according to the value of the partition key and the range that the node is responsible for.

#### using virtual nodes

Prior to Cassandra 1.2, you had to calculate and assign a single token to each node in a cluster. Each token determined the node's position in the ring and its portion of data according to its hash value. 

一个token就对应环上的一段数据区间，Cassandra 1.2中的一个节点只能被分配一个token，所以一个节点只能管理环上的一段数据区间。

In Cassandra 1.2 and later, each node is allowed **many** tokens. The new paradigm is called virtual nodes (vnodes). Vnodes allow each node to own **a large number of small partition ranges** distributed throughout the cluster.

##### Ring without virtual nodes

![image-20241115223224905](.\image\image-20241115223224905.png)

Each node is assigned a single token that represents a location in the ring. 

Each node also contains copies of each row from other nodes in the cluster. For example, if the replication factor is 3, range E replicates to nodes 5, 6, and 1. 

Notice that a node owns exactly one **contiguous** partition range in the ring space.

##### Ring with virtual nodes

![image-20241115223550798](.\image\image-20241115223550798.png)

virtual nodes are **randomly** selected and **non-contiguous**.  The placement of a row is determined by the hash of the partition key within many smaller partition ranges belonging to each node.

默认配置下，每个物理节点会被分配到**256**个token，也就是256个vnode。

利用虚拟节点，可以更容易的维护一个包含异构机器的集群。对于那些计算资源多的节点，就可以多分配些vnode，对于处理能力不强的节点，就少分配些vnode。

#### Partitioners

a partitioner is a function for deriving a token representing a row from its partition key, typically by hashing.

Cassandra offers the following partitioners：

- `Murmur3Partitioner` (default): uniformly distributes data across the cluster based on **MurmurHash** hash values.
- `RandomPartitioner`: uniformly distributes data across the cluster based on **MD5** hash values.
- `ByteOrderedPartitioner`: keeps an ordered distribution of data lexically by key bytes

The `RandomPartitioner` uses a cryptographic hash that takes longer to generate than the `Murmur3Partitioner`. Cassandra doesn't really need a cryptographic hash, so using the `Murmur3Partitioner` results in a 3-5 times improvement in performance.

Partitioner使用的Hash算法不需要有加密特性。

### 复制策略

Cassandra stores replicas on multiple nodes to ensure reliability and fault tolerance. The total number of replicas across the cluster is referred to as the **replication factor**.

A replication factor of 1 means that there is only one copy of each row in the cluster. If the node containing the row goes down, the row cannot be retrieved. 

A replication factor of 2 means two copies of each row, where each copy is on a different node. 

All replicas are equally important; there is no primary or master replica.

The replication factor should not exceed the number of nodes in the cluster.

#### 在哪些节点上存放replica

第一个replica总是被放置到由 Partitioner 哈希计算后指定的那个节点上，其余的replica要根据 `replication strategy`放置。

Cassandra直接提供了两种复制策略：`SimpleStrategy`和`NetworkTopologyStrategy`。

SimpleStrategy 从 Partitioner 指定的节点开始，将每个replica放置到环中**连续**的节点上。

NetworkTopologyStrategy允许你为每个datacenter指定一个不同的副本因子。在一个datacenter中，Cassandra会适当地**将各个replica放置到不同的rack**上以获得最大的availability。对于在production环境的部署，都推荐使用NetworkTopologyStrategy，因为如果需要，使用这个策略可以更容易地增加额外的datacenter。

每个 keyspace 的复制策略是单独设置的。

##### SimpleStrategy

Use only for a single datacenter and one rack. `SimpleStrategy` places the first replica on a node determined by the partitioner. Additional replicas are placed on the next nodes clockwise in the ring without considering topology.

##### NetworkTopologyStrategy

Use `NetworkTopologyStrategy` when you have (or plan to have) your cluster deployed across multiple datacenters. This strategy specifies how many replicas you want in each datacenter.

`NetworkTopologyStrategy` places replicas in the same datacenter by walking the ring clockwise until reaching the first node in **another rack**. `NetworkTopologyStrategy` attempts to place replicas **on distinct racks** because nodes in the same rack often fail at the same time due to power, cooling, or network issues.

##### How many replicas to configure in each datacenter

When deciding how many replicas to configure in each datacenter, the two primary considerations are (1) being able to satisfy reads locally, without incurring cross data-center latency, and (2) failure scenarios.

The two most common ways to configure multiple datacenter clusters are:

- **Two replicas in each datacenter**: This configuration tolerates the failure of a single node per replication group and still allows local reads at a consistency level of `ONE`.
- **Three replicas in each datacenter**: This configuration tolerates either the failure of one node per replication group at a strong consistency level of `LOCAL_QUORUM` or multiple node failures per datacenter using consistency level `ONE`.

每个datacenter有多少个replica也可以是不固定的。 For example, you can have three replicas in one datacenter to serve real-time application requests and use a single replica elsewhere for running analytics.

## 一致性级别

一致性级别包括ONE、TWO、THREE，分别表示必须响应一个请求的副本节点的绝对数量。

一致性级别QUORUM要求大多数副本节点响应， `Q = floor (RF / 2 + 1)` RF是副本因子。

一致性级别ALL要求所有节点都响应。

在Cassandra中，一致性是可调的，客户端可以对读写操作分别指定所需的一致性级别。

**R + W > RF = 强一致性**。只要满足这个条件，所有读操作就都会看到最新写操作的结果，因此可以得到强一致性。

不要搞混副本因子和一致性级别的概念。副本因子是为每个keyspace设置的，一致性级别则是客户端为每个查询指定的。

一致性级别是基于副本因子，而不是基于集群中的节点数。

## Repairing nodes

节点在中断后重新上线时，可能会错过对其负责的副本数据的写入，这就需要有修复机制能够恢复遗漏的数据。

Over time, data in a replica can become inconsistent with other replicas due to the distributed nature of the database. Node repair corrects the inconsistencies so that eventually all nodes have the same and most up-to-date data. 

Cassandra provides the following repair processes:

- [Hinted Handoff](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/operations/opsRepairNodesHintedHandoff.html)

  If a node becomes unable to receive a particular write, the write's coordinator node preserves the data to be written as a set of *hints*. When the node comes back online, the coordinator effects repair by handing off hints so that the node can catch up with the required writes.

- [Read Repair](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/operations/opsRepairNodesReadRepair.html)

  During the read path, a query assembles data from several nodes. The coordinator node for this read compares the data from each replica node. If any replica node has outdated data, the coordinator node sends it the most recent version.

- [Anti-Entropy Repair](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/operations/opsRepairNodesManualRepair.html)

  Cassandra provides the [nodetool repair](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/tools/toolsRepair.html) tool, which you can use to repair recovering nodes.
  也叫做手动修复（**manual repair**），作为日常维护过程的一部分。它会先执行校验操作，再执行合并操作。

  Cassandra通过Merkle Tree实现校验，一个节点会与其相邻节点交换Merkle Tree，如果不同节点的Merkle Tree不匹配，就会执行修复操作。















