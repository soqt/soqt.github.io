---
layout: post
title: Docker学习笔记
categories: [docker]
description: 根据Udemy上Docker Mastery by Bret Fisher的内容记录的一些笔记
keywords: docker
---

根据Udemy上Docker Mastery by Bret Fisher的内容记录的一些笔记

## 第一个Docker Container
```
docker container run --publish 80:80 --detach --name webhost nginx
```

1. 从Dokcer hub下载nginx的Image
2. 创新新的container
3. 开放80接口
4. 将网络请求路由至80 container接口
5. --detach 让这个container在后台运行

```
docker container ls
```

会显示本地活跃docker实例, 添加 -a会显示全部实例

```
docker container stop container-name
```

停止实例
```
docker container logs container-name
```
查看log
```
docker rm container-name
```
删除container。名称或id可以叠加用于删除多项container

## Image vs. Container
Container是Image的实例

## Cheat sheet

``` 
--env or -e 
```

向container里传递环境参数

``` 
docker container top container-name
 ```

这个container里面的top process

``` 
docker container insepct
```

显示这个container的metadata（配置，网络等）

``` 
docker container stats
```

显示实时信息（简单的监测）

``` 
docker container run -it
```

-t : pseudo-TTY

-i : interactive

创建一个交互(interactive)模式的container

## Use ubuntu
``` 
docker container run -it --name ubuntu ubuntu
```

进入可交互ubuntu

退出后从新进入:
``` 
docker container start -ai ubuntu
```

``` 
docker container exec -it container-name bash
```

进入一个正在运行的container的shell（创建了一个多出的process）

## Docker 网络
-p 用来暴露你的网络接口

* 每一个container接入一个私有虚拟网络“bridge”
* 每一个虚拟网络通过NAT防火墙路由出去
* 所有的container都可以在自己的虚拟网络内部交流（不用-p暴露给公网）
* 最好为每一个独立App建立一个自己的虚拟网络（比如给mongo和node单独创建一个虚拟网络）

```
docker container port container-name
```

显示路由

``` 
docker container inspect --format '\{\{ .NetworkSettings.IPAddress\}\}' container-name
```

查询container地址  --format 是filter

![docker-network](../../../../images/posts/docker/docker-network.png) docker network

``` 
docker network ls
```

显示所有网络
birdge是默认网络，连接外网
host是绕过bridge直接连接外网（性能好，安全性低）
none什么都不连接

``` 
docker network inspect container-name
```

查看网络
docker network inspect bridge 可以查看哪些container正在连着bridge。“IPAM”是自动被赋值的IP地址。默认subnet “172.17.0.0/16”

``` 
dokcer network create --drive
```

建立一个网络
--drive 指定一个drive（bridge host none或者第三方dirve）默认bridge

``` 
docker network connet
```

连接一个网络
一个container可以连接到两个network上

``` 
docker network disconnect
```

退出一个网络

如果要让新的container连接到该网络:
``` 
docker container run -d --name new_nginx --network new_network_name nginx
```

## DNS
因为container中的IP是不固定的，所以需要DNS
两个在相同虚拟网络下的container可以默认通过名字互通

``` 
docker container exec -it con2 ping con1
```

其中con1和con2在同一网络下（需要先apt-get update && apt-get install -y inputils-ping）


## DNS Round Robin Test

1. 新建一个虚拟网络
2. 创建两个elasticsearch:2的镜像

``` 
eg: docker container run --name elastic1 -d --network test --network-alias search elasticsearch:1
```

3. 使用--network-alias为两个container标记alias
4. 运行docker container run --rm --net ass centos curl -s search:9200 附加为--net查看同样DNS名称下的两个网络
5. centos curl -s search:9200 --net

## Docker image
``` 
docker pull nginx:latest
```

生产环境下，最好为Image标注一个固定的版本号，不要用latest


``` 
docker history nginx:latest
```

显示全部nginx的历史layer，每一个layer都代表了一次更新，每一层layer共同组成了一个image
共同使用的layer不会被下载，每一个layer有唯一的SHA区分

``` 
docker inspect nginx:latest
```

显示这个image的metadata比如“ContainerConfig”: "ExpposedPorts"说明哪个接口会被期望被开通，"Cmd"显示哪些command在运行时会被运行...

``` 
docker image tag nginx dockerHubName/nginx
```

