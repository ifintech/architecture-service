# 服务注册/发现
> 在一个微服务应用中，一组运行的服务实例是动态变化的，实例有动态分配的网络地址，因此，为了使得客户端能够向服务发起请求，必须要要有服务发现机制。

> 服务发现的关键是服务注册中心，服务注册中心是可用服务实例的数据库，它提供了管理和查询使用的API。服务实例使用这些管理API进行服务的注册和注销，系统组件使用查询API来发现可用的服务实例。

### 为什么使用服务发现

设想下，我们写了一些通过REST API或者Thrift API调用某个服务的代码，为了发起这个请求，你的代码需要知道服务实例的网络地址(IP 地址和端口号）。在传统运行在物理机器上的应用中，某个服务实例的网络地址一般是静态的，比如，代码可以从只会偶尔更新的配置文件中读取网络地址。然而在现在流行的基于云平台的微服务应用中，有更多如下图所示的困难问题需要去解决：

![](http://upload-images.jianshu.io/upload_images/3912920-4742d0f9ff9bdeb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

服务实例需要动态分配网络地址，而且一组服务实例可能会因为自动扩展、失败或者升级发生动态变化，因此 你的客户端代码应该使用更加精细的服务发现机制。

有两种主要的服务发现机制：**客户端发现**和**服务端发现**。

### 客户端发现模式

当我们使用客户端发现的时候，客户端负责决定可用服务实例的网络地址并且在集群中对请求负载均衡, 客户端访问服务登记表，也就是一个可用服务的数据库，然后客户端使用一种负载均衡算法选择一个可用的服务实例然后发起请求。

![](http://upload-images.jianshu.io/upload_images/3912920-76cc7f3f5107c3af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

服务实例的网络地址在服务启动的时候被登记到服务注册表中，当实例终止服务时从服务注册表中移除。服务实例的注册一般是通过心跳机制阶段性的进行刷新。

客户端发现机制有诸多**优势和劣势**：该模式除了服务注册表之外没有其他的活动部分了，相对来说还是简单直接的，而且由于客户端知道相关的可用服务实例，那么就可以使用更加智能的，特定于应用的负载均衡机制，比如一致性哈希。一个明显的缺点是它把客户端与服务注册表紧耦合了，你必须为每一种消费服务的客户端对应的编程语言和框架实现服务发现逻辑。


### *服务端发现模式* （在使用）

服务发现的另一种模式就是服务端发现模式。下图展示了该模式的结构：

![](http://upload-images.jianshu.io/upload_images/3912920-76dce8ab07216514.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


客户端通过一个负载均衡器向服务发送请求，负载均衡器查询服务注册表并把请求路由到一台可用的服务实例上。和客户端发现一样，服务实例通过服务注册表进行服务的注册和注销。

类似NGINX或者openresty(我们就是选择了openresty改造成了[httpgateway](ipc/rest.md))这些HTTP服务器和负载均衡器可以作为服务端发现负载均衡来使用。使用ConsulTemplate动态重配置NGINX反向代理，ConsulTemplate是一种根据存储在Consul服务注册表的配置数据阶段性重新生成任意配置文件的工具，每当文件发生变化时，它将运行任意的Shell命令。参考[httpgateway](ipc/rest.md)的实现

服务端发现模式有一些优势也有一些劣势：一个巨大的优势是，服务发现的细节对客户端来说是抽象的，客户端仅需向负载均衡器发送请求即可。这种方式减少了为消费服务的不同编程语言与框架实现服务发现逻辑的麻烦。当然，正如前面所述，一些部署环境已经提供了该功能。这种模式也有一些劣势： 除非部署环境已经提供了负载均衡器，否则这又是一个需要额外设置和管理的可高可用的系统组件。


### 服务注册中心

**服务注册中心是服务发现的关键部分**，它是一个包含服务实例网络地址的的数据库。一个服务注册表需要高可用和实时更新，客户端可以缓存从服务注册表获取的网络地址。然而，这样的话缓存的信息最终会过期，客户端不能再根据该信息发现服务实例。因此，服务注册表对集群中的服务实例使用复制协议来维护一致性。

#### 服务注册选项

正如前面提到的那样，服务实例必须使用服务注册表来进行服务的注册和注销，我们有几种方式来处理服务的注册和注销。
- 服务实例自己注册自己也就是self-registration模式。
- 系统的其他组件管理服务实例的注册，也就是 third-party registration 模式。


Self-Registration模式（在使用）

当使用self-registration 模式时，服务实例自己负责通过服务注册表对自己进行注册和注销，另外如果有必要的话，服务实例可以通过发送心跳请求防止注册过期，下图展示了该模式的结构：

![](http://upload-images.jianshu.io/upload_images/3912920-5bd07f6c772a719f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 选型 consul

安装配置consul

consul集群
```shell
sudo -s
wget https://releases.hashicorp.com/consul/0.7.5/consul_0.7.5_linux_amd64.zip
unzip consul_0.7.5_linux_amd64.zip
mkdir /etc/consul.d
mkdir -p /data1/consul
mv consul /bin

nohup consul agent -server -bootstrap-expect=3 -data-dir=/data1/consul -node=sa-consul1 -bind=10.0.4.14 -client=10.0.4.14 -domain=domain -ui >> /data1/consul/run.log 2>&1 &
nohup consul agent -server -bootstrap-expect=3 -data-dir=/data1/consul -node=sa-consul2 -bind=10.0.4.15 -client=10.0.4.15 -join=10.0.4.14 -domain=domain -ui >> /data1/consul/run.log 2>&1 &
nohup consul agent -server -bootstrap-expect=3 -data-dir=/data1/consul -node=sa-consul3 -bind=10.0.4.16 -client=10.0.4.16 -join=10.0.4.14 -domain=domain -ui >> /data1/consul/run.log 2>&1 &
```


client端安装

```shell
sudo -s
wget https://releases.hashicorp.com/consul/0.7.5/consul_0.7.5_linux_amd64.zip
unzip consul_0.7.5_linux_amd64.zip
mkdir /etc/consul.d
mkdir -p /data1/consul
mv consul /bin

nohup consul agent -data-dir=/data1/consul -node=app-contract2 -bind=10.0.0.4 -join=10.0.4.13 -config-file=/etc/consul.d/config.data >> /data1/consul/run.log 2>&1 &
```

consul配置文件 /etc/consul.d/config.data

consul agent -data-dir=/data1/consul -node=app-contract2 -bind=10.0.0.4 -join=10.0.4.14 -config-file=/etc/consul.d/config.data >> /data1/consul/run.log 2>&1 &

```shell
{
  "service": {
    "id": "app-contract2",
    "name": "contract",
    "address": "10.0.0.4",
    "port": 9000,
    "enableTagOverride": false,
    "checks": [
      {
        "id": "fastcgi",
        "name": "php fastcgi tcp 9000",
        "tcp": "localhost:9000",
        "interval": "10s",
        "timeout": "1s",
        "deregister_critical_service_after": "5m"
      }
    ]
  }
}
```

curl --request PUT --data @config.data https://10.0.0.4:8500/v1/agent/service/register

```shell
//通过curl api注册
{
  "ID": "app-contract2",
  "Name": "contract",
  "Address": "10.0.0.4",
  "Port": 9000,
  "EnableTagOverride": false,
  "Check": {
    "DeregisterCriticalServiceAfter": "1m",
    "TCP": "localhost:9000",
    "Interval": "3s",
    "TTL": "15s"
  }
}
```



添加服务自启动 vim /etc/rc.local

```shell
nohup consul agent -data-dir=/data1/consul -node=app-contract2 -bind=10.0.0.4 -join=10.0.4.13 -config-file=/etc/consul.d/config.data >> /data1/consul/run.log 2>&1 &
```


