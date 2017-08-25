# 服务报告

通过日志服务及监控服务，我们已经将系统资源、网络请求、服务运行的各种信息收集回来，并统一存储，但通过这些繁杂的数据无法直观地得出系统运行的关键指标。服务报告就致力于将这些数据可视化，并加之组合形成**从下到上全栈的系统报表**。

系统报表可分为两种形式，分别为：

- 实时报表

  实时报表反馈了系统当前整体的运行情况，包含基础层，中间件层，应用层的监控指标项，并且支持回溯至某一段时间上。通过报表项目负责人可以了解系统的从上到下整体的情况；并且当系统出现问题时，可及时了解影响的范围及规模，帮助技术人员快速定位问题。

- 离线报表

  离线报表是对一段时间内的系统报表进行保存归档，方便日后的查询及回溯；并且会通过一定手段(如邮件等)发送至系统负责人处，方便负责人时时查阅，了解系统近段时间内的服务质量。

## 实例

### 建立可视化

> 通过kibana实现

- 导入应用报表

  1. 在kibana后台创建`logstash-app-*`的索引
  2. 在kibana后台*#/management/kibana/objects*处导入可视化面板[模板](https://raw.githubusercontent.com/ifintech/service/master/build/template/app.json)

- 导入Nginx报表

  1. 在kibana后台创建`logstash-nginx-*`的索引
  2. 在kibana后台导入可视化面板[模板](https://raw.githubusercontent.com/ifintech/service/master/build/template/nginx.json)

- 导入docker报表

  ```shell
  docker run -t --rm \
  ifintech/metricbeat ./scripts/import_dashboards -es http://{ES_HOST}:9200 -url https://artifacts.elastic.co/downloads/beats/beats-dashboards/beats-dashboards-5.5.0.zip
  ```

### 实时报表

已与[**服务管理中心**](https://github.com/ifintech/service)整合，登录后台便可查看每个服务的实时报表，支持回溯。

![WX20170825-161212@2x](https://ws2.sinaimg.cn/large/006tNc79ly1fiw2uovwioj31kw0rlae7.jpg)

![WX20170825-161248@2x](https://ws3.sinaimg.cn/large/006tNc79ly1fiw2uptx1bj31kw0o242h.jpg)

### 离线报表

