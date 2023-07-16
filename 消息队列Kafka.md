# 消息队列Kafka

### 简介

kafka是一个分布式、高吞吐量、高扩展性的消息队列系统。

### 相关概念

#### 消息队列的两行模式

#### Kafka的相关概念

##### 生产者

##### 主题

##### 分区

##### 副本

提高分区的可靠性，只是一个备份，副本不提供消费的能力

##### broker

##### 消费者

##### 消费者组

##### leader和follower

##### Zookeeper

2.8版本以后zookeeper为可选组件

### Kafka的部署

#### 单机版本测试环境的部署

1. 首先从官网下载对应的二进制软件包 https://kafka.apache.org/downloads

![image-20230716144821611](/Users/renchan/Desktop/Archive/笔记/assets/image-20230716144821611.png)

2. 然后将压缩包解压到服务器的对应目录
3. 进入到bin目录中 执行`sudo bash ./zookeeper-server-start.sh ../config/zookeeper.properties`
4. 如果zookeeper启动成功，重新打开一个窗口，启动kafka `sudo bash ./kafka-server-start.sh ../config/server.properties`
5. 至此，kafka已经启动成功了，创建主题的命令是 `./kafka-topics.sh --create --topic xxx --bootstrap-server localhost:9092`



### Kafka常用命令

### SpringBoot整合Kafka

#####  依赖

```xml
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
```



##### yml配置

```yml
#配置kafka 服务器
spring:
    kafka:
      bootstrap-servers: 127.0.0.1:9092
      producer:
        # 发生错误后，消息重发的次数。
        retries: 0
        #当有多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。该参数指定了一个批次可以使用的内存大小，按照字节数计算。      batch-size: 16384      # 设置生产者内存缓冲区的大小。
        buffer-memory: 33554432
        # 键的序列化方式
        key-serializer: org.apache.kafka.common.serialization.StringSerializer
        # 值的序列化方式
        value-serializer: org.apache.kafka.common.serialization.StringSerializer
        # acks=0 ： 生产者在成功写入消息之前不会等待任何来自服务器的响应。
        # acks=1 ： 只要集群的首领节点收到消息，生产者就会收到一个来自服务器成功响应。
        # acks=all ：只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。
        acks: 1
      consumer:
        # 自动提交的时间间隔 在spring boot 2.X 版本中这里采用的是值的类型为Duration 需要符合特定的格式，如1S,1M,2H,5D
        auto-commit-interval: 1S
        # 该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下该作何处理：
        # latest（默认值）在偏移量无效的情况下，消费者将从最新的记录开始读取数据（在消费者启动之后生成的记录）
        # earliest ：在偏移量无效的情况下，消费者将从起始位置读取分区的记录
        auto-offset-reset: earliest
        # 是否自动提交偏移量，默认值是true,为了避免出现重复数据和数据丢失，可以把它设置为false,然后手动提交偏移量
        enable-auto-commit: false
        # 键的反序列化方式
        key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
        # 值的反序列化方式
        value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      listener:
        # 在侦听器容器中运行的线程数。
        concurrency: 5
        #listner负责ack，每调用一次，就立即commit
        ack-mode: manual_immediate
        missing-topics-fatal: false

```

##### Topic初始化

```java
@Configuration
public class KafkaConfig {

    /**
     * 创建一个名为topic.test的Topic并设置分区数为8，分区副本数为2
     */
    @Bean    public NewTopic initialTopic() {
        return new NewTopic("topic.test", 8, (short) 2);
    }
}

```







### Kafka原理与调优