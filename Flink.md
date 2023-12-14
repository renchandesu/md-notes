# Flink

## Quick Start

### 下载地址

https://www.apache.org/dyn/closer.lua/flink/flink-1.18.0/flink-1.18.0-bin-scala_2.12.tgz

### 运行程序

![image-20231210220139891](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20231210220139891.png)



### 提交一个Flink Job

官方提供了一些example的jar包

```bash
./bin/flink run examples/streaming/WordCount.jar
```

### 查看dashboard

Flink提供了一个web页面查看job

[localhost:8081](http://localhost:8081)

### 停止程序

```bash
/bin/stop-cluster.sh
```



### 简单的小程序

- 首先，执行mvn命令，可以获取到一个demo小程序（用于检测银行卡诈骗行为）

  (注意：如果是windows环境的话，使用git bash)

```mvn
mvn archetype:generate -DarchetypeGroupId=org.apache.flink -DarchetypeArtifactId=flink-walkthrough-datastream-java -DarchetypeVersion=1.18.0 -DgroupId=frauddetection -DartifactId=frauddetection -Dversion=0.1 -Dpackage=spendreport -DinteractiveMode=false
```

#### v0版本

```java
/**
 * Skeleton code for implementing a fraud detector.
 * 这个类实现了KeyedProcessFunction 三个泛型分别是key input output
 * processElement方法可以处理输入 collector用于收集输出
 */
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private static final double SMALL_AMOUNT = 1.00;
    private static final double LARGE_AMOUNT = 500.00;
    private static final long ONE_MINUTE = 60 * 1000;

    @Override
    public void processElement(
          Transaction transaction,
          Context context,
          Collector<Alert> collector) throws Exception {

       Alert alert = new Alert();
       alert.setId(transaction.getAccountId());

       collector.collect(alert);
    }
}
```

```java
public class FraudDetectionJob {
    public static void main(String[] args) throws Exception {
       // 设置 StreamExecutionEnvironment。 StreamExecutionEnvironment是设置作业属性、创建源以及最终触发作业执行的方式。
       StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

       // 源从外部系统（如 Apache Kafka、Rabbit MQ 或 Apache Pulsar）引入数据到 Flink 作业中。
       // demo使用一个源，该源生成无限流的信用卡交易记录供您处理。
       // 每笔交易都包含账户ID（accountId）、交易发生时间的时间戳（timestamp）和美元金额（amount）。
       // name仅用于调试目的，如果出现问题，我们将知道错误的来源。
       DataStream<Transaction> transactions = env
          .addSource(new TransactionSource())
          .name("transactions");

       // partition and process
       // 交易流包含来自大量用户的大量交易，因此需要由多个欺诈检测任务并行处理。
       // 我们使用Flink的keyBy()方法将交易流分区，以便每个分区都由单独的任务处理。
       DataStream<Alert> alerts = transactions
          .keyBy(Transaction::getAccountId)
          .process(new FraudDetector())
          .name("fraud-detector");

       // 接收器将 DataStream 写入外部系统;例如 Apache Kafka。
       // AlertSink 使用日志级别 INFO 记录每个警报记录，而不是将其写入持久性存储，因此您可以轻松查看结果。
       alerts
          .addSink(new AlertSink())
          .name("send-alerts");

       // 执行作业。
       env.execute("Fraud Detection");
    }
}
```

#### v1 版本

上个版本的demo会在每个transcation到来时都发出告警 

我们希望其符合规则：当出现一个小数额交易后立即出现一个大数额交易

需要处理数据时记住状态，这就用到了state

Flink给我们提供了可容错的状态

```java
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private static final double SMALL_AMOUNT = 1.00;
    private static final double LARGE_AMOUNT = 500.00;
    private static final long ONE_MINUTE = 60 * 1000;

    private ValueState<Boolean> state;

    @Override
    public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
       // ValueState需要初始化 需要在onOpen中初始化
       // ValueState只能在keyed context 中使用
       state = getRuntimeContext().getState(new ValueStateDescriptor<>("state", Boolean.class));
    }

    @Override
    public void processElement(
          Transaction transaction,
          Context context,
          Collector<Alert> collector) throws Exception {

       Boolean value = state.value();
       if (value == Boolean.TRUE){
          if (transaction.getAmount() > LARGE_AMOUNT){
             Alert alert = new Alert();
             alert.setId(transaction.getAccountId());
             collector.collect(alert);
          }
          // 清空状态
          state.clear();
       }
       if (transaction.getAmount() < SMALL_AMOUNT){
          state.update(Boolean.TRUE);
       }

    }
}
```

#### v2版本

我们希望在上个版本基础上，在一个时间限制，检测时间要在一分钟内

要求：

1. 当检测到一个可疑值，启动一个定时器，1min后执行，清空状态
2. 当出现了一个alert后，应该停止当前的定时器，并清空其他状态

```java
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private static final double SMALL_AMOUNT = 1.00;
    private static final double LARGE_AMOUNT = 500.00;
    private static final long ONE_MINUTE = 60 * 1000;

    private ValueState<Boolean> state;

    private ValueState<Long> lastTime;

    @Override
    public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
       // ValueState需要初始化 需要在onOpen中初始化
       // ValueState只能在keyed context 中使用
       state = getRuntimeContext().getState(new ValueStateDescriptor<>("state", Boolean.class));
       lastTime = getRuntimeContext().getState(new ValueStateDescriptor<>("lastTime", Long.class));
    }

    @Override
    public void processElement(
          Transaction transaction,
          Context context,
          Collector<Alert> collector) throws Exception {

       Boolean value = state.value();
       if (value != null){
          if (transaction.getAmount() > LARGE_AMOUNT){
             Alert alert = new Alert();
             alert.setId(transaction.getAccountId());
             collector.collect(alert);
          }
          // 清空状态 清空定时器
          context.timerService().deleteProcessingTimeTimer(lastTime.value());
          state.clear();
          lastTime.clear();

       }
       if (transaction.getAmount() < SMALL_AMOUNT){
          // 设置状态和注册定时器
          state.update(Boolean.TRUE);
          Long timer = context.timerService().currentProcessingTime() + ONE_MINUTE;
          lastTime.update(timer);
          context.timerService().registerProcessingTimeTimer(timer);
       }

    }

    // 当注册的定时器到达指定的处理时间时触发
    @Override
    public void onTimer(long timestamp, KeyedProcessFunction<Long, Transaction, Alert>.OnTimerContext ctx, Collector<Alert> out) throws Exception {
       lastTime.clear();
       state.clear();
    }
}
```

### Flink的官方demos

https://github.com/apache/flink-playgrounds

### Flink的基本术语

#### checkpoint storage

checkpoint时的状态快照保存在哪里（JobManager的JVM 或者 文件系统）

#### Flink Application

是一个Java程序，用于把一个或者多个Job提交到Flink执行。提交操作通过调用execute()方法来完成

#### Flink Application Cluster

一个特定的Flink集群，只执行由Flink Application 提交的Job 它的生命周期与Flink Application联系在一起

#### Flink Job

Flink Job代表了一个逻辑图（数据流），这个逻辑图被应用程序创建并提交给Flink

#### Flink JobManager

是Flink集群的编排器。

由Flink Resource Manager, Flink Dispatcher以及每个Job都存在一个的Flink JobMaster组成

#### JobResultStore

把终止、取消、失败的Job的结果持久化到文件系统。这些结果之后将被Flink用来决定jobs是否需要被恢复。

#### Logical Graph(dataflow)

一个逻辑图是一个有向图，其中node代表Flink的算子，而边则定义了算子之间的输入输出关系，并且对应了数据流或者数据集。

#### Operator

Flink的算子，即逻辑图的Node 一个算子进行一种特定的操作，Sources和Sinks是特殊的算子

#### Operator Chain

一个算子链包含至少两个连续的算子，并且两个算子间是没有分区操作的。算子链中的算子之间的数据传递不需要经过Flink网络的序列化

#### Partition

一个Partition是一个数据流（数据集）中的一个独立的子集

一个数据流会被分成若干的Partiton，在将每个record分配到一个或多个分区之后

分区后的数据流被Tasks消费

**repartition** 这个操作改变了数据流的分区方式

#### Physical Graph

物理图是将逻辑图翻译为用于执行Job的实际运行时的结果。node是Tasks 边代表了输入输出关系或者分区关系

#### Record

记录是数据集或数据流的构成元素。运算符和函数接收Record作为输入，并将Record作为输出发出。

#### Execution Mode

分为批处理与流处理

#### Flink Session Cluster

是一个长久执行的Flink集群，可以接受多个Flink Job 它的生命周期与Flink Job无关

#### State Backend

对于流处理程序，一个Job的State backend决定了如何存储job的状态在每个TaskManager上

#### Task

 物理图的节点。任务是基本的工作单元，由 Flink 的运行时执行。任务只封装了 Operator 或 Operator Chain 的一个并行实例。

#### Sub-Task

Sub-Task是负责处理数据流分区的任务。Sub-Task强调同一算子或算子链存在多个并行任务。

#### Flink TaskManager

TaskManager 是 Flink 集群的工作进程。任务被调度到 TaskManagers 执行。它们相互通信以在后续任务之间交换数据。

#### Transformation

转换应用于一个或多个数据流或数据集，并生成一个或多个输出数据流或数据集。转换可能会基于每条记录更改数据流或数据集，但也可能仅更改其分区或执行聚合。虽然 Operator 和 Functions 是 Flink API 的“物理”部分，但 Transformations 只是一个 API 概念。具体来说，大多数转换都是由某些 Operator 实现的。

#### Table Program

使用 Flink 的关系 API（表 API 或 SQL）声明的管道的通用术语。

#### UID

算子的唯一标识符，由用户提供或根据作业结构确定。提交申请时，将转换为 UID hash。

#### UID hash

Operator 在运行时的唯一标识符