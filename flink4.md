## DataStream, Operator, Operator Subtask

在 Flink 中，应用程序由**DataStream**组成，这些DataStream可以通过用户定义的各种**operator**进行转换。所有DataStream构成有向无环图DAG，该DAG起始于一个或多个**source**，并终止于一个或多个**sink**。

可以将**DataStream**看作是一个不可变的数据集合(但其中可能包含重复数据)，这些集合中的元素可以是有限的，也可以是无限的。

DataStream是不可变的，这意味着它们一旦被创建，就不能向其添加或删除元素，也不能简单地查看集合内部的元素，而只能通过DataStream API来处理。

你可以通过在Flink程序中添加一个source创建一个**初始**DataStream。然后，你可以从这个初始DataStream派生出新的DataStream，并使用诸如map、filter等API将上下游的DataStream连接起来。

<img src="./image4/program_dataflow.svg" alt="img" />

Flink天生具备**并行性**和**分布式**特性。在程序执行过程中，一个DataStream会拥有一个或多个**stream partition**，这使得每个operator拥有一个或多个 **operator subtask**。这些operator subtask彼此相互独立，它们会分布在不同的线程中执行，更有可能会在不同的机器或容器中执行。

Operator subtask的数量即那个特定operator的**并行度**。在同一程序里，**不同的operator可能具有不同级别的并行度**。

<img src="./image4/parallel_dataflow.svg" alt="img" />

DataStream能够以**一对一**模式或**shuffle**模式在两个operator之间传输数据。

## Flink Architecture

`Flink runtime`由两种类型的进程组成：JobManager和一个或多个TaskManager：

<img src="./image4/processes.svg" alt="img" />



### Client

在Flink架构中，Client负责与JobManager交互并对用户代码进行**预处理**。

Client接收用户编写的Flink程序代码，对代码逻辑进行解析和处理，并将其转换为Flink内部可以识别的**JobGraph**。

例如，用户编写了一个Flink程序，从Kafka读取数据，进行一些数据处理操作(如Map、Filter等)，然后将数据写入HDFS。客户端会将这个逻辑流程整理成一个JobGraph，其中包含了各个操作的顺序和依赖关系。

一旦Client将用户代码整理成JobGraph，它会将这个JobGraph发送给JobManager，开启这个Job的执行。

### JobManager

JobManager进程中包含如下component：

- **JobMaster**

  一个JobMaster负责单个Job的管理和执行。它接收从Client发来的JobGraph，并将其转换为**ExecutionGraph**，依据集群资源情况和ExecutionGraph对任务进行调度。
  同时，JobMaster还负责**监控**任务的执行状态，处理任务执行失败时的**恢复**工作。例如，当任务执行失败时，JobMaster会根据作业的配置和执行状态，决定是否需要重新调度该任务，或者根据Checkpoint信息进行状态恢复。
  **每个Job都有属于自己单独的JobMaster**。

- **ResourceManager**

  ResourceManager负责Flink集群中的资源管理和分配。它会将Flink集群中的资源划分成一个个**任务槽(task slot)**，`task slot`是Flink集群中资源调度的基本单位。
  ResourceManager与集群中所有的TaskManager保持通信，以掌握集群中的资源状况，并为JobMaster分配所需的任务槽。
  Flink针对不同环境实现了多种类型的ResourceManager(如针对 YARN、Kubernetes、StandAlone等)。在StandAlone部署模式下，ResourceManager只能分配现有TaskManager中的任务槽，不能主动启动新的TaskManager。

- **Dispatcher**
  Dispatcher为用户提供了一个REST接口，用于提交Flink应用程序，并为每个提交的Job启动一个JobMaster。
  它还负责运行Flink的WebUI，以提供Job执行的信息。

- **Checkpoint Coordinator**

  Checkpoint Coordinator会定期向TaskManager发送Checkpoint触发信号，接收和管理来自TaskManager的Checkpoint完成报告，确保Checkpoint操作的一致性和完整性，以便在出现故障时，能够将Job恢复到之前的一致状态。

Flink集群中至少要包含一个JobManager。如果要对Flink集群采用高可用的设置，可能会有多个JobManager，其中一个是leader，其他的是备用节点(standby)。

### TaskManagers

