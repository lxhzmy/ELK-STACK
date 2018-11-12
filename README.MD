# EFK搭建

**@Author**  :      矫宗本

**@TimeStamp**: 2018年11月12日

**@Email** ： qingdao1968@163.com

### 应用架构拓扑图:

![](D:\GITLAB\ELK-STACK\assets\efk.png)

整体架构主要分为 5 个模块，分别提供不同的功能

**Filebeat**：轻量级数据收集引擎。基于原先 Logstash-fowarder 的源码改造出来。换句话说：Filebeat就是新版的 Logstash-fowarder，也会是 ELK Stack 在 Agent 的第一选择。

**Socket**： springboot appender异步向 kafka消息队列推送数据。（推送数据第二选择）

**Kafka**: 数据缓冲队列。作为消息队列解耦了处理过程，同时提高了可扩展性。具有峰值处理能力，使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃。

**Logstash** ：数据收集处理引擎。支持动态的从各种数据源搜集数据，并对数据进行过滤、分析、丰富、统一格式等操作，然后存储以供后续使用。

**Elasticsearch** ：分布式搜索引擎。具有高可伸缩、高可靠、易管理等特点。可以用于全文检索、结构化检索和分析，并能将这三者结合起来。Elasticsearch 基于 Lucene 开发，现在使用最广的开源搜索引擎之一，Wikipedia 、StackOverflow、Github 等都基于它来构建自己的搜索引擎。

**Kibana** ：可视化平台。它能够搜索、展示存储在 Elasticsearch 中索引数据。使用它可以很方便的用图表、表格、地图展示和分析数据。

**Docker** ： 整个ELK STACK 集群都是搭建在DOCKER.上的。容器化部署方便快捷。



### 版本说明

```
System : Redhat7 
docker: 18.06-ce,build e68fc7a
Filebeat: 6.4.0
Kafka : wurstmeister/kafka:1.1.0
zookeeper:3.4
Logstash: 6.4.0
Elasticsearch: 6.4.0
Kibana: 6.4.0
```



### Install  docker 

​	reference： https://mirror.tuna.tsinghua.edu.cn/help/docker-ce/



