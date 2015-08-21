title: 在Docker中部署nodejs应用
date: 2015-08-21 14:33:58
tags:
- docker
- nodejs

---

##介绍 & 安装
关于Docker的介绍与安装，这里就不再细写，请利用搜索引擎，或者参考[What is docker](https://www.docker.com/whatisdocker), [安装手册](https://docs.docker.com/installation/)。
为了方便，我直接在DigitalOcrean部署了一个docker环境。

##获取应用代码
准备一个应用，这里我用了一个简单的express应用，并侦听4000端口。https://github.com/NeilQ/cTemplate。
将代码获取到本地目录，并进入。
```shell
git clone https://github.com/NeilQ/cTemplate
cd cTemplate
```

##编译镜像
在项目目录下，能看到一个Dockerfile，这是编译镜像时用到的配置文件，内容是这样的:
```shell
从官方镜像仓库docker hub中获取google/nodejs镜像，是一个配置了nodejs的虚拟环境
FROM google/nodejs

WORKDIR /app
ADD package.json /app/
RUN npm install
ADD . /app

CMD []
ENTRYPOINT ["/nodejs/bin/npm", "start"]
```

假如你的项目中没有这个文件，需要手动将其加入。然后编译一个叫"ctemplate"的docker镜像
```shell
docker build -t ctemplate .
```

编译成功后，你就能在镜像列表中看到ctemplate这个镜像
![docker-nodejs-1](/img/docker-nodejs-1.png)

##启动容器
然后我们就能将镜像载入容器启动。
```shell
sudo docker run --name ctemplate5 -p 80:4000 -d ctemplate
```
参数说明：
* --name ctemplate5 为容器起个名字
* -p 80:4000 将容器内部的4000，端口与外部环境的80端口映射。为了方便我直接映射到80端口，但实际上应该映射到其他端口，并通过nginx来做反向代理，统一管理。
* -d 让容器在后台运行

这样，容器就启动起来了，我们可以通过访问http://ip 直接访问站点。
同时，可以用"docker ps"列出正在运行的容器，用"docker logs ctemplate5"查看运行日志等等，详细的命令请查看官方api。
![docker-nodejs-2](/img/docker-nodejs-2.png)

##Docker使用设想
从上面的实践我们可以发现，利用docker进行程序的打包及部署是非常方便的，同时它使用linux container虚拟化技术，性能趋向于外部环境，这意味着它既能做开发、测试环境，也能上生产环境。
对于个人，可以用它来做个人沙箱，假如你有个vps或者云虚拟机，哪天更换厂商做迁移将非常方便，编译成镜像再导入到新的docker环境中就可以了。
对于企业，可以搭建自动化的开发、测试、生成环境，能避免系统环境的差异带来的一些列问题。比如开发说：这个bug在我本地环境不重现，直接将测试环境容器打包丢给开发就行了。比如部署时，devOps只要编译一个配置好的镜像，很容易就能一键部署到各个测试或者生成环境上。

##其他参考资料
[深入浅出Docker（四）：Docker的集成测试部署之道](http://www.infoq.com/cn/articles/docker-integrated-test-and-deployment)
[Docker 从入门到实践 - 标准化开发测试和生产环境](http://dockerpool.com/static/books/docker_practice/cases/environment.html)
