# docker 用户组
docker daemon 绑定了一个unix socker.默认unix sockerd 的所以者是root, 所以docker daemon 都是用root 用户运行。为了不用root ,运行docker 命令，创建一个docker组， 并将用户加入该组。

```
sudo groupadd docker
sudo usermod -aG docker  useease
```

# docker registry
docker 仓库搭建
从官方获取镜像

```
docker pull registry
```
官方镜像 仓库存储在 /tmp/data 下。将其挂载出来

```
docker run -d -p 5000:5000 -v /home/useease/data/registry:/tmp/registry --name registry registry

```
*push的时候就挂了。配置认证信息，所以这是一个“不安全”的registry。Docker要求在docker daemon的启动参数里增加--insecure-registry*。

---

*证书使用的是usease.com 的证书，使用自己生成的证书，需要修改daemon 启动参数*

```
docker run -d -p 5000:5000 --name registry     --restart=always -v /var/lib/registry:/var/lib/registry -v /certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/1_useease.com_bundle.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key registry

```

# docker hub 简易前端

```
docker run -d -p 8080:8080 --name web --link registry -e REGISTRY_URL=https://registry:5000/v2 -e REGISTRY_TRUST_ANY_SSL=true -e REGISTRY_BASIC_AUTH="YWRtaW46Y2hhbmdlbWU="            -e REGISTRY_NAME=localhost:5000 hyper/docker-registry-web

```

# docker 本地镜像上传

```
docker tag mongo m.useease.com:6060/mongo
docker push m.useease.com:6060/mongo
```
# docker postus 前端

```
docker run -it -d \
    --name portus \
    --net host \
    --restart=always \
    -v /certs:/certs \
    -v /usr/sbin/update-ca-certificates:/usr/sbin/update-ca-certificates \
    -v /etc/ca-certificates:/etc/ca-certificates \
    --env DB_ADAPTER=mysql2 \
    --env DB_ENCODING=utf8 \
    --env DB_HOST=192.168.0.13 \
    --env DB_PORT=3306 \
    --env DB_USERNAME=portus \
    --env DB_PASSWORD=portus \
    --env DB_DATABASE=portus \
    --env RACK_ENV=production \
    --env RAILS_ENV=production \
    --env PUMA_SSL_KEY=/certs/domain.key\
    --env PUMA_SSL_CRT=/certs/1_useease.com_bundle.crt \
    --env PUMA_PORT=443 \
    --env PUMA_WORKERS=4 \
    --env MACHINE_FQDN=192.168.0.13 \
    --env SECRETS_SECRET_KEY_BASE=secret-goes-here \
    --env SECRETS_ENCRYPTION_PRIVATE_KEY_PATH=/certs/domain.key \
    --env SECRETS_PORTUS_PASSWORD=portuspw \
    h0tbird/portus:v2.0.2-1
```
连接docker daemon 失败，待解决
> http: TLS handshake error from 192.168.0.1:39478: remote error: unknown certificate authority


参考 [轻松搭建Docker Registry运行环境](http://qinghua.github.io/docker-registry/)
[用容器轻松搭建Portus运行环境](http://qinghua.github.io/portus/)
