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

### 日志服务

### 监控服务

### Auth服务

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