为image加一个tag,tag不会改变Image ID，如果后面不添加tag（详情下一行）,默认latest
需要加自己dockerhub的tag才可以push上去

``` 
docker image tag nginx dockerHubName/ngnix dockerHubName/nginx:testing
```

为这个Image添加一个testing的tag

如果想让repo是私人的，现在docker hub上创建一个private repo再push

## Dockerfile
> docker build -f some-dockerfile

> FROM: required

选择一个minimal distribution. (debian, centos), 很多工具都不具备

> WORKDIR /etc/nginx

相当于cd

> COPY

复制source code从local

> EVN: 

eg. NGINX_VERSION 1.11.10-jessie
导入环境变量

> RUN:

运行Shell command, 两个command之间可以用&&连接，表示在同一layer
RUN可以有多个

**Docker有自己的log file(stdout, stderr)，所以用Nginx自带的log并不是最理性的解决方案
```
RUN ln -sf /dev/stdout /var/log/nginx/access.log && ln -sf /dev/stderr /var/log/nginx/error.log
```

将nginx的log导入进docker

> EXPOSE:

允许暴露的接口，比如web需要暴露 **EXPOSE: 80 443**
但是这只是允许权限,还是需要用**-p**在host中暴露这些接口

> CMD: []  required 但是可以inherit from FROM image

当container运行的时候运行的命令，Dockerfile中只能存在一个CMD，如果存在多个，最后一个优先级最高

## Build Dockerfile
``` 
dokcer image build -t customnginx .
```

第一次build时间较长，但是所有步骤会被存cache。
修改一行Dockerfile中的文件后，这一行之后的所有步骤都会重新build，所以文件order很重要，把多变的代码放在后面。

## 小节
创建Dockerfile，如果能用offical repo的base image就用official的，如果不能满足要求就去Docker hub看看有没有可靠高的image。都不能满足要求可以自己用minimal distribution创建自己的Dockerfile。
``` 
docker image build -t tag-name .
```

build已创建的Dockerfile并标注tag
```
docker container run -p 80:80 tag-name
```

运行刚刚创建的image
``` 
docker image tag tag-name:additional-tag dockerHubName/tag-name:additional-tag
docker push dockerHubName/tag-name:additional-tag
```

## 数据保存
container是不可更改，稍纵即逝的，不应该用于保存数据。
Docker有两种解决方式：Volumes和Bind Mounts
Volumes是在container外部规定一个区域用来存储数据
Bind Mounts用来加载外部数据。

### Volumes
在Dockerfile中添加Volume规则
> VOLUME /path/to/db
删除container后不会影响Volume，需要多一个步骤将其删除。

```
docker volume ls
```
可以用来查看当前机器创建了多少Volumes
```
docker volume inspect XXX
```
如果在linux机器上，通过Mountpoint地址可以看到数据。Mac和Windows看不到(在linux VM里)

如果需要创建Volume，记得在docker container run的时候添加 -v name:/path/to/db 来定义Volume名称。否则很难区分Volume对应的container

> docker volume create 


## Bind Mounting
将host的文件或目录映射到container的文件或目录。
无法在Dockerfile里写，只能通过```container run -v /Users/username/stuff:/path/container```实现。
```
docker container run -d --name nginx -p 80:80 -v $(pwd):/usr/share/nginx/html nginx
```
将当前目录$(pwd)映射到/usr/share/nginx/html里面，当当前目录变的时候，container里面的文件也会变。


## Docker Compose
* 保存docker run settings
* 使用YAML
* CLI tool

```
version: '3.1'

service:
	servicename: #DNS name inside network
		image:
		command: #replace the default CMD specified by the image
		environment:
		volumes:
		ports:
		  - 80:80
	servicename2:

volumes:

networks:
```
```
docker-compose up -d
```
```
docker-compose down
```
后台运行当前docker compose
```
docker compose top
```
查看container中的services

```
services:
  proxy:
    build:
      context: .
      dockerfile: nginx.Dockerfile
  image: nginx-custom  
  ports:
      - '80:80'
  web:
    image: httpd
    volumes:
      - ./html:/usr/local/apache2/htdocs/
```
dockerfile指向当前目录自定义的dockerfile，这里是一个nginx的自定义image  
第二个service是server，把当前html目录绑定到container里面，所以可以在runtime情况下改变网页文件  
一般情况下会有第三个service作为database  












