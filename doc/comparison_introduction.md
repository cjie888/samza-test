#对比说明

这里有一些高层的设计决策,是我们认为Samza不同于其他流处理项目的特点。

##流模型

Samza作业主要是流的输入和输出。Samza有很强的流模型,这不仅仅是一个简单的消息交换机制。 Samza流是一个分区,可复制,没分区有序的，多订阅者,无丢失的序列信息。流不仅仅是系统的输入和输出,而且是互相隔离处理过程的缓冲区。

这个强大的模型需要持久性、容错和流缓冲实现,但它有几个好处。

首先,下游处理阶段的延迟不能阻止一个上游的处理。Samza工作可以停止消费流几分钟,甚至几小时(也许是因为糟糕的部署,或长时间运行计算)但对上游的处理没有任何影响。这使得Samza适合处理所有数据流在一个大公司的大型部署:在不同的代码库,由不同的团队使用不同SLA，写时隔离是至关重要的。

这是出于我们的在Hadoop经验总结的离线处理管道。在Hadoop MapReduce作业处理阶段,处理阶段的输出是一个HDFS目录的文件。输入到下一个处理阶段只是前期产生的文件。我们发现这种强烈的隔离阶段可以由不同的团队数以百计的松散耦合的工作,组成一个离线处理生态系统。我们的目标是复制这种丰富的生态系统在近实时环境。

这个强大的模型的第二个好处是,所有的阶段是多订阅者。这意味着在实践中,如果一个人增加了一套产生输出数据流处理流,别人就能看到输出并使用它,并在其上进行构建,没有引入任何之间耦合的代码工作。快乐的副作用,这使得调试流容易,因为您可以手动检查任何阶段的输出。

最后,这个强大的流模型大大简化了Samza框架中功能的实现。每个作业只需要关心自己的输入和输出,在错误的情况下,每一个作业都可以独立地恢复和重新启动。不需要中央控制整个数据流图。

我们需要做出的权衡这个强流模型是消息写入磁盘。我们愿意让这种权衡因为MapReduce和HDFS表明持久存储可以提供非常高的读写吞吐量,而且几乎无限的磁盘空间。这个观察是Kafka的基础,它允许数百MB /秒的吞吐量,复制和许多每节点的TB磁盘空间。以这种方式使用时,磁盘的吞吐量通常不是瓶颈。

MapReduce有时被批评为写入磁盘是没有必要的。然而,这种批评不适用于流处理:批处理像MapReduce经常用于处理大型历史数据的集合在一个短的时间内(如查询一个月的数据在10分钟),而流处理大多需要跟上稳态的数据流(处理10分钟的数据在10分钟内)。这意味着原始流处理吞吐量要求,一般来说,数量级要低于批处理。

##状态

只有非常简单的流处理的问题是无状态的(即一次可以处理一条消息,独立于所有其他消息)。许多流处理应用程序需要一个作业来维护一些状态。例如:
- 如果你想知道有多少事件对于特定用户ID,你需要保持一个计数器为每个用户ID。
- 如果你想知道每天有多少不同的用户访问你的网站,你需要保持一组你今天已经产生至少有一个事件的所有用户id。
- 如果你想加入两个流(例如,如果你想要确定广告的点击率,加入广告印象事件流的广告点击事件)您需要存储事件从一个流,直到你收到相应的事件从另一个流。
- 如果你想增加事件和一些信息从数据库(例如,扩展一个页面浏览事件和一些信息的用户浏览页面),这项工作需要访问数据库的当前状态。

Some kinds of state, such as counters, could be kept in-memory in the tasks, but then that state would be lost if the job is restarted. Alternatively, you can keep the state in a remote database, but performance can become unacceptable if you need to perform a database query for every message you process. Kafka can easily handle 100k-500k messages/sec per node (depending on message size), but throughput for queries against a remote key-value store tend to be closer to 1-5k requests per second — two orders of magnitude slower.

In Samza, we have put particular effort into supporting high-performance, reliable state. The key is to keep state local to each node (so that queries don’t need to go over the network), and to make it robust to machine failures by replicating state changes to another stream.

This approach is especially interesting when combined with database change capture. Take the example above, where you have a stream of page-view events including the ID of the user who viewed the page, and you want to augment the events with more information about that user. At first glance, it looks as though you have no choice but to query the user database to look up every user ID you see (perhaps with some caching). With Samza, we can do better.

Change capture means that every time some data changes in your database, you get an event telling you what changed. If you have that stream of change events, going all the way back to when the database was created, you can reconstruct the entire contents of the database by replaying the stream. That changelog stream can also be used as input to a Samza job.

Now you can write a Samza job that takes both the page-view event and the changelog as inputs. You make sure that they are partitioned on the same key (e.g. user ID). Every time a changelog event comes in, you write the updated user information to the task’s local storage. Every time a page-view event comes in, you read the current information about that user from local storage. That way, you can keep all the state local to a task, and never need to query a remote database.

Stateful Processing

In effect, you now have a replica of the main database, broken into small partitions that are on the same machines as the Samza tasks. Database writes still need to go to the main database, but when you need to read from the database in order to process a message from the input stream, you can just consult the task’s local state.

This approach is not only much faster than querying a remote database, it is also much better for operations. If you are processing a high-volume stream with Samza, and making a remote query for every message, you can easily overwhelm the database with requests and affect other services using the same database. By contrast, when a task uses local state, it is isolated from everything else, so it cannot accidentally bring down other services.

Partitioned local state is not always appropriate, and not required — nothing in Samza prevents calls to external databases. If you cannot produce a feed of changes from your database, or you need to rely on logic that exists only in a remote service, then it may be more convenient to call a remote service from your Samza job. But if you want to use local state, it works out of the box.

##Execution Framework

One final decision we made was to not build a custom distributed execution system in Samza. Instead, execution is pluggable, and currently completely handled by YARN. This has two benefits.

The first benefit is practical: there is another team of smart people working on the execution framework. YARN is developing at a rapid pace, and already supports a rich set of features around resource quotas and security. This allows you to control what portion of the cluster is allocated to which users and groups, and also control the resource utilization on individual nodes (CPU, memory, etc) via cgroups. YARN is run at massive scale to support Hadoop and will likely become an ubiquitous layer. Since Samza runs entirely through YARN, there are no separate daemons or masters to run beyond the YARN cluster itself. In other words, if you already have Kafka and YARN, you don’t need to install anything in order to run Samza jobs.

Secondly, our integration with YARN is completely componentized. It exists in a separate package, and the main Samza framework does not depend on it at build time. This means that YARN can be replaced with other virtualization frameworks — in particular, we are interested in adding direct AWS integration. Many companies run in AWS which is itself a virtualization framework, which for Samza’s purposes is equivalent to YARN: it allows you to create and destroy virtual “container” machines and guarantees fixed resources for these containers. Since stream processing jobs are long-running, it is a bit silly to run a YARN cluster inside AWS and then schedule individual jobs within this cluster. Instead, a more sensible approach would be to directly allocate a set of EC2 instances for your jobs.

We think there will be a lot of innovation both in open source virtualization frameworks like Mesos and YARN and in commercial cloud providers like Amazon, so it makes sense to integrate with them.