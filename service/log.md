# 日志服务

> 日志用来记录用户操作、系统运行状态等，是系统中的重要组成部分。日志记录的好坏直接关系到系统出现问题时定位的速度，同时通过对日志的观察和分析，可以提前发现系统可能的风险，避免线上事故的发生。

## 日志规范

> 为了方便对日志的查询及聚合, 对日志当中的常用字段进行了统一规约. 
>
> 另外为了方便日志的解析处理, 推荐日志使用**json**的格式进行记录和传输.

#### 应用(PHP)日志字典

| 字段            | 说明                |
| ------------- | ----------------- |
| date          | 日志记录时间            |
| x_rid         | 请求ID              |
| server_ip     | 服务器IP             |
| client_ip     | 客户端IP             |
| uri           | 访问路径              |
| params        | 请求参数              |
| exectime      | 执行时间(毫秒)          |
| succ          | 执行结果 succ/fail    |
| retcode       | 返回码  成功为`2000000` |
| retmsg        | 返回信息              |
| error_type    | 错误类型              |
| error_code    | 错误码               |
| error_message | 错误信息              |
| error_trace   | 错误栈信息             |

#### Nginx日志字典

| 字段           | 说明                 |
| ------------ | ------------------ |
| date         | 日志记录时间             |
| x_rid        | 请求ID               |
| server_ip    | 服务器IP              |
| client_ip    | 客户端IP              |
| domain       | 主机域名               |
| uri          | 访问路径               |
| size         | 请求体大小(字节)          |
| responsetime | 请求响应时间(秒)          |
| upstreamtime | nginx向后端转发请求的时间(秒) |
| upstreamhost | nginx服务器地址         |
| xff          | 客户端真实ip(使用代理时)     |
| referer      | 请求来源               |
| agent        | 用户访问代理(系统信息 浏览器信息) |
| status       | http 状态码           |

#### 日志级别

- **INFO** 该种日志记录系统的正常运行状态，例如某个子系统的初始化，某个请求的成功执行等等。
- **DEBUG** 该级别日志的主要作用是对系统每一步的运行状态进行精确的记录。通过该种日志，可以查看某一个操作每一步的执行过程，可以准确定位是何种操作，何种参数，何种顺序导致了某种错误的发生。
- **WARNING** 该日志表示系统可能出现问题。对于那些目前还不是错误，然而不及时处理也会变为错误的情况，也可以记为WARNING日志。
- **ERROR**  该级别的错误需要立刻被处理。当ERROR错误发生时，已经影响了用户的正常访问。
- **STAT** 打点日志, 统计用。如记录用户访问的页面，业务流程中的某个动作等记录到数据仓库，方便后续的统计分析

## 日志收集

纵览当前容器日志收集的场景，无非两种方式：一是直接采集Docker标准输出，容器内的服务将日志信息写到标准输出，再将Docker的标准输出发送到相应的收集程序中；二是延续传统的日志写入方式，容器内的服务将日志直接写到普通文件中，通过Docker volume将日志文件映射到Host上，日志采集程序就可以收集它。

Docker官方建议采用使用标准输出来记录日志，这种方式也符合`一容器一个服务`的使用方式；同时也足够简单清晰，能够统一所有服务中日志的记录方式，方便接入日志服务，无需额外依赖。

#### [Logspout](https://github.com/gliderlabs/logspout)

Logspout是一款用于收集Docker容器日志的工具。它可以连接到主机上的所有容器，捕获其标准输出及错误输出，然后将其路由到你想让让它去的地方，支持Syslog，Logstash，Redis，Kafka。

其部署简单，配置方便，故作为日志收集工具选型。

##  日志传输

日志收集服务可以将一台主机上所有容器都采集回来，我们需要将采集回来的日志发送到日志处理服务处以供后续的处理消费。视日志数据的量级，有多种日志传输中间件供选型。

#### Syslog

Syslog常被称为系统日志或系统记录，是一种用来在互联网协议（TCP/IP）的网络中传递记录档讯息的标准。在日志数据量较少，日志处理服务压力不大，可以采用Syslog传输日志，无需依赖其他服务。

Syslog无落地策略，存在日志丢失的风险，请注意。

> 使用消息队列服务可以避免日志丢失的风险，并且当日志写入量过大，后方日志处理服务无法及时处理时，可以使用消息队列来缓冲日志，削峰填谷。

#### Redis

Redis是一个开源，内存存储的数据结构服务器，可用作数据库，高速缓存和消息队列代理。当日志数据量不是十分大的情况下，可以使用Redis作为传输方案。

#### Kafka

Kafka是一种分布式的，基于发布/订阅的消息系统。主要设计目标如下：**以时间复杂度为O(1)的方式提供消息持久化能力，并保证即使对TB级以上数据也能保证常数时间的访问性能**。当日志数据量过大时，推荐使用Kafka。

## 日志处理

> 系统里的各种服务都会产生日志，采集回来的日志格式也是各种各样，而且针对不同服务的日志会需要不同的方式来处理它们。

日志处理服务从上游获取日志数据，并将不同的格式日志数据解析并格式化，最后按日志类型来选择不同的方式来消费日志。消费方式有：

- 将解析好的日志数据放入日志存储服务
- 日志文本落地持久化

#### Logstash

Logstash 是开源的具有实时输入数据能力的数据收集引擎。Logstash可以把来自不同的源的数据动态地写入你选择的目的地并对输入的数据进行规范化，它有强大的插件功能，可以满足我们对日志预处理的需求。

## 日志存储

为了解决服务治理的痛点，我们需要基于日志之上的分析，以便快速发现问题，定位问题，排查问题。日志存储服务会对日志数据进行索引，并向外提供日志查询，聚合，分析的功能。

