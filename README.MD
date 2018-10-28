# Elasticsearch, Logstash, Kibana,Kafka,FileBeat (ELK) Docker image



### Reference ###

This document provides a convenient centralised log server and log management web interface, by packaging [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html),  [Logstash](https://www.elastic.co/guide/en/logstash/current/index.html),  [Kibana](https://www.elastic.co/guide/en/kibana/current/index.html), [Kafka](https://kafka.apache.org/08/documentation.html), and [FileBeat](https://www.elastic.co/guide/en/beats/filebeat/current/index.html) collectively known as ELK-STACK.

The following tags are available:

- `V640`: ELK-STACK 6.4.0.

 ### Contents ###

- [Prerequisites](#prerequisites)
- [Installation](#Installation)
  - [Standalone Mode](#Standalone Mode)
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

1.1 Kafka、zookeeper

- docker-kafka.yml:（kafka、zookeeper）

  ```yaml
  version: '3.4'
  services:
    zoo001:
      image: zookeeper:3.4
      ports:
      - "2181:2181"
      restart: always
      networks:
        backend:
          aliases:
          - zoo001
  
    broker001:
      image: wurstmeister/kafka:1.1.0
      ports:
      - "9092:9092"
      environment:
      - KAFKA_HOST_NAME=broker001
      - KAFKA_ADVERTISED_HOST_NAME=broker001   
      - KAFKA_PORT=9092
      - KAFKA_ADVERTISED_PORT=9092
      - KAFKA_ZOOKEEPER_CONNECT=zoo001:2181
      - KAFKA_LISTENERS=PLAINTEXT://broker001:9092
      volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      depends_on:
      - zoo001
      restart: always
      networks:
        backend:
          aliases:
          - broker001
  
  networks:
    backend:
      external:
        name: backend
  ```

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
/opt/kafka/bin/kafka-topics.sh --create --zookeeper 10.164.201.27:2181 --replication-factor 1 --partitions 1 --topic log
 ```

- 查看刚才创建的主题

 ```
/opt/kafka/bin/kafka-topics.sh --list --zookeeper 10.164.201.27:2181
 ```

- 描述刚才创建的主题详情:

 ```
 /opt/kafka/bin/kafka-topics.sh --describe --zookeeper 10.164.201.27:2181 --topic log
 ```

- kafka 生产消息:

 ```
 /opt/kafka/bin/kafka-console-producer.sh --broker-list 10.164.201.27:9092 --topic log
 ```

- 消费消息:

 ```
 /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.164.201.27:9092 --from-beginning --topic log
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

- 修改agent机器的host,否则filebeat找不到broker001:

 ```
vi /etc/hosts
## 添加如下dns
10.164.201.27 broker001
 ```

- 启动filebeat 发送信息到Kafka：

  ```
  service filebeat start
  ```


1.2 elasticsearh 、logstash、kibana

-  **Elasticsearch**:

     - elasticsearch.yml:

  ```
  cluster.name: "es-cluster"
  network.host: 0.0.0.0
  discovery.zen.minimum_master_nodes: 1
  xpack.security.enabled: false # 不启用密码登陆
  xpack.monitoring.collection.enabled: true
  ```

- **Logstash**

  - logstash.conf

   ```
  input {
      kafka {
          bootstrap_servers => ["kafka001:9092"]
          auto_offset_reset => "latest"
          consumer_threads => 3 # 3个消费线程，默认是1个
          topics => ["log","log-collect"]
      }
  }
  filter {
    mutate{  # 切分日志信息并添加相应字段
      split => [ "message"," | " ]
  
      add_field => {
        "timestamp" => "%{[message][0]}"
      }
  
      add_field => {
        "level" => "%{[message][2]}"
      }
  
      add_field => {
        "server_name" => "%{[message][1]}"
      }
  
      add_field => {
        "ip" => "%{[message][3]}"
  
      }
  
      add_field => {
        "device" => "%{[message][4]}"
      }
  
      add_field => {
        "thread_class_method" => "%{[message][5]}"
      }
  
      add_field => {
        "content" => "%{[message][6]}"
      }
  
      remove_field => [ "message" ]
    }
  
    date {  # 将上面得到的日期信息，也就是日志打印的时间作为时间戳
      match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS" ]
      locale => "en"
      target => [ "@timestamp" ]
      timezone => "Asia/Shanghai" # 这里如果不设置时区，在Kibana中展示的时候会多了8个小时
    }
  
    geoip { # 分析ip
      source => "ip"
    }
  
    useragent { # 分析User-Agent
      source => "device"
      target => "userDevice"
      remove_field => [ "device" ]
    }
  
  }
  output {
      stdout{ codec => rubydebug } # 输出到控制台
      elasticsearch { # 输出到 Elasticsearch
          action => "index"
          hosts  => ["es001:9200"]
          index  => "logstash-%{server_name}-%{+yyyy.MM.dd}"
          document_type => "%{server_name}"
          # user => "elastic" # 如果选择开启xpack security需要输入帐号密码
          # password => "changeme"
      }
  
   ```

    -  logstash.yml

  ```
  http.host: "0.0.0.0"
  xpack.monitoring.elasticsearch.url: http://es001:9200 # Docker版的Logstash此配置的默认地址是http://elasticsearch:9200
  
  # xpack.monitoring.elasticsearch.username: "elastic" # 如果选择开启xpack security需要输入帐号密码
  # xpack.monitoring.elasticsearch.password: "changeme"
  ```

- #### Kibana

  - kibana.yml

  ```
  server.name: kibana
  server.host: "0"
  elasticsearch.url: http://es001:9200
  xpack.monitoring.ui.container.elasticsearch.enabled: true
  #elasticsearch.username: "elastic"
  #elasticsearch.password: "changeme"
  ```

- **ELK**

  - docker-elk.yml

  ```
  version: '3'
  services:
    es001:
      image: docker.elastic.co/elasticsearch/elasticsearch:6.4.0
      ports:
        - "9200:9200"
      restart: always
      environment:
        - discovery.type=single-node
        - ES_JAVA_OPTS=-Xms512m -Xmx512m
      volumes:
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      deploy:
        placement:
          constraints:
          - node.role == manager
      networks:
        backend:
          aliases:
            - es001
  
    kibana:
      image: docker.elastic.co/kibana/kibana:6.4.0
      ports:
        - "5601:5601"
      restart: always
      deploy:
        placement:
          constraints:
          - node.role == manager
      networks:
        backend:
          aliases:
            - kibana
      volumes:
      - ./config/kibana.yml:/usr/share/kibana/config/kibana.yml
      depends_on:
        - es001
  
    logstash:
      image: docker.elastic.co/logstash/logstash:6.4.0
      ports:
        - "5601:5601"
      restart: always
      environment:
        - LS_JAVA_OPTS=-Xmx512m -Xms512m
      volumes:
        - ./config/logstash.conf:/etc/logstash.conf
        - ./config/logstash.yml:/usr/share/logstash/config/logstash.yml
      deploy:
        placement:
          constraints:
          - node.role == manager
      networks:
        backend:
          aliases:
            - logstash
      depends_on:
        - elk-elasticsearch
      entrypoint:
        - logstash
        - -f
        - /etc/logstash.conf
  
  # docker network create -d=overlay --attachable backend
  # docker network create --opt encrypted -d=overlay --attachable --subnet 10.10.0.0/16 backend
  networks:
    backend:
      external:
        name: backend
  ```
