# 容器镜像

## 基础镜像


## 镜像仓库
> 线上采用docker-registry，线下采用gitlab内置的registry
> **docker-registry**是官方提供的工具，可以用于构建私有的镜像仓库。

### 安装配置

1. 拉取镜像

   ```shell
   docker pull registry
   ```

2. 配置访问认证

  ```shell
  mkdir /etc/docker/auth
  docker run --entrypoint htpasswd registry -Bbn [用户名] [密码] > /etc/docker/auth/htpasswd
  ```

3. 启动服务

   ```shell
   docker run -d --restart=always --name dockerhub \
   -v /tmp:/var/lib/registry \
   -v /etc/docker/auth:/auth \
   -e "REGISTRY_AUTH=htpasswd" \
   -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
   -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
   -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
   -p 5000:5000 \
   registry
   ```

   - /tmp:/var/lib/registry  指定镜像存储的路径

4. 验证

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