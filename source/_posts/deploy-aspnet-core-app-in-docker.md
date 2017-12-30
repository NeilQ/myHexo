title: Docker中部署aspnet core应用
date: 2017-11-17 15:24:52
tags:
- docker
- dotnetcore
categories:
- dotnetcore
---

准备将asp.net core 2.0应用部署到docker中，将过程予以记录。Docker的介绍和安装不再赘述，我所用平台是Ubuntu 16.04。附[安装手册](https://docs.docker.com/engine/installation/)。

## 准备应用代码
首先我们需要准备应用代码及Dockerfile，并且确保`dotnet run`可运行，示例代码结构为：
```
- Myapp
    - Myapp.Core
    - Myapp.Api
    - Myapp.sln
    - Dockerfile
```

其中Dockerfile：

```Dockerfile
FROM microsoft/aspnetcore-build:2.0 AS build-env
WORKDIR /app

# copy everything else and build
COPY ./ ./
RUN dotnet restore
RUN dotnet publish -c Release -o out

# build runtime image
FROM microsoft/aspnetcore:2.0
WORKDIR /app
COPY --from=build-env /app/Myapp.Api/out .
RUN mkdir logs
RUN mkdir pictures
ENTRYPOINT ["dotnet", "Myapp.Api.dll"]
```

更多的示例，可以在[dotnet官方示例](https://github.com/dotnet/dotnet-docker-samples)找到。

## 修改Docker镜像地址
由于某些不可描述的原因，我们是无法访问Dockerhub的，导致镜像无法下载，又或者你有个私有镜像地址，需要重新注册一个镜像地址，执行命令：
```shell
$ dockerd --registry-mirror=https://registry.docker-cn.com
```

当然，我们也可以直接修改配置文件，位于`/etc/docker/daemon.json`:
```json
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

## 编译镜像并启动容器
此时，我们可以编译代码镜像：
```shell
$ docker build -t myapp .
```

接着运行：
```shell
$ docker run -dit --restart always -p 9999:80 -v /mnt/pictures:/pictures -v /mnt/logs:/logs --name myapp-prod parkingapi
```

运行参数说明：
- `-dit`: 在后台运行并且进入容器终端bash。这里也可以用`-d`(在后台运行，不进入容器终端), `-it`(进入容器终端bash，但不在后台运行)。
- `--restart always`: 设置容器自动启动。
- `-p 9999:80`: 将host端口9999映射到容器端口80。
- `-v /mnt/pictures:/pictures`: 将host目录`/mnt/pictures`映射到容器目录`/pictures`。
- `--name myapp-prod`: 容器命名。

此时我们可以看到容器启动，app将可以访问。执行`docker ps`可以查看当前容器状态。

## Docker开机自启动
目前大部分linux发行版都使用`systemd`管理服务启动，运行以下命令以启用Docker开机自动启动：
```shell
$ sudo systemctl enable docker
```

假如你需要禁用，可以运行：
```shell
$ sudo systemctl disable docker
```

## 清理本地镜像
运行命令`docker images`可以查看本地所有镜像
```shell
$ docker images

myapp                        latest              b4ceb4115da4        2 hours ago         306MB
<none>                       <none>              9b92ef1cfa33        2 hours ago         2.01GB
<none>                       <none>              583a1c14d487        2 hours ago         306MB
<none>                       <none>              a1dd5cd624ec        2 hours ago         2.01GB
<none>                       <none>              00deb9fb4e39        3 hours ago         306MB
<none>                       <none>              23219b3ff84b        3 hours ago         2.01GB
<none>                       <none>              560e20429df8        3 hours ago         2.01GB
<none>                       <none>              6927faf3fbff        3 hours ago         2.01GB
<none>                       <none>              4f8320a8afc3        3 hours ago         1.97GB
<none>                       <none>              b2738d8f6285        3 hours ago         1.96GB
microsoft/aspnetcore-build   2.0                 f07b15f2852c        24 hours ago        1.9GB
microsoft/aspnetcore         2.0                 757f574feed9        24 hours ago        299MB
```
多次实验过程中，会产生一些缓存镜像，过滤查看这些镜像
```shell
$ docker images --filter "dangling=true"

<none>                       <none>              9b92ef1cfa33        2 hours ago         2.01GB
<none>                       <none>              583a1c14d487        2 hours ago         306MB
<none>                       <none>              a1dd5cd624ec        2 hours ago         2.01GB
<none>                       <none>              00deb9fb4e39        3 hours ago         306MB
<none>                       <none>              23219b3ff84b        3 hours ago         2.01GB
<none>                       <none>              560e20429df8        3 hours ago         2.01GB
<none>                       <none>              6927faf3fbff        3 hours ago         2.01GB
<none>                       <none>              4f8320a8afc3        3 hours ago         1.97GB
<none>                       <none>              b2738d8f6285        3 hours ago         1.96GB
```

我们可以批量清理以释放一些硬盘空间：
```shell
$ docker rmi $(docker images -f "dangling=true" -q)
```

✎。