#### Elasticsearch

Elasticsearch 是一个分布式、可扩展、实时的搜索与数据分析引擎. 它支持全文检索、结构化搜索、数据分析、复杂的语言处理、地理位置和对象间关联关系等。Elasticsearch 有非常多的优点: 高性能、可扩展、近实时搜索，并支持大数据量的数据分析。

## 日志可视化

对于日志统计分析的结果，需要一个可视化的界面进行展示，方便系统负责人对系统的关键指标和运行状况能有一个直观的了解。

#### 展示项

**应用**

- 访问量
- 访问分布
- 成功率
- 平均执行时长
- 警告数
- 错误数

**Nginx**

- 访问量
- 访问分布
- 成功率
- 平均执行时长
- 4XX状态数量
- 5XX状态数量

#### Kibana

Kibana是一个Elasticsearch数据分析和可视化的开源平台，使用Kibana能够搜索、展示存储在Elasticsearch中的索引数据，使用它可以很方便用图表、表格、地图展示和分析数据。

## 日志查看

在系统异常时，需要查看具体的线上日志来定位具体问题。kibana有数据的展示界面，但对日志的查看不太友好，因此需要一个直观方便的日志查看工具来满足这种需求。

#### Terminal

提供一个终端界面，可以访问、查询落地的日志，操作方法同使用终端一致。

**方案**

在日志机上运行容器，把日志目录以只读挂载进去，并向外提供ssh接入的端口。用户通过终端连接到容器里，查看日志。此种方案可以隔离应用日志，方便管理日志。

#### [LogTrail](https://github.com/sivasamyk/logtrail)

kibana的插件 在kibana中提供一个类命令行的界面来查看日志。

在输入框中输入查询语句进行查询搜索, 查询语法同[kibana查询语法](http://www.tuicool.com/articles/VZfim2)

![screenshot](https://ws1.sinaimg.cn/large/006tNc79ly1fi2tssfdwfj312w0mcn7y.jpg)

## 实例

> 所有服务皆基于**docker-swarm**安装运行

#### Logspout

```shell
docker service create --name logspout \
  --mode global \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --limit-cpu .5 \
  --limit-memory 512m \
  --network servicenet \
  gliderlabs/logspout syslog+tcp://log_logstash:10514
```

**使用**

- logspout可以忽略特定的容器，不去收集它们的日志，只需要在运行容器时添加一个环境变量`LOGSPOUT=ignore`进去即可。
- 容器启动时如果带有`-t`参数，logspout也将不再收集它们的日志。

#### ELK

> **注意**：logstash镜像中已打包了一个[默认的日志解析配置](https://github.com/ifintech/dockerhub-base/blob/master/logstash/logstash.conf)，如果日志格式不同，则需要按需修改配置文件。

1. 指定Elasticsearch运行的机器

   > ES的数据是需要永久存储，并且需要在每个节点有独立的存储路径来存储数据。所以我们需要确保es运行在指定的节点上，而不会被调度到其他节点上。

   ```shell
   docker node update --label-add type=es service-docker1
   ```

2. 修改ES集群上所有主机的配置参数(ElasticSearch需求)

   ```shell
   sudo sysctl -w vm.max_map_count=262144
   sudo mkdir -m 777 /mnt/docker/es #es数据存储路径
   ```

3. 以**stack**方式启动服务

   添加配置文件 **compose-stack-log.yml**

   ```yaml
   version: "3.2"

   services:
     logstash:
       image: ifintech/logstash
       networks:
         - servicenet
       volumes:
         - type: bind
           source: /tmp/logs #日志落地路径
           target: /mnt/logs
       deploy:
         replicas: 1
         resources:
           limits:
             cpus: '2'
             memory: 2G
           reservations:
             cpus: '.1'
             memory: 100M
         update_config:
           parallelism: 1
           delay: 30s
     elasticsearch:
       image: ifintech/swarm-elasticsearch
       ports:
         - 9200:9200
       networks:
         - servicenet
       environment:
         SERVICE_NAME: elasticsearch
         discovery.zen.minimum_master_nodes: 3
         path.data: /usr/share/elasticsearch/data
       volumes:
         - type: bind
           source: /mnt/docker/es #es数据存储路径
           target: /usr/share/elasticsearch/data
       deploy:
         replicas: 3
         placement:
           constraints:
             - node.labels.type == es
         resources:
           limits:
             cpus: '1'
             memory: 4G
           reservations:
             cpus: '.1'
             memory: 1G
         update_config:
           parallelism: 1
           delay: 30s
     kibana:
       image: kibana:5.5.1
       networks:
         - servicenet
       environment:
         ELASTICSEARCH_URL: http://elasticsearch:9200
       deploy:
         replicas: 2
         resources:
           limits:
             cpus: '.5'
             memory: 1G
           reservations:
             cpus: '.1'
             memory: 50M
         update_config:
           parallelism: 1
           delay: 10s
   networks:
     servicenet:
       external: true
   ```

   启动

   ```shell
   docker stack deploy log --compose-file compose-stack-log.yml
   ```

4. 验证

   - ElasticSearch

     ```shell
     curl -v 'http://{HOST_IP}:9200/_cluster/health?pretty' #查看集群状态
     ```

   - Kibana

     ```shell
     curl -I 'http://{HOST_IP}:5601' #返回200即代表安装成功
     ```

##  参考资料

- [ELK中文指南](http://kibana.logstash.es/content/)
- [logstash插件文档](https://www.elastic.co/guide/en/logstash/current/index.html)
- [grok表达式检测](http://grokdebug.herokuapp.com)
- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
- [kibana查询语法](http://www.tuicool.com/articles/VZfim2)