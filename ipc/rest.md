# 同步请求 RESTFUL

## HTTP 消息总线

假设服务之间直接通讯。每个服务暴露一组REST API，外部的服务或者客户端通过REST API来调用。明显的，这种模型对于简单的微服务架构应用有效。但是随着服务数量的增加，它会慢慢变得复杂。这也是为什么在SOA里面要用ESB来避免杂乱的点对点的连接。让我们试着总结一下点对点模式的弊端。

* 非功能需求，比如用户认证、流控、监控等必须在每个微服务里实现；
* 由于通用功能的重复，每个微服务的实现变得复杂；
* 在服务和客户端之间没有通讯控制（甚至对于监控、跟踪、过滤等都没有）；
* 对于大的微服务实现来说直接的通讯形式通常被认为是[反模式](http://www.infoq.com/articles/seven-uservices-antipatterns)。

因此， 在复杂的微服务应用场景下，不要使用点对点直连或者中央的ESB，我们可以使用一个**轻量级的中央消息总线**给所有微服务提供一个抽象层，而且可以用来实现各种非功能的能力。这种风格也叫做API Gateway风格。

![](http://img.dockerinfo.net/2016/07/20160718114652.jpg)

#### 功能点

1. 授权认证和访问控制
2. 流控、熔断
3. 日志、跟踪

## API设计规范

> [规范地址](https://ifentech.gitbooks.io/rdbuild/content/rule/api.html)

#### header头约定

> x-source 来源应用  
> x-time 时间戳  
> x-m 加密串  
> x-field 参与校验的字段  
> x-rid 请求唯一标识

## 选型

### Orange

> 一个基于OpenResty / Nginx的HTTP API Gateway，通过orange我们可以实现服务及API级别上的日志监控，认证，访问控制，流控等功能。

- [部署orange](../api-gateway.md#Orange)
- 添加代理配置  如将`http://http-gateway/app/*`的请求代理到`http://app/*`的服务上