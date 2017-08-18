# 监控报警

> 监控报警服务是针对资源及应用进行监控，采集监控指标，通过指标设置报警。
>
> 通过监控服务，可以全面了解线上资源使用情况、应用运行状况。借助报警服务，可以及时做出反应，保证服务稳定运行。

## 监控

### 资源监控

> 对服务资源进行监控, 保证运行环境的可用性

通过在主机上部署数据采集工具，采集各项指标，上送到数据存储中。

- 系统监控
  - CPU 使用率
  - 内存使用率
  - 硬盘吞吐量
  - 硬盘读写次数
  - 网卡出入带宽
- 容器监控
  - 容器状态
  - CPU使用
  - 内存使用
  - 网卡出入带宽
- 中间件监控
  - Mysql
  - Redis
  - Mq

### 应用监控

> 对应用日志进行监控, 保证业务持续稳定地运行

将应用日志通过日志服务进行解析和聚合，从而获得各项监控指标。

- 异常监控
  - error级别日志数
  - warining级别日志数
- 平均访问时长
- 访问成功率

### 通信监控

> 对Nginx日志进行监控, 保证线上通信稳定可靠

将Nginx日志通过日志服务进行解析和聚合，从而获得各项监控指标。

- 状态码监控
  - 500以上状态数
  - 400以上状态数
- 平均请求时长
- 请求成功率

### 实例

#### [Beats](https://www.elastic.co/cn/products/beats)

> 资源监控数据采集工具选型

**beats**是Elastic Stack下的一套开源数据采集工具。其中**Metricbeat**工具能够收集系统负载、内存、硬盘以及每个进程的情况, 也能深入容器内部进行采集，支持嗅探网络流量, 支持HTTP, Thrift-RPC, Mysql, Redis等协议, 最后将数据送到Logstash或Elasticsearch中, 进行后续的处理。

##### 部署

1. 通过docker swarm启动

   ```shell
   docker service create --name metricbeat \
     --mode global \
     --limit-cpu .5 \
     --limit-memory 128m \
     --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
     --mount type=bind,src=/proc,dst=/hostfs/proc,ro=true \
     --mount type=bind,src=/sys/fs/cgroup,dst=/hostfs/sys/fs/cgroup,ro=true \
     --mount type=bind,src=/,dst=/hostfs,ro=true \
     --env ES={ES_HOST}:9200 \
     --network host \
     ifintech/metricbeat
   ```

2. 导入可视化模板到kibana中

   ```shell
   docker run -t --rm \
   ifintech/metricbeat ./scripts/import_dashboards -es http://{ES_HOST}:9200 -url https://artifacts.elastic.co/downloads/beats/beats-dashboards/beats-dashboards-5.5.0.zip
   ```

## 报警

报警服务提供监控数据的报警功能。通过设置报警规则来定义报警系统如何检查监控数据，并在监控数据满足报警条件时发送报警通知。通过对重要的监控指标设置报警规则，可以在第一时间得知指标数据发生异常，迅速处理故障。

#### 报警级别

- **Notice** 该级别表示需要注意, 系统可能发生了问题, 需要人员进行关注
- **Warning** 该级别表示系统出现了一定错误, 但不影响主要业务的进行, 需要负责人查看具体情况并处理
- **Error** 该级别表示已经影响到了主要的业务流程, 需要人员立即介入处理

#### 报警方式

- 邮件
- 短信
- 钉钉
- 微信

#### 功能

- 报警触发

  按一定的规则检测监控数据，获取监控指标，超过设定阈值时则触发报警动作。

- 报警处理

  - 通知相关负责人
  - 自动化运维(触发一些系统动作)

- 控制台

  - 报警规则管理
  - 报警用户管理
  - 报警记录

### [报警控制台](https://github.com/ifintech/service)

日志服务及监控服务统一将数据统一存储在Elasticsearch中，而报警控制台通过Elasticsearch的查询聚合的功能来获取需要的监控指标。报警规则支持Elasticsearch已有的查询聚合方法，支持按多种维度对监控数据进行监测。

