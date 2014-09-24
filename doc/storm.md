#Storm

人们通常想知道类似的系统比较。我们已经尽力对比与Samza相当的其他系统的特性集。但我们不是专家在这些框架方面,当然我们完全有偏见。如果我们有搞错,请让我们知道,我们会改正它。

Storm和Samza相当类似。两个系统都提供了许多相同的高级特性:分区流模型,分布式执行环境,流处理的API,容错,Kafka的集成等。

Storm和Samza使用不同的词来描述相似的概念:Storm中的Spout与Samza流消费者相似,bolt类似任务,元组类似于Samza消息。Storm也有一些额外的构建块在Samza没有直接的对应。

##Ordering and Guarantees

Storm允许您选择保证你想要处理的消息的级别:

- 最简单的方式是至多一次处理、发出的消息如果不能被正确处理,或如果机器处理失败。这种模式不需要特殊的逻辑,处理消息的顺序与它们产生的Spout一致。

- 还有至少一次处理,通过保持内存中的所有元组发出的记录，追踪是否每个输入元组(和任何下游生成的元组)在配置超时内成功处理。任何元组在超时时间内没有完全处理，则Spout重新发射。这意味着一个螺bolt可能看到相同的元组不止一次,消息处理将不能保证顺序。这种机制也需要一些用户代码操作,为了正确地确认它的输入必须保持记录的祖先。这在[Storm的wiki](https://github.com/nathanmarz/storm/wiki/Guaranteeing-message-processing)中做了深度的解释。

- 最后,Storm用Trident抽象来保证有且处理一次语义。这个模式使用相同的故障检测机制像至少一次模式。元组实际上是加工至少一次,但Storm的状态实现允许重复检测和忽略。(重复检测只适用于状态管理的Storm。如果您的代码有其他副作用,例如发送消息到服务之外的拓扑,它不会只有一次语义。)在这种模式下,Spout将输入流分成批次,按严格的顺序处理批次。

Samza也提供处理保证——目前只有至少一次发送,但只有一次语义在计划中。在每个流分区,Samza总是按在分区中出现的顺序处理消息,但没有保证不同输入流或分区之间的孙逊。这个模型允许Samza提供至少一次语义没有祖先跟踪的开销。Samza,不会有性能优势使用至多一次交付失败(即删除消息),这就是为什么我们不提供模式——消息传递总是得到保证。

此外,由于Samza从未处理一个信息无序分区,它更适合处理按关键字的数据。举个例子,如果你有一个数据库更新流,稍后更新可能取代更早更新,然后重新排序消息可能改变最终的结果。提供的所有更新相同的关键字出现在相同的流分区,Samza能够保证一致的状态。

##State Management

Storm的低级bolt API不提供任何在一个流的过程的状态管理。一个bolt可以维护状态在内存(如果bolt死亡将丢失),也可以调用远程数据库读写状态。然而,拓扑通常可以处理消息以更高的速度比调用一个远程数据库,因此远程调用每个消息迅速成为一个瓶颈。

作为高级TridentAPI的一部分,Storm提供自动状态管理。它在内存中保持状态,周期性的检查持久到远程数据库(例如Cassandra),因此远程数据库调用的成本跟数处理的元组数据一样。通过维护元数据和状态,Trient可以实现只有一次处理语义——例如,如果你是计数的事件,这种机制允许计数器是正确的,即使机器失败和元组重播。

Storm的缓存和批处理状态改变工作良好，如果在每个bolt的状态量相当小——也许小于100 kb。使其适应跟踪计数器,最小值,最大值和平均值的一个指标。然而,如果你需要维护大量的状态,这种方法本质上下降由于处理元组进行数据库调用,与相关的性能成本。

Samza状态管理采用一个完全不同的方法。它不是使用远程数据库持久存储,每个Samza任务包括嵌入式键值存储,位于同一台机器上。这种存储的读和写非常快,即使存储的内容大于可用内存。这个键值存储更改复制到集群中的其他机器,这样,如果一台机器挂了,任务的状态是在另一台机器上运行可以恢复。

通过共存的存储和处理在同一台机器上,Samza能够达到非常高的吞吐量,即使有大量的状态。这是必要的,如果你想执行状态操作,不只是计数器。例如,如果您想要执行一个窗口连接多个流,用一个数据库表或者加入一个流(复制到Samza通过“更改日志”),或一组几个相关消息到一个更大的消息,那么您需要维护这么多状态,它是更有效保持状态在本地任务。

Samza状态处理的一个局限是,目前不支持只有一次语义——现在只支持至少一次。但是我们正在努力解决,所以请继续关注更新。

##Partitioning and Parallelism

Storm’s parallelism model is fairly similar to Samza’s. Both frameworks split processing into independent tasks that can run in parallel. Resource allocation is independent of the number of tasks: a small job can keep all tasks in a single process on a single machine; a large job can spread the tasks over many processes on many machines.

The biggest difference is that Storm uses one thread per task by default, whereas Samza uses single-threaded processes (containers). A Samza container may contain multiple tasks, but there is only one thread that invokes each of the tasks in turn. This means each container is mapped to exactly one CPU core, which makes the resource model much simpler and reduces interference from other tasks running on the same machine. Storm’s multithreaded model has the advantage of taking better advantage of excess capacity on an idle machine, at the cost of a less predictable resource model.

Storm supports dynamic rebalancing, which means adding more threads or processes to a topology without restarting the topology or cluster. This is a convenient feature, especially during development. We haven’t added this to Samza: philosophically we feel that this kind of change should go through a normal configuration management process (i.e. version control, notification, etc.) as it impacts production performance. In other words, the code and configuration of the jobs should fully recreate the state of the cluster.

When using a transactional spout with Trident (a requirement for achieving exactly-once semantics), parallelism is potentially reduced. Trident relies on a global ordering in its input streams — that is, ordering across all partitions of a stream, not just within one partion. This means that the topology’s input stream has to go through a single spout instance, effectively ignoring the partitioning of the input stream. This spout may become a bottleneck on high-volume streams. In Samza, all stream processing is parallel — there are no such choke points.

##Deployment & Execution

A Storm cluster is composed of a set of nodes running a Supervisor daemon. The supervisor daemons talk to a single master node running a daemon called Nimbus. The Nimbus daemon is responsible for assigning work and managing resources in the cluster. See Storm’s Tutorial page for details. This is quite similar to YARN; though YARN is a bit more fully featured and intended to be multi-framework, Nimbus is better integrated with Storm.

Yahoo! has also released Storm-YARN. As described in this Yahoo! blog post, Storm-YARN is a wrapper that starts a single Storm cluster (complete with Nimbus, and Supervisors) inside a YARN grid.

There are a lot of similarities between Storm’s Nimbus and YARN’s ResourceManager, as well as between Storm’s Supervisor and YARN’s Node Managers. Rather than writing our own resource management framework, or running a second one inside of YARN, we decided that Samza should use YARN directly, as a first-class citizen in the YARN ecosystem. YARN is stable, well adopted, fully-featured, and inter-operable with Hadoop. It also provides a bunch of nice features like security (user authentication), cgroup process isolation, etc.

The YARN support in Samza is pluggable, so you can swap it for a different execution framework if you wish.

##Language Support

Storm是用Java和Clojure编写的,但有很好的支持non-JVM语言支持。它遵循一个类似于MapReduce流的模式:non-JVM任务在一个单独的进程启动,数据发送给它的stdin,读取其stdout输出。

Samza是用Java和Scala编写的。它是建立多语言支持,但目前只支持JVM语言。

##Workflow

Storm provides modeling of topologies (a processing graph of multiple stages) in code. Trident provides a further higher-level API on top of this, including familiar relational-like operators such as filters, grouping, aggregation and joins. This means the entire topology is wired up in one place, which has the advantage that it is documented in code, but has the disadvantage that the entire topology needs to be developed and deployed as a whole.

In Samza, each job is an independent entity. You can define multiple jobs in a single codebase, or you can have separate teams working on different jobs using different codebases. Each job is deployed, started and stopped independently. Jobs communicate only through named streams, and you can add jobs to the system without affecting any other jobs. This makes Samza well suited for handling the data flow in a large company.

Samza’s approach can be emulated in Storm by connecting two separate topologies via a broker, such as Kafka. However, Storm’s implementation of exactly-once semantics only works within a single topology.

##Maturity

我们不便评论Storm的成熟度,但它有一个众多的的用户数量,一个强大的特性集,似乎正在积极开发之中。它集成了许多常见的消息传递系统(RabbitMQ,Kestrel,Kafka等)。

Samza很不成熟,但它是建立在坚实的组件之上。YARN是相当新的,但是已经被雅虎运行在3000 +节点集群上!,该项目目前正在积极被Hortonworks和Cloudera开发。Kafka有很强一面,最近看到使用增加。这也是Storm大量用的原因。Samza是一个全新的项目,LinkedIn正在使用它。我们希望别人会觉得它有用,并采取它。

##Buffering & Latency

Storm uses ZeroMQ for non-durable communication between bolts, which enables extremely low latency transmission of tuples. Samza does not have an equivalent mechanism, and always writes task output to a stream.

On the flip side, when a bolt is trying to send messages using ZeroMQ, and the consumer can’t read them fast enough, the ZeroMQ buffer in the producer’s process begins to fill up with messages. If this buffer grows too much, the topology’s processing timeout may be reached, which causes messages to be re-emitted at the spout and makes the problem worse by adding even more messages to the buffer. In order to prevent such overflow, you can configure a maximum number of messages that can be in flight in the topology at any one time; when that threshold is reached, the spout blocks until some of the messages in flight are fully processed. This mechanism allows back pressure, but requires topology.max.spout.pending to be carefully configured. If a single bolt in a topology starts running slow, the processing in the entire topology grinds to a halt.

A lack of a broker between bolts also adds complexity when trying to deal with fault tolerance and messaging semantics. Storm has a clever mechanism for detecting tuples that failed to be processed, but Samza doesn’t need such a mechanism because every input and output stream is fault-tolerant and replicated.

Samza takes a different approach to buffering. We buffer to disk at every hop between a StreamTask. This decision, and its trade-offs, are described in detail on the Comparison Introduction page. This design decision makes durability guarantees easy, and has the advantage of allowing the buffer to absorb a large backlog of messages if a job has fallen behind in its processing. However, it comes at the price of slightly higher latency.

As described in the workflow section above, Samza’s approach can be emulated in Storm, but comes with a loss in functionality.

##Isolation

Storm提供标准的UNIX进程级别的隔离。拓扑结构可能影响另一个拓扑的性能(或者相反)如果占用过多的CPU、磁盘、网络、内存等。

Samza依靠YARN提供资源级别隔离。目前,YARN提供了显式控制内存和CPU限制(通过cgroups),并且两者都与Samza成功使用。没有隔离磁盘或网络提供的YARN。

##Distributed RPC

在Storm中,您可以编写拓扑不仅接受固定事件流,但也允许客户按需运行分布式计算。查询作为一个元组发送到拓扑的一个特殊的Spout,然后拓扑计算答案,然后返回给客户端(谁是同步等待答案)。这称为“分布式RPC(DRPC)。

Samza目前没有一个DRPC等价的API,但是您可以使用Samza流处理原语构建它。

##Data Model

Storm模型所有元组的消息定义一个数据模型,使用可插拔的序列化。

Samza的序列化和数据模型都是可插拔的。我们不是非常固执己见的说哪种方法是最好的。