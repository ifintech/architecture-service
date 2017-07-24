# 实例
### 集群创建
- 申请虚拟云主机 4核16G
- 初始化主机
- 更新时区同步时间
```shell
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
- 设置dns
```shell
vim /etc/resolv.conf
nameserver 10.0.4.9 //添加到首行
```
- 安装docker
```shell
curl -sSL https://get.daocloud.io/docker | sh
```
- 使用阿里云加速镜像&更新镜像存储位置
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
- init swarm
```bash
docker swarm init --advertise-addr <MANAGER-IP>
```

- 添加进集群 
```bash
docker swarm join \
--token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
192.168.99.100:2377
```

### 创建服务

#### dns服务
```bash
# 自定义内网dns
echo '127.0.0.1 a.com' >> /data1/dns/etc/dnshosts
# 设置配置
docker config create dns-dnshosts /data1/dns/etc/dnshosts
# 启动服务 2个实例 对外提供53端口dns服务
docker service create --name dns \
    -p 53:53/udp \
    --replicas 2 \
    --limit-cpu .5 \
    --limit-memory 128mb \
    --config source=dns-dnshosts,target=/etc/dnshosts \
    ifintech/online-dns
# 重启服务(加入更新配置后)
docker service update dns --force --update-delay 15
```

#### 对外nginx服务
```bash
# 设置配置
docker config create openresty-upstream /data1/openresty/upstream.conf
docker config create openresty-www.conf /data1/openresty/www.conf
# 设置密钥
docker secret create domain.key /data1/openresty/domain.key
docker secret create domain.crt /data1/openresty/domain.crt
# 启动服务 2个实例 对外提供443端口https服务
docker service create --name https-gw \
    -p 443:443 \
    --replicas 2 \
    --config source=openresty-upstream,target=/etc/nginx/upstream.conf \
    --config source=openresty-www.conf,target=/etc/nginx/vhosts/www.conf \
    --secret domain.key \
    --secret domain.crt \
    --limit-cpu 2 \
    --limit-memory 2048mb \
    ifintech/online-openresty
```
#### 内部http消息总线
```bash
# 设置配置
docker config create httpgateway-upstream /data1/openresty/upstream
docker config create httpgateway-www.conf /data1/openresty/www.conf
docker config create httpgateway-service.json /data1/openresty/service.json
# 启动服务 2个实例 对外提供80端口http服务
docker service create --name http-gw \
    -p 443:443 \
    --replicas 2 \
    --config source=httpgateway-upstream,target=/etc/nginx/upstream \
    --config source=httpgateway-www.conf,target=/etc/nginx/vhosts/www.conf \
    --config source=httpgateway-service.json,target=/etc/nginx/service.json \
    --limit-cpu 2 \
    --limit-memory 2048mb \
    ifintech/online-httpgateway
```