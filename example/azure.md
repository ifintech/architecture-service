# 安装部署

### 创建集群

1. 申请虚拟云主机(4核16G **centos**系统)并初始化

2. 更新时区 同步时间 

   ```shell
   cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
   ```

3. [挂载硬盘](https://docs.azure.cn/zh-cn/virtual-machines/linux/add-disk?toc=%2fvirtual-machines%2flinux%2ftoc.json)

   因主机上需要存储docker镜像及其他落地的数据，而主机自带的存储空间不大，需按需来挂载数据磁盘。

4. [安装docker](../docker/docker.md#安装Docker)

5. [创建Swarm集群](../docker/swarm.md#使用)

### 镜像仓库

- [搭建私有镜像仓库](../docker/dockerhub.md#Registry)

> 私有镜像仓库需要绑定一个域名，以便推送及拉取镜像。可以通过搭建内网dns在线上环境解析该域名。

- 搭建内网dns

```shell
# 定义私有镜像仓库域名解析
 echo '127.0.0.1 dockerhub.com' >> /data1/dns/etc/dnshosts
 docker config create dns-dnshosts /data1/dns/etc/dnshosts
 
# 启动服务 2个实例 对外提供53端口dns服务
 docker service create --name dns \
     -p 53:53/udp \
     --replicas 2 \
     --limit-cpu .5 \
     --limit-memory 128m \
     --config source=dns-dnshosts,target=/etc/dnshosts \
     --update-parallelism 1 \
     --update-delay 5s \
     ifintech/dns
```

- 在所有节点上配置内网dns

```shell
vim /etc/resolv.conf
nameserver {DNS_IP} #添加到首行
```

- 在master节点上登录私有镜像仓库

```shell
docker login [REGISTRY_HOST] -u [用户名] -p [密码]
```

### HTTP网关

- [部署orange](../api-gateway.md)

  需要部署两个网关，分为应用网关及sa网关。应用网关公开80端口负责处理应用的流量，sa网关负责处理服务管理的流量，并限制访问ip。

- [申请负载均衡](https://docs.azure.cn/zh-cn/load-balancer/load-balancer-get-started-ilb-arm-portal)

  负载均衡与网关一一对应，连接到网关的80/81端口，并探测后端网关节点的健康情况。这样即使有节点宕掉，也不会影响整体Web服务的可用性。

- 配置orange

  1. 配置外网请求代理(示例：将`http://demo.com`的请求全部转发到demo_nginx的服务上)

     ![WX20170817-171125@2x](https://ws2.sinaimg.cn/large/006tNc79ly1fimup9cw8bj315s0mgabt.jpg)

     ![WX20170817-172803@2x](https://ws4.sinaimg.cn/large/006tKfTcly1fimupaob7lj315o0rignx.jpg)

  2. 配置内网请求代理(示例：转发`http://gateway/demo/*`的请求到demo_nginx的服务上)

     > **注意**：此配置在转发时不会带上query参数

     ![WX20170817-171621@2x](https://ws4.sinaimg.cn/large/006tKfTcly1fimupa85tuj315s0meaby.jpg)

     ![WX20170817-173001@2x](https://ws3.sinaimg.cn/large/006tKfTcly1fimup9q133j315q0tyjtz.jpg)

  3. 配置访问限制 (限制`/admin`的路由只能由限定的ip访问)

     ![WX20170818-152000@2x](https://ws3.sinaimg.cn/large/006tNc79ly1finwtdu2olj315e0m2myz.jpg)

     ![WX20170818-152356@2x](https://ws2.sinaimg.cn/large/006tNc79ly1finwtcwls9j313a0jsgnc.jpg)

### 出口网关

> 如果集群中的机器无法访问外网或某些依赖的外部服务有访问限制，必须以固定ip去访问，在此时就需要一个出口网关，负责代理那些外网流量。

1. 部署squid

```shell
docker service create --name out-gateway \
-p 3128:3128 \
--mount type=bind,src=/srv/docker/squid/cache,dst=/var/spool/squid3 \
--replicas 2 \
--limit-cpu 0.5 \
--limit-memory 512mb \
ifintech/squid
```

2. 代理地址为`http://{HOST_IP}:3128`

### 日志服务

1. [部署日志服务](../service/log.md#实例)

   > 如果日志格式不同或日志的处理方式有异，则需要自定义logstash的配置文件，重新打包logstash镜像。

2. 在SA网关为kibana添加代理配置

   ![WX20170818-112612@2x](https://ws4.sinaimg.cn/large/006tNc79ly1finwtf0eogj315q0maq4o.jpg)

   ![WX20170818-112826@2x](https://ws4.sinaimg.cn/large/006tNc79ly1finwtfv6wcj313a0r0wgl.jpg)

3. 访问kibana，初始化kibana。

### 监控服务

1. [部署beats](../service/monitor.md#Beats)
2. [部署服务管理中心](https://github.com/ifintech/service)
3. 在SA网关为服务管理中心添加代理配置(配置方法同上)
4. 在监控报警控制台，添加报警规则及报警组
5. 绑定报警检测脚本

### 认证服务

> 向外提供ldap及二步认证服务

**服务编排模板**

```yaml
version: "3.2"

services:
  ldap:
    image: ifintech/ldap2http
    ports:
      - "10389:10389"
    environment:
      HOST: 0.0.0.0
      PORT: 10389
      AUTH_URL: http://auth_nginx
      AUTH_TOKEN: {AUTH_SECRET}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 50M
      update_config:
        parallelism: 1
        delay: 30s
    networks:
      - servicenet
  php:
    image: dockerhub.jrmf360.com/sa/auth
    command: php-fpm
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 50M
      update_config:
        parallelism: 1
        delay: 30s
    networks:
      - servicenet
  nginx:
    image: ifintech/nginx-php
    networks:
      - servicenet
    environment:
      APP_NAME: auth
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 50M
      update_config:
        parallelism: 1
        delay: 30s
networks:
  servicenet:
    external: true
```

### 应用栈

> 一个应用栈中会包含多个应用，我们通过服务编排统一启动服务

#### PHP应用栈

**服务编排模板**

```yaml
version: "3.2"

services:
  front: # web前端应用
    image: {IMAGE_ADDRESS}
    networks:
      - servicenet
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 100M
      update_config:
        parallelism: 1
        delay: 10s
  php: # php应用
    image: {IMAGE_ADDRESS}
    networks:
      - servicenet
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.1'
          memory: 100M
      update_config:
        parallelism: 1
        delay: 10s
    command: ["php-fpm"]
  nginx: #nginx应用 负责给php应用提供http通信的支持
    image: ifintech/nginx-php
    networks:
      - servicenet
    environment:
      APP_NAME: {APP_NAME}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 50M
      update_config:
        parallelism: 1
        delay: 10s
networks:
  servicenet:
    external: true
```

**启动**

```shell
docker stack deploy {APP} -c compose-stack-{APP}.yml --with-registry-auth
```

**更新镜像**

```shell
docker service update {SERVICE_NAME} --image {IMAGE} --with-registry-auth
```

#### JAVA应用栈

**服务编排模板**

```yaml
version: "3.2"

services:
  api:
    image: {IMAGE_ADDRESS}
    networks:
      - servicenet
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.1'
          memory: 250M
      update_config:
        parallelism: 1
        delay: 30s
  core:
    image: {IMAGE_ADDRESS}
    networks:
      - servicenet
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.1'
          memory: 250M
      update_config:
        parallelism: 1
        delay: 30s
networks:
  servicenet:
    external: true
```

**启动**

```shell
docker stack deploy {APP} -c compose-stack-{APP}.yml --with-registry-auth
```

**更新镜像**

```shell
docker service update {SERVICE_NAME} --image {IMAGE} --with-registry-auth
```

### 任务调度

> **注意**：通过此方式调度的任务无法连接swarm network，故无法直接访问swarm网络中的服务。

- [部署任务调度agent](https://github.com/ifintech/job) (为了保证高可用，需要至少双机部署)
- [部署任务调度服务端](https://github.com/ifintech/service)(已与服务管理中心后台整合 无需重复部署)
- 在后台按需添加定时任务及一次性任务