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

## Flink的基本术语

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



## Flink 核心概念

### 流处理与批处理

### 并行数据流

### 及时的流处理

Flink能给重新处理已经处理过的流数据，并且能够给出一致的处理结果。

Flink能够处理事件时间的数据，而不是基于事件的到达时间。事件事件代表了数据的产生时间，而不是它们被传输过来处理的时间。这样能够推断出一组事件何时完成。

### 有状态的流处理

Flink 的算子可以是有状态的。一个事件的处理方式可能取决于该事件之前发生的所有事件的累积影响。

Flink 应用程序在分布式集群上并行运行。算子的并行实例独立运行。 并行算子实例的state实际上是分片键值存储。每个并行实例负责处理特定组键的事件，并且这些键的状态保存在本地。 

下图显示了作业图中前三个运算符以 2 并行度运行的Task，并终止于并行度为 1 的sink算子。第三个算子是有状态的，并且第二个算子对数据进行了partition,这样做是为了按某个键对流进行分区，相同的键的数据将被同一个算子一起处理。

![](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/learn-flink/parallel-job.png)

State总是在本地被访问，这样能让Flink的应用有更高的吞吐量与更低的延时

### 通过状态快照实现容错

Flink 能够通过状态快照和流重放的组合来提供容错、一次性语义。

这些快照记录系统的整个状态，记录输入队列中的偏移量以及整个作业图的状态。

当发生故障时，数据源将回滚，状态将恢复，并继续处理。

这些状态快照是异步进行记录的，不会妨碍正在进行的处理。



## Data Stream API 

### Stream execution environment 

每个 Flink 应用程序都需要一个执行环境。

流式处理应用程序需要使用 StreamExecutionEnvironment。 

在应用程序中进行的 DataStream API 调用会生成附加到 StreamExecutionEnvironment 的作业图。

当调用 env.execute（） 时，Job被发送到 JobManager，后者并行化作业并将其切片分发给任务管理器执行。作业的每个并行切片都将在slot中执行。

 如果不调用 execute（），则不会运行应用程序。

![](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/distributed-runtime.svg)

### Stream Source

Stream的数据必须是可序列化的

Flink支持序列化Java Pojo以及Java tuples

#### 简单读取数据

```java
DataStream<String> lines = env.socketTextStream("localhost", 9999);
DataStream<String> lines = env.readTextFile("file:///path");
DataStream<Person> flintstones = env.fromCollection(people);
```



### Stream Sink

简单的sink

```java
adults.print()
```

结果解释：1> and 2> indicate which sub-task (i.e., thread) produced the output.



### Data Pipeline & ETL

Apache Flink 的一个常见用途是实现 ETL（提取、转换、加载）管道，这些管道从一个或多个源获取数据，执行一些转换或扩充，然后将结果存储在某个地方。下面将介绍一些常用的API



#### 无状态的数据转换

##### map()

将输入类型转换为输出类型

```java
DataStream<TaxiRide> rides = env.addSource(new TaxiRideSource(...));

DataStream<EnrichedRide> enrichedNYCRides = rides
    .filter(new RideCleansingSolution.NYCFilter())
    .map(new Enrichment());

enrichedNYCRides.print();
```

```java
public static class Enrichment implements MapFunction<TaxiRide, EnrichedRide> {

    @Override
    public EnrichedRide map(TaxiRide taxiRide) throws Exception {
        return new EnrichedRide(taxiRide);
    }
}
```

##### flatMap()

map用于1对1的转换，如果想要1对N的转换（一个输入对应0到多个输出）使用flatMap

```java
public static class NYCEnrichment implements FlatMapFunction<TaxiRide, EnrichedRide> {

    @Override
    public void flatMap(TaxiRide taxiRide, Collector<EnrichedRide> out) throws Exception {
        FilterFunction<TaxiRide> valid = new RideCleansing.NYCFilter();
        if (valid.filter(taxiRide)) {
            out.collect(new EnrichedRide(taxiRide));
        }
    }
}
```

```java
DataStream<TaxiRide> rides = env.addSource(new TaxiRideSource(...));

DataStream<EnrichedRide> enrichedNYCRides = rides
    .flatMap(new NYCEnrichment());

enrichedNYCRides.print();
```

#### Keyed Streams

##### keyBy()

能够按照数据的一个属性对流进行分区通常非常有用，用于将具有相同属性值的所有事件组合在一起。

```java
rides
    .flatMap(new NYCEnrichment())
    .keyBy(enrichedRide -> enrichedRide.startCell);
```

因为keyBy会导致Flink集群的网络通信，并且需要序列化与反序列化，因此是一个开销较大的算子



keyBy能够通过计算，来进行分区

key必须以确定性的方式生成，因为在需要时会重新计算密钥，而不是将其附加到流记录。

```java
keyBy(ride -> GeoUtils.mapToGridCell(ride.startLon, ride.startLat));
```

