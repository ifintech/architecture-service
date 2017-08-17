# 镜像仓库

>  当我们在服务器上部署镜像，我们就需要一个集中的存储、分发镜像的服务，镜像仓库就是这样的服务。
>
>  我们把公共的应用或组件的镜像存放于公开的镜像仓库服务中(如dockerhub)，而业务相关的镜像则存放于私有的镜像仓库服务中。对于私有的镜像仓库服务，线上采用docker-registry，用来存储生产镜像；线下采用gitlab内置的registry服务，用来存储开发镜像。

## Registry

> **docker registry**是官方提供的工具，可以用于构建私有的镜像仓库服务。

### 搭建

1. 拉取镜像

   ```shell
   docker pull registry
   ```

2. 配置访问认证

  ```shell
  mkdir /etc/docker/auth
  docker run --rm -t --entrypoint htpasswd registry -Bbn [用户名] [密码] > /etc/docker/auth/htpasswd
  ```

3. 启动服务

   ```shell
   docker run -d --restart=always --name dockerhub \
   -v /mnt/dockerhub:/var/lib/registry \
   -v /etc/docker/auth:/auth \
   -e REGISTRY_AUTH=htpasswd \
   -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
   -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
   -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
   -p 5000:5000 \
   registry
   ```

   - `/mnt/dockerhub`  镜像存储的本地路径

4. 配置请求转发

   nginx 负责向外提供https的接口, 并转发请求到registry服务上

   ```nginx
   # 配置示例
   server {
       listen       443;
       server_name  REGISTRY_HOST;

       access_log  /var/log/nginx/dockerhub.access.log  main;
       error_log   /var/log/nginx/dockerhub.error.log;

       location / {
           proxy_pass http://[REGISTRY_IP]:5000;
       }
   }
   ```

   如果服务器开启了SELinux, 则需要允许通过网络连接到*httpd*服务

   ```shell
   setsebool -P httpd_can_network_connect 1
   ```

5. 验证

   访问`https://[REGISTRYHOST]/v2`  状态码为200即安装成功

### 使用

- 推送镜像
  ```shell
  docker tag [IMAGE_ID] [REGISTRY_HOST/NAME:TAG]
  docker push [REGISTRY_HOST/NAME:TAG]
  ```
- 拉取镜像
  ```shell
  docker pull [REGISTRY_HOST/NAME:TAG]
  ```
- 登录私有仓库
  ```shell
  docker login [REGISTRY_HOST] -u [用户名] -p [密码]
  ```
- 登出私有仓库
  ```shell
  docker logout [REGISTRY_HOST]
  ```
- 查看镜像目录
  访问 `https://[REGISTRY_HOST]/v2/__catalog`

### 参考资料

> 官方文档 [Docker Registry](https://docs.docker.com/registry/)