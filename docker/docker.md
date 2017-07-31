#服务容器

### 为什么要使用 Docker？

作为一种新兴的虚拟化方式，Docker跟传统的虚拟化方式相比具有众多的优势。

- 更高效的利用系统资源

    由于容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，Docker 对系统资源的利用率更高。无论是应用执行速度、内存损耗或者文件存储速度，都要比传统虚拟机技术更高效。因此，相比虚拟机技术，一个相同配置的主机，往往可以运行更多数量的应用。

- 更快速的启动时间

    传统的虚拟机技术启动应用服务往往需要数分钟，而 Docker 容器应用，由于直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间。大大的节约了开发、测试、部署的时间。

- 一致的运行环境

    开发过程中一个常见的问题是环境一致性问题。由于开发环境、测试环境、生产环境不一致，导致有些 bug 并未在开发过程中被发现。而 Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性，从而不会再出现 “这段代码在我机器上没问题啊” 这类问题。

- 持续交付和部署

    对开发和运维（DevOps）人员来说，最希望的就是一次创建或配置，可以在任意地方正常运行。
    
    使用*Docker*可以通过定制应用镜像来实现持续集成、持续交付、部署。开发人员可以通过Dockerfile来进行镜像构建，并结合 持续集成(Continuous Integration) 系统进行集成测试，而运维人员则可以直接在生产环境中快速部署该镜像，甚至结合持续部署(Continuous Delivery/Deployment)系统进行自动部署。
    
    而且使用Dockerfile使镜像构建透明化，不仅仅开发团队可以理解应用运行环境，也方便运维团队理解应用运行所需条件，帮助更好的生产环境中部署该镜像。

- 更轻松的迁移

    由于Docker确保了执行环境的一致性，使得应用的迁移更加容易。Docker 可以在很多平台上运行，无论是物理机、虚拟机、公有云、私有云，甚至是笔记本，其运行结果是一致的。因此用户可以很轻易的将在一个平台上运行的应用，迁移到另一个平台上，而不用担心运行环境的变化导致应用无法正常运行的情况。

- 更轻松的维护和扩展

    Docker使用的分层存储以及镜像的技术，使得应用重复部分的复用更为容易，也使得应用的维护更新更加简单，基于基础镜像进一步扩展镜像也变得非常简单。此外，Docker 团队同各个开源项目团队一起维护了一大批高质量的官方镜像，既可以直接在生产环境使用，又可以作为基础进一步定制，大大的降低了应用服务的镜像制作成本。

### 无状态的应用程序设计
微服务架构的创建者倾向于在任何可能的情况下使用无状态的服务、而不是有状态的服务。无状态应用程序设计的主要优点在于：**它能够平稳地应对为服务添加或移除某些实例的场景，而无需对应用程序进行重大的变更或进行配置的改动**。比方说，如果服务的负载产生了突发性的增长，可以为服务加入更多无状态的web服务器，而如果某个无状态的服务器挂机了，也可以方便地用另外一台服务器取代它。因此，无状态的服务更容易实现敏捷性和适应性。

### 安装使用

#### docker安装

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

使用阿里云加速镜像&更新镜像存储位置
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
"graph": "/mnt/docker",
"registry-mirrors": ["https://emst96p0.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```




