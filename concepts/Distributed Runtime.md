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

