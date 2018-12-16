# EFK搭建

**@Author**  :      矫宗本

**@TimeStamp**: 2018年11月12日

**@Email** ： qingdao1968@163.com

### 应用架构拓扑图:

![](.\assets\efk.png)

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

### 具体实践

#### Docker 

​	reference： https://mirror.tuna.tsinghua.edu.cn/help/docker-ce/



#### Filebeat

为什么用 Filebeat ，而不用原来的 Logstash 呢？

原因很简单，资源消耗比较大。

由于 Logstash 是跑在 JVM 上面，资源消耗比较大，后来作者用 GO 写了一个功能较少但是资源消耗也小的轻量级的 Agent 叫 Logstash-forwarder。

后来作者加入 elastic.co 公司， Logstash-forwarder 的开发工作给公司内部 GO 团队来搞，最后命名为 Filebeat。

Filebeat 需要部署在每台应用服务器上，可以通过 Salt 来推送并安装配置。

```
cd /etc/yum.repo.d/
```

新建 elasticsearch.repo,输入以下内容 :wq! 保存

```
[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

```
yum install -y filebeat
```

至此 filebeat 安装完成.

修改filebeat默认配置文件

vi /etc/filebeat/filebeat.yml

```
filebeat.prospectors:
- paths:
    - /data/logs/loanplatform/**/*.log
  input_type: log
  enabled: true
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after
  fields:
      service: loanplatform
- paths:
    - /data/logs/outreach*/**/*.log
  input_type: log
  enabled: false
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after
  fields:
      service: outreach

output.kafka:
  hosts: ["10.164.201.27:9091","10.164.201.27:9092","10.164.201.27:9093"]
  topic: '%{[fields][service]}'
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  max_message_bytes: 1000000

#output.console:
#  pretty: true
```

以上的配置文件作用:

- 读取两个地方的log 并且multiline多行首行不是%{yyyy-MM-dd}格式的数据 合并为一个事件。也就是一个message
- 发送日志到kafka集群 "10.164.201.27:9091","10.164.201.27:9092","10.164.201.27:9093" ;并且指定topic : loanplatform、outreach

命令行启动启动测试:

```
/usr/share/filebeat/bin/filebeat -e -c /etc/filebeat/filebeat.yml
```

测试成功后: service 启动

```
service filebeat start
```

**注： 请查询service filebeat 设置开机自启动**



#### Kafla  cluster

##### 单机版配置

kafka-standalone.yml

```
version: '3.4'
services:
  zoo001:
    image: zookeeper:3.4
    ports:
    - "2181:2181"
    restart: always
    environment:
        ZOO_MY_ID: 1
        ZOO_SERVERS: server.1=zoo001:2888:3888
    networks:
      backend:
        aliases:
        - zoo001

  broker001:
    image: wurstmeister/kafka:1.1.0 
    ports:
    - "9091:9092"
    restart: always
    depends_on:
      - zoo001
    environment:
    - KAFKA_BROKER_ID=1        
    - KAFKA_ZOOKEEPER_CONNECT=zoo001:2181
    - KAFKA_LISTENERS=PLAINTEXT://broker001:9092
    - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.164.204.153:9091
    - KAFKA_AUTO_CREATE_TOPICS='enable'
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    networks:
      backend:
        aliases:
        - broker001
  kafka-manager:
    image: sheepkiller/kafka-manager:1.3.0.4
    ports:
    - "9000:9000"
    restart: always
    environment:
    - ZK_HOSTS=zoo001:2181
    networks:
      backend:
        aliases:
        - kafka-manager      
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  portainer:
    image: portainer/portainer
    ports:
      - "9001:9000"
    command: -H unix:///var/run/docker.sock
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:

networks:
  backend:
    external:
      name: backend
