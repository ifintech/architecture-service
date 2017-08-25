# 实例

## 数据采集

> 通常来说，建议一个打点指标，一张数据库表，一个文件与数据仓库的索引一一对应。不过如果对数据存在纵向拆分的情况，则需要合并后再对应到一个索引上。

#### go-mysql-elasticsearch

- 待补充

#### logstash-input-jdbc

- 待补充

## ELK

> 通过docker swarm安装服务，与日志服务共用logstash，不需要再另外部署。

1. 指定Elasticsearch运行的机器

   > ES的数据是需要永久存储，并且需要在每个节点有独立的存储路径来存储数据。所以我们需要确保es运行在指定的节点上，而不会被调度到其他节点上。

   ```shell
   docker node update --label-add type=stat-es service-docker1
   ```

2. 修改ES集群上所有主机的配置参数(ElasticSearch需求)

   ```shell
   sudo sysctl -w vm.max_map_count=262144
   sudo mkdir -m 777 /mnt/docker/es #es数据存储路径
   ```

3. 以**stack**方式启动服务

   添加配置文件 **compose-stack-stat.yml**

   ```yaml
   version: "3.2"

   services:
     es:
       image: ifintech/swarm-elasticsearch
       ports:
         - 9201:9200
       networks:
         - servicenet
       environment:
         SERVICE_NAME: stat_es
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
             - node.labels.type == stat-es
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
       image: kibana:5.5.0
       networks:
         - servicenet
       environment:
         ELASTICSEARCH_URL: http://stat_es:9200
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
   docker stack deploy stat -c compose-stack-stat.yml
   ```

## 数据可视化

> 登录kibana，为数据配置可视化图形。

- [如何使用Kibana仪表盘和可视化](https://www.howtoing.com/how-to-use-kibana-dashboards-and-visualizations/)
- [kibana可视化图形介绍](https://kibana.logstash.es/content/kibana/v5/visualize/)

## 数据报表

### 实时报表

> 通过**iframe**的形式嵌入报表到自有系统中

1. 设置kibana服务器允许iframe请求

   - 添加HTTP响应头 `X-Frame-Options ALLOWALL`

2. 在前端展示页面中嵌入iframe

   ```html
   <iframe id="kibana" height="1200px" frameborder="0" scrolling="no" marginheight="0" marginwidth="0" width="100%" src="<?=$url?>"></iframe>
   ```

### 离线报表

> 定时调用**phantomjs**对相应报表进行截图保存，再进行后续处理（发送邮件、备份等）。

- 截图

  ```shell
  #示例
  docker run --rm -v /tmp:/tmp ifintech/phantomjs http://www.baidu.com /tmp/screen_shot.png
  ```

- 将截图文件备份&发送邮件

- 通过**任务调度服务**绑定此任务