当JobManager完成任务的调度和分配后，TaskManager会接收这些任务并执行它们。这些任务通常是对DataStream进行各种操作，如Map、Filter、Reduce等操作。
当一个任务处理完数据后，需要将处理后的数据传递给下一个任务，此时TaskManager负责在不同任务之间交换数据。数据交换可以在同一个TaskManager内的不同任务之间进行，也可以在不同的TaskManager之间进行。

在TaskManager中，`task slot`是资源调度的最小单元。TaskManager中的`task slot`数量表示该TaskManager能够并发处理的任务数量。

值得注意的是，**多个Operator可以在一个任务槽中执行**。这样做可以减少数据在不同Operator之间的传输开销，因为它们处于同一任务槽内，可以更高效地共享数据和资源，避免了跨任务槽的数据传输和网络通信带来的延迟。例如，在一个Job中，可能会在一个map operator后面紧跟着filter operator，这两个operator就可以被安排在同一个任务槽中执行，形成一个Operator Chain，从而提高处理效率。

#### Tasks and Operator Chains

For distributed execution, Flink *chains* operator subtasks together into **tasks**. **Each task is executed by one thread.**

Chaining operators together into tasks is a useful optimization: it reduces the overhead of thread-to-thread handover and buffering, and increases overall throughput while decreasing latency.

<img src="./image4/tasks_chains.svg" alt="img" />

对Flink Job中task划分的简单理解：以Shuffle操(如`keyBy`、`rebalance`、`partitionCustom` 等)作为边界将整个JobGraph划分成多个段，在一个段中的Operator都能够放入到一个Task中执行，而在不同段中的Operator必须放入到不同的Task中执行。

#### Task Slots and Resources

Each TaskManager is a *JVM process*, and may execute one or more subtasks in separate threads. To control how many tasks a TaskManager accepts, it has so called **task slots**.

Each *task slot* represents a fixed subset of resources of the TaskManager. A TaskManager with three slots will dedicate 1/3 of its managed memory to each slot. Slotting the resources means that a subtask will not compete with subtasks from other jobs for managed memory, but instead has a certain amount of reserved managed memory. Note that **no CPU isolation** happens here; currently **slots only separate the managed memory of tasks**.

在Flink中，默认情况下允许多个subtask共享任务槽，即使这些subtask对应不同的task，只要这些subtask都来自同一个job就行。

所以在极端情况下，one slot may hold an entire pipeline of the job. 

<img src="./image4/slot_sharing.svg" alt="img" />

## Flink Application Execution

A *Flink Application* is any user program that spawns one or **multiple** Flink jobs from its `main()` method. 

The jobs of a Flink Application can either be submitted to a **long-running** `Flink Session Cluster` or a `Flink Application Cluster`. 

### Flink Application Cluster

A Flink Application Cluster is a dedicated Flink cluster that only executes jobs from **one** Flink Application and where the **main() method runs on the cluster rather than the client**. The lifetime of a Flink Application Cluster is bound to the lifetime of the Flink Application.

You don’t need to start a Flink cluster first and then submit a job to the existing cluster session; instead, you package your application logic and dependencies into a executable job JAR and the cluster entrypoint is responsible for calling the `main()` method to extract the JobGraph. This allows you to deploy a Flink Application like any other application on **Kubernetes**. 

In a Flink Application Cluster, the ResourceManager and Dispatcher are scoped to a **single** Flink Application, which provides a better separation of concerns than the Flink Session Cluster.

### Flink Session Cluster

In a Flink Session Cluster, the client connects to a **pre-existing, long-running** cluster that can accept multiple job submissions. Even after all jobs are finished, the cluster will keep running until the session is manually stopped. The lifetime of a Flink Session Cluster is not bound to the lifetime of any Flink Job.

TaskManager slots are allocated by the ResourceManager on job submission and released once the job is finished. 

Because all jobs are sharing the same cluster, there is some competition for cluster resources — like network bandwidth in the submit-job phase. One limitation of this shared setup is that if one TaskManager crashes, then all jobs that have tasks running on this TaskManager will fail; in a similar way, if some fatal error occurs on the JobManager, it will affect all jobs running in the cluster.

拥有一个预先存在的Flink集群可以节省大量申请资源和启动TaskManager的时间。这对那些执行时间非常短、交互式的Flink应用非常重要，因为较高的启动时间会对端到端的用户体验产生负面影响，在这种情况下，人们期望Job能够利用现有资源迅速地进行计算。