```

容器启动:

```
docker-compose -f  kafka-standalone.yml up 
```

作用: 

​	zookeeper 开放内外端口映射2181

​	kafka 容器内端口9092，外网端口9091.

​	KAFKA_LISTENERS 是kafka配置的内网hosts

​	KAFKA_ADVERTISED_LISTENERS 是kafka配置的外网访问ip:port 



##### 集群版配置

因为kafka docker构建的 ，所以 这里直接上docker 配置文件:

kafka-cluster.yml

```
version: '3.4'
services:
  zoo001:
    image: zookeeper:3.4
    ports:
    - "2181:2181"
    restart: always
    environment:
        ZOO_MY_ID: 1
        ZOO_SERVERS: server.1=zoo001:2888:3888 server.2=zoo002:2888:3888 server.3=zoo003:2888:3888 server.4=zoo004:2888:3888:observer    
    networks:
      backend:
        aliases:
        - zoo001
  zoo002:
    image: zookeeper:3.4
    ports:
    - "2182:2181"
    restart: always
    environment:
        ZOO_MY_ID: 2
        ZOO_SERVERS: server.1=zoo001:2888:3888 server.2=zoo002:2888:3888 server.3=zoo003:2888:3888 server.4=zoo004:2888:3888:observer    
    networks:
      backend:
        aliases:
        - zoo002
  zoo003:
    image: zookeeper:3.4
    ports:
    - "2183:2181"
    restart: always
    environment:
        ZOO_MY_ID: 3
        ZOO_SERVERS: server.1=zoo001:2888:3888 server.2=zoo002:2888:3888 server.3=zoo003:2888:3888 server.4=zoo004:2888:3888:observer    
    networks:
      backend:
        aliases:
        - zoo003
  zoo004:
    image: zookeeper:3.4
    ports:
    - "2184:2181"
    restart: always
    environment:
        ZOO_MY_ID: 4
        PEER_TYPE: observer            
        ZOO_SERVERS: server.1=zoo001:2888:3888 server.2=zoo002:2888:3888 server.3=zoo003:2888:3888 server.4=zoo004:2888:3888:observer    
    networks:
      backend:
        aliases:
        - zoo004        
  broker001:
    image: wurstmeister/kafka:1.1.0 
    ports:
    - "9091:9092"
    restart: always
    depends_on:
      - zoo001
      - zoo002
      - zoo003
      - zoo004           
    environment:
    - KAFKA_BROKER_ID=1        
    - KAFKA_ZOOKEEPER_CONNECT=zoo001:2181,zoo002:2181,zoo003:2181,zoo004:2181
    - KAFKA_LISTENERS=PLAINTEXT://broker001:9092
    - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.164.201.27:9091
    - KAFKA_AUTO_CREATE_TOPICS='enable'
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    networks:
      backend:
        aliases:
        - broker001
  broker002:
    image: wurstmeister/kafka:1.1.0 
    ports:
    - "9092:9092"
    restart: always
    depends_on:
      - zoo001
      - zoo002
      - zoo003
      - zoo004           
    environment:
    - KAFKA_BROKER_ID=2        
    - KAFKA_ZOOKEEPER_CONNECT=zoo001:2181,zoo002:2181,zoo003:2181,zoo004:2181
    - KAFKA_LISTENERS=PLAINTEXT://broker002:9092
    - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.164.201.27:9092
    - KAFKA_AUTO_CREATE_TOPICS='enable'  
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    networks:
      backend:
        aliases:
        - broker002
  broker003:
    image: wurstmeister/kafka:1.1.0 
    ports:
    - "9093:9092"
    restart: always
    depends_on:
      - zoo001
      - zoo002
      - zoo003
      - zoo004           
    environment:
    - KAFKA_BROKER_ID=3      
    - KAFKA_ZOOKEEPER_CONNECT=zoo001:2181,zoo002:2181,zoo003:2181,zoo004:2181
    - KAFKA_LISTENERS=PLAINTEXT://broker003:9092
    - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.164.201.27:9093
    - KAFKA_AUTO_CREATE_TOPICS='enable'    
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    networks:
      backend:
        aliases:
        - broker003               

  kafka-manager:
    image: sheepkiller/kafka-manager:1.3.0.4
    ports:
    - "9000:9000"
    restart: always
    environment:
    - ZK_HOSTS=zoo001:2181
    networks:
      backend:
        aliases:
        - kafka-manager      
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock     

  portainer:
    image: portainer/portainer
    ports:
      - "9001:9000"
    command: -H unix:///var/run/docker.sock
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
          
