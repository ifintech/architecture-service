# 服务容器&集群

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
> http://www.dockerinfo.net/4374.html

### Docker Swarm具有如下基本特性：
- 集群管理集成进Docker Engine
使用内置的集群管理功能，我们可以直接通过Docker CLI命令来创建Swarm集群，然后去部署应用服务，而不再需要其它外部的软件来创建和管理一个Swarm集群。
- 去中心化设计
Swarm集群中包含Manager和Worker两类Node，我们可以直接基于Docker Engine来部署任何类型的Node。而且，在Swarm集群运行期间，我们既可以对其作出任何改变，实现对集群的扩容和缩容等，如添加Manager Node，如删除Worker Node，而做这些操作不需要暂停或重启当前的Swarm集群服务。
- 声明式服务模型（Declarative Service Model）
在我们实现的应用栈中，Docker Engine使用了一种声明的方式，让我们可以定义我们所期望的各种服务的状态，例如，我们创建了一个应用服务栈：一个Web前端服务、一个后端数据库服务、Web前端服务又依赖于一个消息队列服务。
- 服务扩容缩容
对于我们部署的每一个应用服务，我们可以通过命令行的方式，设置启动多少个Docker容器去运行它。已经部署完成的应用，如果有扩容或缩容的需求，只需要通过命令行指定需要几个Docker容器即可，Swarm集群运行时便能自动地、灵活地进行调整。
- 协调预期状态与实际状态的一致性
Swarm集群Manager Node会不断地监控集群的状态，协调集群状态使得我们预期状态和实际状态保持一致。例如我们启动了一个应用服务，指定服务副本为10，则会启动10个Docker容器去运行，如果某个Worker Node上面运行的2个Docker容器挂掉了，则Swarm Manager会选择集群中其它可用的Worker Node，并创建2个服务副本，使实际运行的Docker容器数仍然保持与预期的10个一致。
- 多主机网络
我们可以为待部署应用服务指定一个Overlay网络，当应用服务初始化或者进行更新时，Swarm Manager在给定的Overlay网络中为Docker容器自动地分配IP地址，实际是一个虚拟IP地址（VIP）。
- 服务发现
Swarm Manager会给集群中每一个服务分配一个唯一的DNS名称，对运行中的Docker容器进行负载均衡。我们可以通过Swarm内置的DNS Server，查询Swarm集群中运行的Docker容器状态。
- 负载均衡
在Swarm内部，可以指定如何在各个Node之间分发服务容器（Service Container），实现负载均衡。如果想要使用Swarm集群外部的负载均衡器，可以将服务容器的端口暴露到外部。
- 安全策略
在Swarm集群内部的Node，强制使用基于TLS的双向认证，并且在单个Node上以及在集群中的Node之间，都进行安全的加密通信。我们可以选择使用自签名的根证书，或者使用自定义的根CA（Root CA）证书。
- 滚动更新（Rolling Update）
对于服务需要更新的场景，我们可以在多个Node上进行增量部署更新，Swarm Manager支持通过使用Docker CLI设置一个delay时间间隔，实现多个服务在多个Node上依次进行部署。这样可以非常灵活地控制，如果有一个服务更新失败，则暂停后面的更新操作，重新回滚到更新之前的版本。

### 创建集群
init swarm
```bash
docker swarm init --advertise-addr <MANAGER-IP>
```

添加进集群
```bash
docker swarm join \
--token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
192.168.99.100:2377
```

显示token cmd
```bash
docker swarm join-token manager
```

集群节点
```bash
docker node ls
```

### 管理服务
在Swarm集群上部署服务，必须在Manager Node上进行操作。先说明一下Service、Task、Container（容器）这个三个概念的关系，如下图（出自Docker官网）非常清晰地描述了这个三个概念的含义：
![](http://img.dockerinfo.net/2017/03/20170315210902.jpg)

创建Docker服务，可以使用docker service create命令实现
```bash
docker service create --replicas 2 --name my_web nginx
```
扩容缩容服务
```bash
docker service scale my_web=3
```
删除服务
```bash
docker service rm my_web
```
滚动更新(**具体参考服务持续发布**)
```bash
docker service update my_web
```

### 构建管理系统  
> ** 等待shipyard-swarm版本发布 **

使用shipyard作为docker及docker service管理工具
> Shipyard enables multi-host, Docker cluster management. It uses Docker Swarm for cluster resourcing and scheduling.

shipyard 安装配置
```bash
curl -sSL https://shipyard-project.com/deploy | TLS_CERT_PATH=$(pwd)/certs bash -s
```

### 监控监控
### 容器镜像

## 安装部署

- 申请云主机3台（统一使用4C16G的虚拟机 作为docker swarm node节点）




