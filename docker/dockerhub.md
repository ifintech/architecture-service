# 容器镜像

## 镜像制作
## 镜像仓库
> 线上采用docker-registry，线下采用gitlab内置的registry
> **docker-registry**是官方提供的工具，可以用于构建私有的镜像仓库。

### 安装配置

1. 拉取镜像

   ```shell
   docker pull registry
   ```

2. 配置HTTPS证书

   - **/etc/docker/certs/registry.crt**
   - **/etc/docker/certs/registry.key**

3. 配置访问认证

  ```shell
  mkdir /etc/docker/auth
  docker run --entrypoint htpasswd registry -Bbn [用户名] [密码] > /etc/docker/auth/htpasswd
  ```

4. 修改系统配置

   - 关闭 SELinux

     ```shell
     /usr/sbin/sestatus -v #查看SELinux状态 如果SELinux status参数为enabled即为开启状态
     setenforce 0 #临时关闭（不用重启机器）
     vim /etc/selinux/config #永久关闭 将SELINUX=enforcing改为disabled 重启机器即可
     ```

5. 启动服务

   ```shell
   docker run -d --restart=always --name registry -v /tmp:/var/lib/registry -v /etc/docker/certs:/certs -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key -v /etc/docker/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -p 443:443 registry
   ```

   - /tmp:/var/lib/registry  用来指定存放仓库的本地路径

6. 验证

   访问`https://[REGISTRYHOST]/v2`  状态码为200即安装成功

### 操作

- 推送镜像

  ```shell
  docker tag [IMAGE] [REGISTRYHOST/NAME:TAG]
  docker push [REGISTRYHOST/NAME:TAG]
  ```

- 拉取镜像

  ```shell
  docker pull [REGISTRYHOST/NAME:TAG]
  ```

- 登录私有仓库

  ```shell
  docker login [REGISTRYHOST] -u [用户名] -p [密码]
  ```

- 登出私有仓库

  ```shell
  docker logout [REGISTRYHOST]
  ```

- 查看镜像目录

  访问 `https://[REGISTRYHOST]/v2/__catalog`

### 参考资料

> 官方文档 [Docker Registry](https://docs.docker.com/registry/)