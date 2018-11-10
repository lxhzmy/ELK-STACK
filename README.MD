# Elasticsearch, Logstash, Kibana,Kafka,FileBeat (ELK) Docker image



### Reference ###

This document provides a convenient centralised log server and log management web interface, by packaging [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html),  [Logstash](https://www.elastic.co/guide/en/logstash/current/index.html),  [Kibana](https://www.elastic.co/guide/en/kibana/current/index.html), [Kafka](https://kafka.apache.org/08/documentation.html), and [FileBeat](https://www.elastic.co/guide/en/beats/filebeat/current/index.html) collectively known as ELK-STACK.

The following tags are available:

- `V640`: ELK-STACK 6.4.0.

 ### Contents ###

- [Prerequisites](#prerequisites)
- [Installation](#Installation)
  - [Standalone Mode](#Standalone Mode)
  - [Replicated Mode](#Replicated Mode)
- [Configuration](#Configuration)
- [Forwarding logs](#Forwarding logs)
- [Building the image](#Building the image)
- [Tweaking the image](#Tweaking the image)
- [Persisting log data](#Persisting log data)
- [Snapshot and restore](#Snapshot and restore)
- [Setting up an Elasticsearch cluster](#Setting up an Elasticsearch cluster)
- [Security considerations](#Security considerations)
- [Frequently encountered issues](#Frequently encountered issues)
- [Known issues](#Known issues)
- [Troubleshooting](#Troubleshooting)
- [Reporting issues](#Reporting issues)
- [References](#References)
- [About](#About)

## [Prerequisites]<a name="Prerequisites"></a>

## [Installation]<a name="Installation"></a>

- **Install Docker**

- ```
  yum install docker docker-compose	
  ```

  **Pull Images**

 ```
  docker pull zookeeper:3.4
  docker pull wurstmeister/kafka:1.1.0
  docker pull docker.elastic.co/elasticsearch/elasticsearch:6.4.0
  docker pull docker.elastic.co/kibana/kibana:6.4.0
  docker pull docker.elastic.co/logstash/logstash:6.4.0
 ```

## [Configuration]<a name="Configuration"></a>

#### [Standalone Mode]<a name="Standalone Mode"></a>

1.1    Kafka、zookeeper



- 展示docker现有容器:

 ```
docker ps 
 ```

- 进入kafka容器

 ```
docker exec -it containerId/Name bash
 ```

- 创建一个主题

 ```
/opt/kafka/bin/kafka-topics.sh --create --zookeeper zoo001:2181,zoo002:2181,zoo003,2181 --replication-factor 3 --partitions 3 --topic lla
 ```

- 查看刚才创建的主题

 ```
/opt/kafka/bin/kafka-topics.sh --list --zookeeper zoo001:2181,zoo002:2181,zoo003:2181
 ```

- 描述刚才创建的主题详情:

 ```
 /opt/kafka/bin/kafka-topics.sh --describe --zookeeper zoo001:2181,zoo002:2181,zoo003:2181 --topic logtest
 ```

- kafka 生产消息:

 ```
 /opt/kafka/bin/kafka-console-producer.sh --broker-list broker002:9092,broker001:9092,broker003:9092 --topic coka
 ```

- 消费消息:

 ```
 /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server broker002:9092,broker001:9092,broker003:9092 --from-beginning --topic ddd
 ```

- 更改agent所在机器的 /etc/filebeat/fileBeat.yml:

 ```yaml

filebeat.prospectors:
- paths:
    - /home/tomcat/Outreachplatform/Outreachplatform/logback/**/*.info.log
  tags:
    - outreach
    - info
    - product
  input_type: log
  document_type: outreach_info
  fields_under_root: true


output.kafka:
  hosts: ["broker001:9092"]
  topic: 'log'
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  max_message_bytes: 1000000

 ```
