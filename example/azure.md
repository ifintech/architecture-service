# 安装部署

### 创建集群

1. 申请虚拟云主机(4核16G **centos**系统)并初始化

2. 更新时区同步时间 

   ```shell
   cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
   ```

3. [安装docker](../docker/docker.md#安装Docker)

4. [创建Swarm集群](../docker/swarm.md#实例)

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

- 在服务器上配置内网dns

```shell
vim /etc/resolv.conf
nameserver {DNS_IP} #添加到首行
```

### HTTP网关

- [**部署orange**](../api-gateway.md)

  需要部署两个网关，分为应用网关及sa网关。应用网关公开80端口负责处理应用的流量，sa网关负责处理服务管理的流量，并限制访问ip。

- 配置orange

  1. 配置外网请求代理(*示例* 将http://demo.com的请求全部转发到demo_nginx的服务上)

     ![WX20170817-171125@2x](https://ws2.sinaimg.cn/large/006tNc79ly1fimup9cw8bj315s0mgabt.jpg)

     ![WX20170817-172803@2x](https://ws4.sinaimg.cn/large/006tKfTcly1fimupaob7lj315o0rignx.jpg)

  2. 配置内网请求代理(示例 转发http://gateway/demo/*的请求到demo_nginx的服务上)

     > **注意**：此配置在转发时不会带上query参数

     ![WX20170817-171621@2x](https://ws4.sinaimg.cn/large/006tKfTcly1fimupa85tuj315s0meaby.jpg)

     ![WX20170817-173001@2x](https://ws3.sinaimg.cn/large/006tKfTcly1fimup9q133j315q0tyjtz.jpg)

  3. 配置访问限制

### 日志服务

### 监控服务

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