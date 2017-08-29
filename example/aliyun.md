# 部署方案

### 网络规划设计

申请阿里云VPS，建议网段10.0.0.0/8，划分路由器10.1.0.0/16为swarm集群网段。
申请阿里云NAT，规格小型，按照需求购买共享带宽包一般20M足够，设置SNAT绑定公网IP。所有集群内机器不再绑定公网IP。
申请外网负载均衡设备，用来用户请求访问

### 集群构建
> 可以通过阿里云的界面申请

1. 网络区域选择VPS所在的区域，选择对应的专有网络，选择swarm mode
1. 节点选择虚拟云主机(系列3，2核16G500G硬盘 **centos**系统)，实例数据量最好大于6台，数据盘选择200G以上，不保留公网IP
1. 创建私有网络service，覆盖所有节点

### 服务构建

#### 镜像仓库

> 使用阿里云自带镜像仓库，设置密码，创建私有本地镜像仓库

#### 外部HTTP网关

#### 内部HTTP网关

#### 日志服务

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

   启动

   ```shell
   docker stack deploy log -c compose-stack-log.yml
   ```

4. 验证

   - ElasticSearch

     ```shell
     curl -v 'http://{HOST_IP}:9200/_cluster/health?pretty' #查看集群状态
     ```





#### 监控服务

#### 认证服务

#### 数据统计

#### 整体服务后台

### 业务构建

### 存储服务

尽量使用云平台提供的云服务方案，比如mysql,redis









