# API Gateway

> API网关就是一个应用的单一入口，它还负责路由请求、组合、协议转换等工作，它为每个应用的客户端提供定制化的API，它也可以通过返回默认值或缓存值来处理失败

## 理论

> [https://www.nginx.com/blog/building-microservices-using-an-api-gateway/?utm\_source=introduction-to-microservices&utm\_medium=blog&utm\_campaign=Microservices](https://www.nginx.com/blog/building-microservices-using-an-api-gateway/?utm_source=introduction-to-microservices&utm_medium=blog&utm_campaign=Microservices)

当我们选择把应用构建成一组微服务的时候，我们需要决定应用的客户端如何与这些微服务进行交互。传统单体应用中，往往只是一组（一般是replicated，负载均衡）的节点，而在微服务架构中，每个微服务都会暴露一组细粒度的节点。我们将检验这种方式如何影响客户端与应用端的通信，并且提出使用API网关的方式来解决这个问题。

#### 客户端直接与微服务通信

理论上客户端可以直接与每一个微服务进行通信，每个微服务将会有一个公开的节点 `https://serviceName.api.company.name`，这个URL将会映射到负载均衡器，然后被分发到可用的实例上被处理，为了获取产品明细，移动客户端需要向上面列出的各个微服务发送请求。

非常不幸的是，这种方案有诸多挑战和限制

* 问题之一就是客户端与每个微服务暴露出的细粒度API之间的不匹配，本例子中的客户端需发送七个不同的请求，在一个更加复杂的应用中请求数可能更多，比如亚马逊在渲染产品页的时候可能要调用上百个服务来渲染页面，一个客户端可以在LAN中发送多个请求，但是在公网上就特别低效，那就不用提在移动设备上了，当然，这种方式也使得客户端异常的复杂。

* 客户端直接调用微服务的另一个问题是，一些服务可能使用对web并不友好的协议实现。一个服务可能使用Thrift二进制的RPC而另一个服务可能使用AMQP消息协议。这些协议都不是浏览器和防火墙友好的，最好是在应用内部被使用。防火墙之外呢，应用最好使用HTTP或者WebSocket。

这种方式另一个劣势是使得微服务重构变得困难，随着时间推移，我们可能需要重新划分、组织微服务，比如我们可能合并两个微服务，也可能把某微服务拆分为两个或多个，如果客户端直接与微服务交互的话，对这些微服务进行重构变得异常困难。

正是由于这些问题，采用客户端直接调用微服务的方式并不明智。

#### 使用API网关

通常更好的方式是使用大家都熟知的API网关，API网关是提供系统唯一入口的一台服务器，它和面向对象设计模式中的门面类似：API网关封装了内部的系统架构并向每个客户端提供裁剪的API，它也可能负责诸如**用户验证、监控、负载均衡、缓存、请求改造和管理以及静态内容响应**等职责。

下图展示了API网关通常适应的架构：

![](https://cdn.wp.nginx.com/wp-content/uploads/2016/04/Richardson-microservices-part2-3_api-gateway.png)

**API网关负责请求路由、组合以及协议转换。**所有来自客户端的请求都先经过API网关，然后被路由分配到相应的微服务中，API网关通常调用多个微服务并聚合其结果来处理请求，它可以在HTTP或者WebSocket这些web友好协议与内部使用的web不友好协议间相互转换。

**API网关可以为每个客户提供定制化的API。**它通常为移动客户端暴露粗粒度的API，比如提供（/productdetails?productid=xxx）节点使得移动应用单一请求就能获取所有的产品明细。API网关调用产品信息、推荐、评分等服务，组合这些结果来处理客户端请求。

一个非常牛的例子就是Netflix API网关，Netflix 流服务在上百种包含电视、机顶盒、智能手机、游戏系统、平板电脑等设备上都可用。起初Netflix想为它们的流服务提供一种 one‑size‑fits‑all API，然而，他们发现由于设备的不同划分以及独特需求，这样设计是不现实的。现在他们使用API网关通过运行设备相关的适配器代码为客户端提供裁剪的API，适配器通常为每个请求调用平均六到七个后台服务， Netflix API网关现在每天处理上亿请求。

#### 使用API网关的优势与劣势

正如你所想，使用API网关有优势也有劣势。一个巨大优势就是它封装了应用的内部结构，而不是让客户端直接调用每个服务，客户端只需要简单的与网关交互即可，另外API网关为不同客户提供定制的API，并且减少了客户端和应用间的网络调用，这也大大简化了客户端代码实现。

API网关也有一些劣势，它本身是一个新的高可用的组件，需要被开发、部署和管理，同时API网关有可能成为开发的瓶颈。开发者为了暴露新的微服务节点必须更新API网关，把更新网关的流程做的尽量轻量级是很重要的，不然的话，开发者更新网关的时候就要被迫在线等待。尽管它有这些劣势，在实战中，应用使用API网关还是明智的选择！

## 实例

### API网关

> [Kong](https://getkong.org/docs/)是Mashape开源的高性能高可用API网关和API服务管理层。它基于OpenResty，进行API管理，并提供了插件实现API的AOP。

#### 安装部署

使用docker-compose部署

```bash
git clone https://github.com/Mashape/docker-kong
cd docker-kong/compose/
docker-compose up
```

#### kong插件开发

### 初级网关

> 在单体时间或者业务发展初期，我们是不需要api网关的，我们需要的是应用级别的接口管理，同时兼顾纯接口级别
>
> [orange](http://orange.sumory.com/docs/) Orange是一个基于OpenResty的API Gateway，提供API及自定义[规则](http://orange.sumory.com/docs/rule.html)的监控和管理，如访问统计、流量切分、API重定向、API鉴权、WEB防火墙等功能。Orange可用来替代前置机中广泛使用的Nginx/OpenResty， 在应用服务上无痛前置一个功能丰富的网关系统

#### 安装部署

使用docker-compose部署

```bash
git clone git@github.com:syhily/docker-orange.git
cd docker-orange
docker-compose run --service-ports --rm orange
```

使用外部依赖数据库

```bash
docker run -d --name orange \
    --link orange-database:orange-database \
    -p 7777:7777 \
    -p 8888:80 \
    -p 9999:9999 \
    --security-opt seccomp:unconfined \
    -e ORANGE_DATABASE={your_database_name} \
    -e ORANGE_HOST=orange-database \
    -e ORANGE_PORT={your_database_port} \
    -e ORANGE_USER={your_database_user} \
    -e ORANGE_PWD={your_database_password} \
    syhily/orange
```

​

