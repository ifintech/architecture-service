# 容器
> 统一使用4C16G的虚拟机

## docker 
docker安装

>官方安装 https://store.docker.com/editions/community/docker-ce-server-centos?tab=description

中国镜像安装最新稳定版
```Shell
curl -sSL https://get.daocloud.io/docker | sh
```

键入docker -v命令，若显示如下返回则表示安装成功

```shell
[root@localhost vagrant]# docker -v
Docker version 17.03.0-ce, build 60ccb22
```

docker服务启动 重启 停止

```Shell
[root@localhost vagrant]# service docker start
Redirecting to /bin/systemctl start  docker.service

[root@localhost vagrant]# service docker restart
Redirecting to /bin/systemctl restart  docker.service

[root@localhost vagrant]# service docker stop
Redirecting to /bin/systemctl stop  docker.service
```

## swarm  
> 现在只把web服务和job服务这种计算**无状态服务**迁移到swarm集群上
> mysql/redis等存储服务尽量使用云服务
> 一类相同的业务使用一套集群

#### 构建集群

创建集群
```Shell
docker swarm init --advertise-addr <MANAGER-IP>
```

添加进集群
```shell
docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377
```
    
显示token cmd
```shell
docker swarm join-token manager
```
    
集群节点
```shell
docker node ls
```

发布服务
```shell
docker service create --replicas 2 -p 80:80 --name app_name image cmd
```

查看服务
```shell
docker service ls
docker service ps app_name
```

调整服务规模
```shell
docker service scale app_name=4
```

更新服务
```shell
docker service update app_name
```

#### docker镜像


#### 服务