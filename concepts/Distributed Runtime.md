### Flink运行环境
#### Tasks和Operator执行链
在分布式的执行环境下，Flink会将一组subtasks的operator串接为对应的执行链（chain），并在分配在同一个task中去运行。每个task会有独立的线程去执行。把operator组成执行链，并由对应的task去执行是一个非常有效的优化策略：
* 降低线程间的交互代价和数据缓存空间
* 增加整体的吞吐量
* 降低端到端的延迟

对于执行链的配置，可以查阅对于的文档。
对于下图示例的数据执行流图，其有5个subtask，因此其有5个并行的执行线程。

![Task Chain](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/tasks_chains.svg)

#### Job Managers, Task Managers和Clients
Flink运行一个集群，包括两类的进程角色JobManager和TaskManager。
JobManager也可以称为Master节点，其主要功能包括：
* 协调整个全局的分布式执行环境
* Job的Task的调度
* Job的Checkpoint和失败恢复等

对于整个集群来说，至少需要一个JobManager节点。如果设置了HA模式，则可以有多个JobManager节点的存在，但是只有一个是Leader，其他的则是standby状态。

TaskManager节点也可以称为worker节点，负责一个或者多个Job的task的执行；同时也负责Job的数据流在多个task或者TaskManager之间的缓存和交互。必须存在至少一个TaskManager节点。

对于JobManager和TaskManager的运行模式可以有多种方式：
* 直接运行在物理机/虚拟机上的独立集群模式（standalone cluster）
* 运行在容器中
* 依赖YARN或者Mesos等资源调度框架进行调度

TaskMananger一旦启动就会连接自身的JobManager，告知JobManager自身的资源情况和状态，然后等待分配任务。
Client并不属于运行集群组件和任务执行的一部分，其主要负责Job的一些初始化工作。如JobGraph的生成，任务代码的提交等。一旦Client完成相应的工作， 就可以断开和JobManager的连接；当然也可以保持连接，以接收任务的进度状态等信息。Client可以在Java、Scala等业务代码中运行，也可以通过命令行$FLINK_HOME/bin/flink run 来执行。

![Flink Arch](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/processes.svg)

#### Task Slots和 Resources
一个TaskManager是一个JVM进程，其可以执行一个或者多个子task，每个子task由独立的线程来执行。对于一个TaskManager可以运行多少子task，则由其slot的数量确定的。一个TaskManager至少一个slot，具体多少个，由TaskManager启动的时候指定。
对于一个task slot来说，会有一组相应的TaskManager资源绑定在上面。举例来说，一个TaskManager有3个slot，那么每个slot就拥有TaskManager的1/3的内存资源。根据slot来分配资源的意义在于，可以根据slot来做资源的隔离，不同的Job的task之间资源抢占。目前而言，slot只支持task的内存资源分配和隔离，而CPU的隔离还不支持。
通过设置每个TaskManager的slot数量，可以确定子task彼此之间隔离的层度。对于一个TaskManager一个slot的场景，则意味着每个task group都是运行在独立的JVM上的（可以由不同的container来启动）。一个TaskManager拥有多个slot，则子task可以共享同一个JVM的资源，包括TCP连接和同JobManager的心跳信息等。也可以共享一些数据集和数据结构，降低每个task的额外开销。

![Task Slots](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/tasks_slots.svg)

默认情况下，Flink允许同一个Job的不同task的一组子task共享同一个slot。这样的可能会使得一个slot可以运行Job的整个pipeline。slot的共享有如下的两大好处：
一个Flink集群至少需要的slot数量是Job的最高并发度。通过slot共享，就不需要计算task的总量了，特别是不同的task的并发度还不一样。
A Flink cluster needs exactly as many task slots as the highest parallelism used in the job. No need to calculate how many tasks (with varying parallelism) a program contains in total.slot共享可以提供更好的资源利用率：
* 如果不共享slot，一些不是资源消耗高的操作，如source/map()的子task 则有可能会被window操作的资源消耗型的子task所阻碍。
* 如果不共享slot，在下图的例子中，如果将基本的并发度从2增加到6，那么就会消费掉集群的所有slot资源；而且不能够保证资源消耗高的子task合理分配到了所有的TaskManager上了。

It is easier to get better resource utilization. Without slot sharing, the non-intensive source/map() subtasks would block as many resources as the resource intensive window subtasks. With slot sharing, increasing the base parallelism in our example from two to six yields full utilization of the slotted resources, while making sure that the heavy subtasks are fairly distributed among the TaskManagers.

![Slot Share](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/slot_sharing.svg)

我们也可以根据API来调整不同的slot共享的分组策略，也可以来禁止slot的共享。
一个基本的原则是，默认的slot数量应该跟CPU的核心数量一致。如果启用了超线程的话，那么每个slot用有2个物理线程的CPU资源。

#### State Backends

Job的key/values等状态数据的存储依赖设置的状态存储后端引擎(state backend)。存储引擎包括内存型的hashmap、基于RocksDB的key/values存储等。对于存储状态的数据结构来说，后端存储引擎需要实现并支持某个时间点的key/value的状态快照，并将快照保存为checkpoint。

![checkpoint](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/checkpoints.svg)

#### Savepoints
基于Data Stream API的开发Job可以通过savepoint进行恢复和重新拉起。通过savepoint可以在你更新Flink集群配置或者重新发布应用的时候，保证不丢失任何应用的状态数据。

Savepoint可以通过人工的触发checkpoint来实现，将状态的快照写入到外部的存储引擎中，同样也可以通过定时的触发checkpoint来完成。当运行的应用运行的worker节点（TaskManager）定期的完成快照后，一旦需要恢复应用则只需要最后一次的完整快照，之前的的快照数据都可以被安全的丢弃掉。


Savepoint和快照没有本质区别，只是触发的方式不一样。Savapiont通常由用户人工触发，而checkpoint是由Job自动触发（启用了checkpoint的情况下）。并且Savepoint不会自动删除之前完成的快照数据。Savepoint可以通过命令行触发，可以调用REST API来取消某个job的savepoint操作。
