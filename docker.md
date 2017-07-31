# 服务容器&集群

> 

### 容器
Docker在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。使得Docker技术比虚拟机技术更为轻便、快捷。

![](https://yeasy.gitbooks.io/docker_practice/content/introduction/_images/docker.png)

对于中小企业有以下好处

- 更高效的利用系统资源

- 一致的运行环境

- 持续交付和部署

- 完整的生态系统


### 集群编排调度

> A swarm is a cluster of Docker engines, or nodes, where you deploy services. The Docker Engine CLI and API include commands to manage swarm nodes (e.g., add or remove nodes), and deploy and orchestrate services across the swarm.

swarm node集群
![](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)

swarm集群编排调度是管理整个系统的服务架构的编排和运行时的调度（弹性伸缩，故障恢复，负载均衡等），从而实现比虚拟机集群时代更高的可靠性，和更多的灵活性。

![](https://docs.docker.com/engine/swarm/images/service-lifecycle.png)
swarm有以下好处

- 声明式服务架构模型

- 服务扩容缩容

- 协调预期状态与实际状态的一致性

- 服务发现

- 负载均衡

- 滚动更新（Rolling Update）

- 其他












  
  



