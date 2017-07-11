# 容器
> 统一使用4C16G的虚拟机 作为node节点

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

## swarm集群及服务
> 现在只把web服务和job服务这种计算**无状态服务**迁移到swarm集群上
> mysql/redis等存储服务尽量使用云服务

## 构建管理系统

使用shipyard作为docker及docker service管理工具

shipyard 安装配置



