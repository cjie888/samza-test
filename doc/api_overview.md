#API Overview

Samza编写一个流处理器时,你必须实现StreamTask接口:

    package com.example.samza;
    
    public class MyTaskClass implements StreamTask {
    
      public void process(IncomingMessageEnvelope envelope,
                          MessageCollector collector,
                          TaskCoordinator coordinator) {
        // process message
      }
    }

当您运行作业时,Samza将为你的类创建多个实例(可能在多个机器上)。这些任务实例处理输入流消息。

在你的作业的配置可以告诉Samza要消费的流。一个不完整的例子可能会像这样(见详细的[配置文档](http://samza.incubator.apache.org/learn/documentation/0.7.0/jobs/configuration.html)):

    # This is the class above, which Samza will instantiate when the job is run
    task.class=com.example.samza.MyTaskClass
    
    # Define a system called "kafka" (you can give it any name, and you can define
    # multiple systems if you want to process messages from different sources)
    systems.kafka.samza.factory=org.apache.samza.system.kafka.KafkaSystemFactory
    
    # The job consumes a topic called "PageViewEvent" from the "kafka" system
    task.inputs=kafka.PageViewEvent
    
    # Define a serializer/deserializer called "json" which parses JSON messages
    serializers.registry.json.class=org.apache.samza.serializers.JsonSerdeFactory
    
    # Use the "json" serializer for messages in the "PageViewEvent" topic
    systems.kafka.streams.PageViewEvent.samza.msg.serde=json

每个Samza接收任务的输入流的消息,处理方法被调用。包装包含重要的三件事:消息,关键字,和消息流来源。

    /** Every message that is delivered to a StreamTask is wrapped
     * in an IncomingMessageEnvelope, which contains metadata about
     * the origin of the message. */
    public class IncomingMessageEnvelope {
      /** A deserialized message. */
      Object getMessage() { ... }
    
      /** A deserialized key. */
      Object getKey() { ... }
    
      /** The stream and partition that this message came from. */
      SystemStreamPartition getSystemStreamPartition() { ... }
    }

键和值声明为对象,需要转化为正确的类型。如果你没有配置一个序列化器/反序列化器,他们通常使用Java字节数组。一个反序列化器可以将这些字节转换成其他类型,例如上面提到的JSON序列化器解析字节数组为java.util.Map,java.util.List和字符串对象。

The getSystemStreamPartition() method returns a SystemStreamPartition object, which tells you where the message came from. It consists of three parts:
getSystemStreamPartition()方法返回一个SystemStreamPartition对象,它告诉你的信息是从哪里来的。它包括三个部分:



1. 系统:系统的名称来定义在你的配置作业的消息来源。可以有多个输入和/或输出系统,每个都有一个不同的名称。
2. 流名称:流在源系统的名称(主题、队列)。这也是在作业配置中定义的。
3. 分区:流通常分为若干个分区,每个分区由Samza分配给一个StreamTask实例。

API看起来像这样:

    /** A triple of system name, stream name and partition. */
    public class SystemStreamPartition extends SystemStream {
    
      /** The name of the system which provides this stream. It is
          defined in the Samza job's configuration. */
      public String getSystem() { ... }
    
      /** The name of the stream/topic/queue within the system. */
      public String getStream() { ... }
    
      /** The partition within the stream. */
      public Partition getPartition() { ... }
    }

在上面的示例任务配置,系统的名字是“kafka”,流的名字是“PageViewEvent”。(这个名字“kafka”不是特殊——你可以给你的系统任何你想要的名字。)如果有多个输入流给StreamTask,您可以使用SystemStreamPartition确定你收到什么样的消息。

那么发送消息呢?如果你看看StreamTask的process()方法,你会发现你得到MessageCollector。

    /** When a task wishes to send a message, it uses this interface. */
    public interface MessageCollector {
      void send(OutgoingMessageEnvelope envelope);
    }

发送一条消息,你创建一个OutgoingMessageEnvelope对象并将其传递给消息收集器。至少,信封应指定您想要发送的消息和系统和流名称发送它。你也可以选择指定分区键和其他参数。有关详细信息,请参阅javadoc。

NOTE: 请只使用MessageCollector对象在process()方法。如果你拥有MessageCollector实例并稍后再使用它,你可能不会将消息正确发送。

例如,这里有一个简单的任务,将每个输入消息分解成单词,并发出每个单词作为一个单独的消息:

    public class SplitStringIntoWords implements StreamTask {
    
      // Send outgoing messages to a stream called "words"
      // in the "kafka" system.
      private final SystemStream OUTPUT_STREAM =
        new SystemStream("kafka", "words");
    
      public void process(IncomingMessageEnvelope envelope,
                          MessageCollector collector,
                          TaskCoordinator coordinator) {
        String message = (String) envelope.getMessage();
    
        for (String word : message.split(" ")) {
          // Use the word as the key, and 1 as the value.
          // A second task can add the 1's to get the word count.
          collector.send(new OutgoingMessageEnvelope(OUTPUT_STREAM, word, 1));
        }
      }
    }