networks:
  backend:
    external:
      name: backend
```

启动命令:

```
docker-compose -f kafka-cluster.yml up 
```

![](.\assets\topics.png)

如果filebeat推送数据消费不了，请对话题添加分区增加并发消费能力 大于当前分区数即可：

![1542035848797](C:\Users\JamesBond\AppData\Roaming\Typora\typora-user-images\1542035848797.png)

作用: 

​	zookeeper集群内网访问 zoo001:2181,zoo002:2181,zoo003:2181,zoo004:2181

​	kafka 集群对外访问：10.164.201.27:9091,10.164.201.27:9092,10.164.201.27:9093

​	KAFKA_LISTENERS 是kafka配置的内网hosts

​	KAFKA_ADVERTISED_LISTENERS 是kafka配置的外网访问ip:port 

#### ELK  

也是用docker搭建的，这里直接上配置文件: 

```
version: '3'
services:
  es001:
    image: elasticsearch:6.4.0
    ports:
      - "9200:9200"
    restart: always
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1024m -Xmx1024m
    volumes:
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /data/esdata001:/usr/share/elasticsearch/data
    networks:
      backend:
        aliases:
          - es001

  kibana:
    image: kibana:6.4.0
    ports:
      - "5601:5601"
    restart: always
    networks:
      backend:
        aliases:
          - kibana
    volumes:
      - ./config/kibana.yml:/usr/share/kibana/config/kibana.yml
    depends_on:
      - es001

  logstash:
    image: logstash:6.4.0
    ports:
      - 9600:9600
    restart: always
    environment:
      - LS_JAVA_OPTS=-Xmx512m -Xms512m
    volumes:
      - ./config/logstash.conf:/etc/logstash.conf
      - ./config/logstash.yml:/usr/share/logstash/config/logstash.yml
    networks:
      backend:
        aliases:
          - logstash
    depends_on:
      - es001
    entrypoint:
      - logstash
      - -f
      - /etc/logstash.conf
      - --config.reload.automatic
# docker network create -d=overlay --attachable backend
# docker network create --opt encrypted -d=overlay --attachable --subnet 10.10.0.0/16 backend


networks:
  backend:
    external:
      name: backend

```

作用: 

搜索引擎es : 单节点 ， 1024M的堆内存



#### Config

Logstash 提供三大功能

- INPUT 进入

- FILTER 过滤功能

- OUTPUT 出去



如果使用 Filter 功能的话，强烈推荐大家使用 [Grok debugger](https://grokdebug.herokuapp.com/) 或者kibana dev tools来预先解析日志格式。

logstash.conf

```

input {
    kafka {
        bootstrap_servers => "10.164.201.27:9091,10.164.201.27:9092,10.164.201.27:9093" # KAFKA集群
        auto_offset_reset => "latest" # 获取最新记录
        consumer_threads => 2 # 3个消费线程，默认是1个
        topics => ["log"]   #解析socket 的日志
        codec => multiline {
          pattern => "^\s"
          what => "previous"
        }
    }
}
  
input {
    kafka {
        bootstrap_servers => "10.164.201.27:9091,10.164.201.27:9092,10.164.201.27:9093" # KAFKA集群
        auto_offset_reset => "latest" # 获取最新记录
        consumer_threads => 2 # 3个消费线程，默认是1个
        topics => ["loanplatform","outreach"]
        codec => json #解析filebeat 的日志
    }
}

