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
- 使用阿里云加速镜像
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
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

创建dns服务
```bash
echo '127.0.0.1 a.com' >> /data1/dns/etc/dnshosts
docker config create dns-dnshosts /data1/dns/etc/dnshosts
docker service create --name dns \
    --dns=127.0.0.1 \
    --dns=168.63.129.16 \
    -p 53:53/udp \
    --config source=dns-dnshosts,target=/etc/dnshosts \
    ifintech/online-dns
```
