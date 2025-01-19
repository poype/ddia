## Stream, State and Time

对于流处理应用框架（Streaming Application Framework）而言，对数据流（Stream）、状态（State）、时间（Time）这三个方面的把控能力至关重要。下面我们来深入了解 Flink 框架在这三个方面的支持情况。

### Streams

Flink 能够同时处理有界流（bounded Stream）与无界流（unbounded Stream）。这意味着它既可以实现实时（real - time）数据的计算，也能处理记录（recorded,  persisting the stream to a storage system and process it later）数据的计算。

<img src="./image4/bounded-unbounded.png" alt="img" style="zoom:40%;" />

### State

在 Flink 中，状态（State）处于核心地位。应用程序借助状态来记录历史Event或中间处理结果，并在后续数据处理过程中，读取之前存储在State中的记录。

Flink 中的状态具备以下特点：

- **超大状态（Very Large State）**：Flink 能够维护规模达数 TB 的应用程序状态。
- **可扩展应用（Scalable Applications）**：Flink 可以将状态重新分配到更多或更少的Worker节点，以此支持有状态应用（Stateful Application）的扩展。
- **精确一次的状态一致性（Exactly - once state consistency）**：Flink 的检查点（checkpoint）和恢复算法确保在发生故障时，应用程序状态依然保持一致性。
- **可插拔状态后端（Pluggable State Backends）**：Flink 支持多种不同的状态后端（State Backend），允许将状态存储在内存或 RocksDB 中。
- **多种状态原语（Multiple State Primitives）**：Flink 针对不同的数据结构，提供了多种状态原语，例如原子值（atomic values）、列表（lists）或映射（maps） 。

### 时间（Time）

众多常见的流计算任务都基于时间展开，windows aggregations、sessionization 以及 time - based joins等。

Flink 提供了丰富且强大的时间相关功能：

- **事件时间模式（Event - time Mode）**：采用事件时间语义处理流的应用程序，依据事件的发生时间来计算结果。所以，无论处理的是记录事件（recorded event）还是实时事件（real - time event），运用事件时间模式都能确保结果的准确性与一致性。
- **水位线支持（Watermark Support）**：Flink 通过水位线（wartermark）判断事件发生时间在特定范围内的事件是否均已到达。水位线是一种极为灵活的机制，能够在处理结果的完整性和时效性之间实现权衡。
- **迟到数据处理（Late Data Handling）**：在以事件时间作为时间基础处理流时，若借助Wartermark判断事件是否全部到达，可能会出现结果计算在所有相关事件到达前就被认定为完成的情况。这类事件被称为**迟到事件（late event）**。Flink 提供了多种处理迟到事件的选项，比如通过侧输出（side outputs）重新路由它们，并更新先前已完成的结果。
- **处理时间模式（Processing - time Mode）**：除了事件时间模式，Flink 还支持处理时间语义，即依据处理机的wall-clock时间触发计算。处理时间模式适用于那些对低延迟有严格要求，且能够容忍近似结果的应用程序。

## 分层式 API

Flink 提供了三层 API，每一层 API 都在简洁性与表达力之间做了不同的权衡。

正如下面这张图所展示的，越靠近底层的 API，其表达能力越是丰富，然而相应地，使用起来也更为复杂。而越靠近顶层的 API，虽然使用起来更为简洁，但在表达力上有所欠缺。

<img src="./image4/api-stack.png" alt="img" style="zoom:40%;" />

## 高可用与Checkpoint

若要让你的 Flink 应用程序实现 7×24 小时不间断运行，**不仅要确保在发生故障后应用程序能够被自动重启，还需确保其内部状态维持一致（注意Flink的应用是有状态的，这与无状态的Pod有鲜明的区别）**，以便应用程序能继续处理数据，仿佛故障从未发生。

Flink 提供了多项功能，用以确保应用程序持续运行且保持一致性：

