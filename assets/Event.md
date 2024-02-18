Event:

在vector中，独立的数据单元称为Event,分为Log Event 和 Metric Event

Component：

Vector中的组件，分为sources transforms sinks

components之间组成了pipeline,数据由source收集，经过transforms处理，最后流入sink

Buffer:

当sink组件的发送速度无法匹配送入的数据，会先把数据缓存在一个可以配置的buffer中。当缓存满了之后，有两种可以配置的策略：

drop_newest: 直接丢弃新到来的event

block:(默认) 当缓存满时，产生backpressure,将会传递给pipeline中的上游的组件。 



支持的source

支持的sink

支持的transforms:

vrl(vector remap lanaguage): 通过vector自定义的表达式语法对数据进行处理

dedupe: 减少重复数据

filter: 基于condition过滤数据

lua: 基于Lua语言

route: 将一个流中的event分为多个流

reduce: 根据condition和合并策略将多个log合并为一条log



本周内容：

1.安装环境与程序 Ubuntu ...

2.测试对比logstash与vector的吞吐量

pipeline设置：

发送数据量设置：

指标的获取：

logstash通过api获取

vector通过其提供的playground，在web界面通过 [GraphQL](https://graphql.org/) API对各个component的吞吐量进行查询



实验结果：

遇到的问题：

1.udp有很多丢包，不管有多少个客户端同时给服务器发送udp报文，vector和logstash测量的每秒吞吐量都是一样的，最后接收到的数据也远远小于发送的消息数量。导致本周没有对比出两者的吞吐量差异

下周内容：

1.想办法继续对比下两者的吞吐量

2.看下目前智源需要的对日志的处理操作 vector是否全部支持

3.考虑重新安装个智源环境，再安装个vector，测试下经过vector处理的log能否被flink中的job正确处理



