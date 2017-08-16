# 服务编排及调度

> 初级的编排，是资源的编排，即针对容器编排；但更高层次的是服务的编排，需要对架构层次在整体上有一个完整的定义类似docker-compose

## swarm
> 摘抄自官方文档 https://docs.docker.com/engine/swarm/#feature-highlights

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

## 管理Swarm Node
Swarm支持设置一组Manager Node，通过支持多Manager Node实现HA。那么这些Manager Node之间的状态的一致性就非常重要了，多Manager Node的Warm集群架构，如下图所示（出自Docker官网）：
![](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)
通过上图可以看到，Swarm使用了Raft协议来保证多个Manager之间状态的一致性。基于Raft协议，Manager Node具有一定的容错功能，假设Swarm集群中有个N个Manager Node，那么整个集群可以容忍最多有(N-1)/2个节点失效。如果是一个三Manager Node的Swarm集群，则最多只能容忍一个Manager Node挂掉。

## 管理服务
一个服务并非一个容器，一个服务可以有多个副本任务task（创建service的时候由replicas指定）。每个task对应着一个容器。副本数量可以通过scale操作来增加或者收缩。一般使得副本任务数能均匀分散在各个节点（默认调度策略也是spread），这样一个节点挂了之后，仍然有其他容器可以提供服务

在Swarm集群上部署服务，必须在Manager Node上进行操作。先说明一下Service、Task、Container（容器）这个三个概念的关系，如下图（出自Docker官网）非常清晰地描述了这个三个概念的含义：
![](http://img.dockerinfo.net/2017/03/20170315210902.jpg)

### 负载均衡
swarm mode里面已经内置了负载均衡的功能。负载均衡器和容器集成在一起，入下图。swarm里面的服务作为一个整体交给swarm通过一个公有端口暴露给外界访问。对服务的访问，可以通过负载均衡，分配到不同的task上。

![](/images/ipvs.png)

### 创建服务

> 官方文档 https://docs.docker.com/engine/swarm/services/#create-a-service

在docker swarm中部署应用程序，是通过创建service来实现，这里的service的概念通过是指在一个大的应用上下文中的一个微服务，比如在电商的购物网站中：用户管理、订单管理、库存管理，都是购物网站这个大应用中的一个一个微服务，这些服务可以是单个或者多个的在集群模式中运行。
service的类型可以是HTTP的应用程序、数据库、缓存服务等分布式环境中任何可执行的程序。
service可定义的选项：

- 可以在swarm集群的外部提供访问服务的端口
- 可以通过覆盖网络(overlay)模式连接到群中的其他服务
- 可以设置CPU和内存限制和预留值
- 可以实现滚动更新策略
- 可以指定要在群中运行的镜像的副本数

简单例子
```bash
docker service create nginx
```

```bash
docker service create --name my_web --replicas 3 --publish 8080:80 nginx
```

### 扩容缩容服务

Docker Swarm支持服务的扩容缩容，Swarm通过--mode选项设置服务类型，提供了两种模式：一种是replicated，我们可以指定服务Task的个数（也就是需要创建几个冗余副本），这也是Swarm默认使用的服务类型；另一种是global，这样会在Swarm集群的每个Node上都创建一个服务。如下图所示（出自Docker官网），是一个包含replicated和global模式的Swarm集群

服务部署的复制模式和全局模式说明
![](https://docs.docker.com/engine/swarm/images/replicated-vs-global.png)

扩容缩容命令
```bash
docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS>
```
### 滚动更新

在swarm中部署Redis3.0.7，并配置10秒更新延迟
```bash
docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.7-alpine
```
我们在部署服务指定滚动更新策略。--update-delay表示更新服务对应的任务或一组任务之间的时间间隔。时间间隔用数字和时间单位表示，m表示分，h表示时，所以10m30s 表示10分30秒的延时。

默认情况下，调度器一次更新一个任务。你可以使用--update-parallelism标志配置调度器每次同时更新的最大任务数量。

默认情况下，如果更新某个任务返回了RUNNING状态，调度器会转去更新另一个任务，直到所有任务都更新完成。如果在更新某个任务的任意时刻返回了FAILED，调度器暂停更新。我们可以在执行 docker service create 命令和 docker service update 命令时使用 --update-failure-action 标志来覆盖这种默认行为。

更新redis服务使用的容器镜像,swarm管理节点会根据UpdateConfig策略来更新节点
```bash
docker service update --image redis:3.2.5-alpine redis 
```

调度器根据下面默认的策略来应用滚动更新：

1. 停止第一个任务。
2. 为停止的任务应用更新。
3. 为更新的任务启动容器。
4. 如果更新任务时返回RUNNING，等待一个指定的延时后停止下一个任务。
5. 如果在更新的任意时刻，某个任务返回FAILED，暂停更新。


### 实例

- init swarm

```bash
docker swarm init --advertise-addr <MANAGER-IP>
```

- join swarm 

```bash
# 显示join-token worker和manager角色拥有不同的join-token
docker swarm join-token worker|manager

docker swarm join \
--token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
<MANAGER-IP>:2377

# 查看集群节点
docker node ls
```

- 创建swarm网络

```shell
docker network create --driver overlay --subnet 192.168.1.0/24 servicenet
```