- **一致性检查点（Consistent Checkpoints）**：Flink 的恢复机制基于应用程序状态（Application state）的一致性检查点（consistent checkpoints）。一旦出现故障，应用程序会被重启，并从最新的检查点（Checkpoints）加载其状态。结合可重置的流数据源（stream sources），此功能能够保证实现 **“exactly-once state consistency”**。
- **高效的检查点（Efficient Checkpoints）**：如果应用程序维护着数 TB 的状态（State），对其执行checkpoint操作的开销可能相当高昂。Flink 能够执行异步和增量式的checkpoint操作，从而最大程度减少检查点对应用程序性能的影响。
- **端到端精确一次（End-to-End Exactly-Once）**：Flink 为特定的存储系统提供了事务性接收器（transactional sink），确保即便发生故障，数据也仅会被精确写入一次。
- **与集群管理器集成（Integration with Cluster Managers）**：Flink 与诸如 Hadoop YARN 或 Kubernetes 等集群管理器紧密集成。当某个进程发生故障时，会自动启动一个新进程来接管其工作。
- **高可用设置（High-Availability Setup）**：Flink 具备高可用模式，可消除所有单点故障。该高可用模式依托久经考验的可靠分布式协调服务 Apache ZooKeeper 实现 。

## Savepoint

在软件领域，任何应用程序都需要进行更新与维护，这可能是为了实现功能升级、改进，或者进行 Bug 修复等。一般来说，不能简单地停止应用程序，然后直接重新部署修复版或改进版，因为丢失应用程序状态往往会带来难以承受的代价。

Flink 的 Savepoint 就是为了解决有状态应用程序更新这一难题而设计的。Savepoint 是应用程序状态的一致性快照，它**与 Checkpoint 极为相似**。不过，和 Checkpoint 不同的是，Savepoint 需要手动触发，而且在应用程序停止时不会自动删除。Savepoint 可用于启动一个与它的状态兼容的应用程序（即对新启动的应用进行初始化）。

Savepoint 支持以下功能：

- **Application Evolution**: Savepoints can be used to evolve applications. A fixed or improved version of an application can be restarted from a savepoint that was taken from a previous version of the application. It is also possible to start the application from an earlier point in time (given such a savepoint exists) to repair incorrect results produced by the flawed version.
- **Cluster Migration**: Using savepoints, applications can be migrated (or cloned) to different clusters.
- **Flink Version Updates**: An application can be migrated to run on a new Flink version using a savepoint.
- **Application Scaling**: Savepoints can be used to increase or decrease the parallelism of an application.
- **A/B Tests and What-If Scenarios**: The performance or quality of two (or more) different versions of an application can be compared by starting all versions from the same savepoint.
- **Pause and Resume**: An application can be paused by taking a savepoint and stopping it. At any later point in time, the application can be resumed from the savepoint.
- **Archiving**: Savepoints can be archived to be able to reset the state of an application to an earlier point in time.

## Deploy Applications Anywhere

Apache Flink 是一个分布式系统，其执行应用程序依赖于计算资源的支持。Flink 能够与所有常见的集群资源管理器（如 Hadoop YARN 和 Kubernetes ）进行集成，也可设置为以**stand-alone**的模式运行。

Flink 在设计上可以与上述每一种资源管理器实现良好的协同运作。这是通过针对特定资源管理器的部署模式来达成的，这些模式使得 Flink 能够以契合各资源管理器习惯的方式与其进行交互。

When deploying a Flink application, Flink automatically identifies the required resources based on the application’s configured parallelism and requests them from the resource manager. In case of a failure, Flink replaces the failed container by requesting new resources. All communication to submit or control an application happens via REST calls. This eases the integration of Flink in many environments.

## Leverage In-Memory Performance

Stateful Flink applications are optimized for **local state access**. Task state is always maintained in memory or, if the state size exceeds the available memory, in access-efficient on-disk data structures. Hence, tasks perform all computations by accessing local, often in-memory, state yielding very low processing latencies. Flink guarantees exactly-once state consistency in case of failures by periodically and asynchronously checkpointing the local state to durable storage.

<img src="./image4/local-state.png" alt="img" style="zoom:35%;" />

下图展示了传统应用架构与事件驱动型应用之间的差异：

<img src="./image4/usecases-eventdrivenapps.png" alt="img" style="zoom:35%;" />

Instead of querying a remote database, event-driven applications access their data locally which yields better performance, both in terms of throughput and latency. The periodic checkpoints to a remote persistent storage can be asynchronous and incremental. Hence, the impact of checkpointing on the regular event processing is very small.

## Data Analytics Applications

<img src="./image4/usecases-analytics.png" alt="img" style="zoom:40%;" />

The results are either written to an external database or maintained as internal state. A dashboard application can read the latest results from the external database or directly query the internal state of the application.

Dashboard可以直接查询Flink的 internal state 获取数据分析结果。

