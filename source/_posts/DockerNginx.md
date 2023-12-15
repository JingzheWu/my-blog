# 使用 Docker 安装 Nginx

```bash
docker pull nginx:latest
```

```bash
docker run --name nginx -p 8080:80 -d nginx
```

```bash
docker exec -it nginx bash
cat /etc/nginx/nginx.conf
exit
```

```bash
mkdir -p /opt/docker/nginx/conf
```

```bash
docker cp nginx:/etc/nginx/nginx.conf /opt/docker/nginx/conf/nginx.conf
docker cp nginx:/etc/nginx/conf.d/ /opt/docker/nginx/conf/conf.d
docker cp nginx:/var/log/nginx/ /opt/docker/nginx/logs
docker cp nginx:/usr/share/nginx/html/ /opt/docker/nginx/html
```

```bash
docker stop nginx
docker rm nginx
docker ps -a
```

```bash
docker run \
-p 80:80 \
--name nginx \
-v /opt/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /opt/docker/nginx/conf/conf.d:/etc/nginx/conf.d \
-v /opt/docker/nginx/logs:/var/log/nginx \
-v /opt/docker/nginx/html:/usr/share/nginx/html \
-d nginx
```

```bash
# 下面这个是有问题的，这样设置里之后，访问 `http://localhost:80` 依然可以访问到 nginx 的默认页面
# 因为这个命名使用了 `--net host` 选项，这个选项会让容器使用宿主机的网络，所以 Docker 中的 Nginx 依然会监听宿主机的 80 端口

# 如果是在云服务器上使用 Docker 安装 Nginx，那么下面的命令会导致，使用 8080 端口也可能访问呢不到 Nginx 的默认页面，因为一般情况下云服务器默认的防火墙会关闭 8080 端口。

docker run \
-p 8080:80 \
--name nginx \
--net host \
-v /opt/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /opt/docker/nginx/conf/conf.d:/etc/nginx/conf.d \
-v /opt/docker/nginx/logs:/var/log/nginx \
-v /opt/docker/nginx/html:/usr/share/nginx/html \
-d nginx
```

参考资料：

- [完整详细使用Docker安装Nginx教程](https://juejin.cn/post/7176299143659257893)
- [docker+nginx 安装部署修改资源目录配置文件和容器端口信息](https://www.cnblogs.com/jimojianghu/p/15932500.html)
- [2023 docker nginx安装教程(含portainer教程)](https://blog.csdn.net/weixin_44248903/article/details/134803724)
- [Docker 安装 nginx 并且配置反向代理遇到的坑](https://blog.csdn.net/m0_49194578/article/details/117341481)