##### Aggregations on Keyed Streams 

我们可以在keyed stream上进行聚合操作

```java
minutesByStartCell
  .keyBy(value -> value.f0) // .keyBy(value -> value.startCell)
    // maxBy也可以使用field作为参数 代表POJO中的field 并且支持嵌套
  .maxBy(1) // duration
  .print();
```

The output stream now contains a record for each key every time the duration reaches a new maximum – as shown here with cell 50797:

```
...
4> (64549,5M)
4> (46298,18M)
1> (51549,14M)
1> (53043,13M)
1> (56031,22M)
1> (50797,6M)
...
1> (50797,8M)
...
1> (50797,11M)
...
1> (50797,12M)
```

上例包含了一个隐式的状态（max） Flink需要给每个不同的key都跟踪最大值状态的变化

使用流时，通常更有意义的是考虑有限窗口上的聚合，而不是整个流。



#### Stateful Transformations with Keyd State

有时候，我们对数据的转换处理需要考虑之前出现的数据的一些状态，比如，去重，我们只需要收集一段时间内没有出现过的key的事件,这时候就需要一些key的记录。Flink为我们做到了这一点。

上文介绍了一些单个数据转换的接口（Map flatMap ...） 接下来介绍的接口是**Rich Function**,比如 `RichFlatMapFunction接口，它有一些额外的方法：

- `open(Configuration c)`
- `close()`
- `getRuntimeContext()`

`open()` is called once, during operator initialization. This is an opportunity to load some static data, or to open a connection to an external service, for example.

`getRuntimeContext()` 包含了关于算子的信息 比如并行度、state等等



##### 使用举例

```java
    env.addSource(new EventSource())
        .keyBy(e -> e.key)
        .flatMap(new Deduplicator())
        .print();
```

```java
public static class Deduplicator extends RichFlatMapFunction<Event, Event> {
    //用于记录对应的key的state
    ValueState<Boolean> keyHasBeenSeen;

    @Override
    public void open(Configuration conf) {
        ValueStateDescriptor<Boolean> desc = new ValueStateDescriptor<>("keyHasBeenSeen", Types.BOOLEAN);
        keyHasBeenSeen = getRuntimeContext().getState(desc);
    }

    @Override
    public void flatMap(Event event, Collector<Event> out) throws Exception {
        if (keyHasBeenSeen.value() == null) {
            out.collect(event);
            keyHasBeenSeen.update(true);
        }
    }
}
```

很多时候，我们要控制key的state的数量，因为在一个无限流中，key可能是无限多的，可以调用

```java
keyHasBeenSeen.clear();
```

清理对应key 的state



##### Non-keyd State

在一个非keyd的context中管理状态也是可能的，但是一般不会用在UDF(user defined function)中.这个特性一般用在source 和 sink中



#### Connected Streams

一个算子可以拥有两个输入流

![](https://nightlies.apache.org/flink/flink-docs-release-1.18/fig/connected-streams.svg)

**用途**：合并两个流、根据信息如阈值、规则等来动态改变transform的行为



##### 使用举例

有一个流streamOfWords,包含了无限的单词输入，还有另一个流control，包含了需要过滤的单词，我们希望让streamOfWrods经过算子的输出，不包含需要过滤的单词。

```java
public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    DataStream<String> control = env
        .fromElements("DROP", "IGNORE")
        .keyBy(x -> x);

    DataStream<String> streamOfWords = env
        .fromElements("Apache", "DROP", "Flink", "IGNORE")
        .keyBy(x -> x);
  
    control
        .connect(streamOfWords) //连接操作
        .flatMap(new ControlFunction()) // rich transform
        .print();

    env.execute();
}
```

`ControlFunction`实现了`RichCoFlatMapFunction` 三个泛型的含义是**IN1 IN2 OUT** 即输入1（control）输入2（streamOfWord）以及 输出的类型

```java
public static class ControlFunction extends RichCoFlatMapFunction<String, String, String> {
    private ValueState<Boolean> blocked;
      
    @Override
    public void open(Configuration config) {
        blocked = getRuntimeContext()
            .getState(new ValueStateDescriptor<>("blocked", Boolean.class));
    }
      
    @Override
    public void flatMap1(String control_value, Collector<String> out) throws Exception {
        blocked.update(Boolean.TRUE);
    }
      
    @Override
    public void flatMap2(String data_value, Collector<String> out) throws Exception {
        if (blocked.value() == null) {
            out.collect(data_value);
        }
    }
}
```



**注意**：无法控制两个流中元素的消费顺序。这两个流中的Event相互竞争，Flink将会根据需要从两个流中读取数据。

当时间/顺序很重要时，有必要缓存事件在state中，以在后续一个时机处理。

通过使用实现**InputSelectable**的自定义运算符，可以对两输入运算符消耗其输入的顺序施加一些有限的控制



### Stream Analytics