filter {

      # 分别对各个平台的报文进行解析
    if([fields][service] == 'loanplatform'){


        grok {
            match => { "message" => "\A%{TIMESTAMP_ISO8601:timestamp}\s*%{DATA:app}\s\[%{WORD:TraceId}?\,%{WORD:SpanId}?\,%{WORD:ParentSpanId}?\,%{WORD:spanExport}?\]\s*\[(?<thread_name>.+?)\]?\s*%{LOGLEVEL:loglevel}\s*%{JAVACLASS:class}\s*%{NOTSPACE:codeLine}\s*%{NOTSPACE:loanThread}\s*%{GREEDYDATA:msg}\s*" }
        }
    }else if ([fields][service] == 'outreach'){

        grok {
            match => { "message" => "\A%{TIMESTAMP_ISO8601:timestamp}\s*%{DATA:app}\s\[%{WORD:TraceId}?\,%{WORD:SpanId}?\,%{WORD:ParentSpanId}?\,%{WORD:spanExport}?\]\s*\[(?<thread_name>.+?)\]?\s*%{LOGLEVEL:loglevel}\s*%{JAVACLASS:class}\s%{GREEDYDATA:msg}\s*" }
        }

    }else {
        grok {
            match => { "message" => "\A%{TIMESTAMP_ISO8601:timestamp}\s*%{DATA:app}\s\[%{WORD:TraceId}?\,%{WORD:SpanId}?\,%{WORD:ParentSpanId}?\,%{WORD:spanExport}?\]\s*\[(?<thread_name>.+?)\]?\s*%{LOGLEVEL:loglevel}\s*%{JAVACLASS:class}\s%{GREEDYDATA:msg}\s*" }
        }
    }
      # UTC时区转换-8小时。并且剔除字段
    date {
        match=> ["timestamp","YYYY-MM-dd HH:mm:ss.SSS"]
        target=> "@timestamp"
        locale => "en"
        timezone => "+08:00"
        remove_field =>[ "timestamp","message","@version","host","beat","offset"]

    }                

}
output {
    stdout{ codec => rubydebug } # 输出到控制台
    elasticsearch { # 输出到 Elasticsearch
        action => "index"
        hosts  => ["es001:9200"]
        index  => "logstash-%{app}-%{+yyyy.MM.dd}"
        document_type => "%{app}"
        # user => "elastic" # 如果选择开启xpack security需要输入帐号密码
        # password => "changeme"
    }
}   


```

ELK具体页面展示:

![](.\assets\kibana.png)

Docker 监控页面：

![](.\assets\docker-monitor-ui.png)



至此，项目搭建完毕。感谢大家耐心看完。  

**如果还有不明白的，咨询我 邮件，我有时间会耐心回复的。**

以上，谢谢！

--------------------------------------------------------华丽的分割线-----------------------------------------------------------











### 常用测试命令

#### Kafka-test

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
/opt/kafka/bin/kafka-topics.sh --create --zookeeper zoo001:2181,zoo002:2181,zoo003,2181 --replication-factor 3 --partitions 3 --topic ms
 ```

- 查看刚才创建的主题

 ```
/opt/kafka/bin/kafka-topics.sh --list --zookeeper zoo001:2181,zoo002:2181,zoo003:2181
 ```

- 描述刚才创建的主题详情:

 ```
 /opt/kafka/bin/kafka-topics.sh --describe --zookeeper zoo001:2181,zoo002:2181,zoo003:2181 --topic lla
 ```

- kafka 生产消息:

 ```
 /opt/kafka/bin/kafka-console-producer.sh --broker-list broker002:9092,broker001:9092,broker003:9092 --topic lla
 ```

- 消费消息:

 ```
 /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server broker002:9092,broker001:9092,broker003:9092 --from-beginning --topic lla
 ```



#### 一键停用所有docker容器:

查看运行容器

```
docker ps
```

查看所有容器

```
docker ps -a
```

进入容器

其中字符串为容器ID:

```
docker exec -it d27bd3008ad9 /bin/bash1
```

停用全部运行中的容器:

```
docker stop $(docker ps -q)
```

删除全部容器：

```
docker rm $(docker ps -aq)
```

一条命令实现停用并删除容器：

```
docker stop $(docker ps -q) & docker rm $(docker ps -aq)

```

 

### 剩下要做的事情：

- ES指定多个data目录进行存储
- 汉化
- 冷热存储X-PACK功能实现
- 搭建教程完善
- 文件备份导出方式，当文件两条大的时候怎么按照时间点，拷贝出来。那么导出来的格式是什么样的呢？