# 实例
### 集群创建
- 申请虚拟云主机 4核16G **centos**系统
- 初始化主机
- 更新时区同步时间

```shell
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

- 设置内网dns

```shell
vim /etc/resolv.conf
nameserver {DNS_IP} #添加到首行
```

- 安装docker

```shell
curl -sSL https://get.daocloud.io/docker | sh #使用中国镜像安装
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

sudo systemctl start docker
sudo docker -v #查看docker版本 判断安装是否成功
```

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

