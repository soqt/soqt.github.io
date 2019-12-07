---
layout: post
title: Docker容器入门及实践
categories: [docker, docker swarm]
description: 根据Udemy上Docker Mastery by Bret Fisher的内容记录的一些笔记
keywords: docker, swarm
date: 2018-10-04 00:00:00
hidden: true
---
Docker学习笔记

<!-- more -->

#### 2019年3月更新
* 更新docker swarm及secret

为什么要用Docker
-------------------
Docker是一个轻量级的虚拟系统，我们叫它容器。因不同系统和版本的不同，部署服务器的时候总是会出现不同的错误，让开发效率大大降低。docker的出现让服务器开发不再受限于系统版本，让一套代码永远可以在不同服务器上一致运行。同时docker也是微服务架构中不可缺少的部分，让不同微服务之间协调效率高效。

第一个Docker Container
--------------------

## Docker安装
Linux可以通过[get.docker.com](https://get.docker.com/)快捷安装。
复制文档前面注释中的代码脚步即可

安装完成后，这是几个常用的CLI(command line interface)命令
```
systemctl start docker   // 启动docker服务
systemctl stop docker    // 停止docker服务
systemctl restart docker // 重启docker服务
systemctl status docker  // 查看docker服务状态
systemctl enable docker  // 开机启动docker服务
systemctl disable docker // 取消开机启动docker服务
```

在运行`systemctl start docker`后，可以试一下`docker container run  hello-world`，之后会在命令栏中print出来行 hello-world即代表安装成功


## 制作一个Nginx的容器
每一个容器都相当于一个虚拟系统
```
docker container run --publish 80:80 --detach --name webhost nginx
```
这行命令的运行流程：
1. 从Dokcer hub下载nginx的镜像(image)
2. 创建新的名为webhost的container
3. `--publish 80:80`为开放容器的80接口
4. 将来自host的80接口网络请求路由至80容器接口
5. --detach 让这个container在后台运行

容器部署成功后，运行下面的命令会显示本地活跃docker实例, 添加 -a会显示全部实例(包括已经停止的实例)
```
docker container ls
```
停止一个容器
```
docker container stop container-name
```
查看log
```
docker container logs container-name
```
删除container。名称或id可以叠加用于删除多项container
```
docker rm container-name
```

## Image镜像 vs Container容器
Container是Image的实例

可以理解为Image是一个class类，container是新建的对象

Image是如和新建Container的一个说明书

## Cheat sheet
向container里传递环境参数
``` 
--env or -e 
```
查看container里面的top process
``` 
docker container top container-name
 ```
显示这个container的metadata（配置，网络等）

``` 
docker container insepct
```

显示实时信息（简单的监测）
``` 
docker container stats
```

进入容器交互(interactive)模式(就是进去虚拟系统)
``` 
docker container run -it CONTAINER_NAME bash
```
-t : pseudo-TTY

-i : interactive


## 创建一个ubuntu的容器
``` 
docker container run -it --name ubuntu ubuntu
```

如果退出后再次进入的命令会不一样:
``` 
docker container start -ai ubuntu
```
进入一个正在运行的container的shell（创建了一个多出的process）
``` 
docker container exec -it container-name bash
```

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


Docker Compose
-----------------
* 保存docker run settings
* 使用YAML
* CLI tool

```yml
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


后台运行当前docker compose

``` docker-compose up -d ```

卸载docker compose
> docker-compose down

查看container中的services
>docker compose top

```yml
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



Docker Swarm
---------------
docker swarm 是docker提供的非常易学高效的分布式部署方式

## Swarm集群初始化
```docker swarm init [OPTIONS]```

options:
⋅⋅⋅*--advertise-addr: 多网卡的情况下，指定需要使用的ip
⋅⋅⋅*--listen-addr: 指定监听的 ip 与端口
<!-- ⋅⋅⋅*--availability: 节点的有效性("active"|"pause"|"drain") -->


```docker service``` 相当于docker container run。区别在于这是给orchestration命令，让它放在queue里自动部署
```docker service update```可以更新正在运行的services的一些参数，用于rolling update



## overlay network
同一swarm下容器之间的访问。

```docker network create --driver overlay NETWORK_NAME```

然后用docker service部署在这个NETWORK_NAME网络中即可

## Routing Mesh
Load balances Swarm services across their tasks
所以在公开接口上的请求都会被自动load balance到不同node上.
这个load balancer是在OSI Layer 3(TCP)上的，不是在Layer4(DNS)，并且是stateless
意思是只能在访问IP和port的时候才可以导流，如果一台服务器运行多个server并运行在一个swarm中，则需要在DNS的Layer上创建一个Nginx(stateful load balancers)

在overlay network上，cluster中访问任意一个node的IP都可以得到相同的结果


Docker Stack
-------------
docker compose file for swarm

```docker stack deploy```自动部署services，但deploy不支持build。需要把自己的image build一下并上传到repo中，在stack中换成repo中的image

```yml
version: "3"
services:
  redis:
    image: redis:alpine
    ports:
      - "6379"
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]
  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - 5000:80
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure
  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - 5001:80
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 5
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  frontend:
  backend:

volumes:
  db-data:
```

部署上面的代码：
```docker stack deploy -c docker-stack.yml voteapp```

```docker stack services STACK_NAME```可以查看此stack部署的services情况
```docker stack ps STACK_NAME```可以查看这个stack怎样运行的
基本和compose差不多，但是version要用3或以上.
deploy可以设置部署多个实例，update时的设置之类的。
deploy.placement.constraints可以说设置只部署在manager node上

如果要update整个stack，最好先改stack file然后再运行```docker -c YML_FILE stack deploy```更新stack


Swarm Secrect
-----------
#在service中

Secrect会被加密储存在docker自己的Raft log中，并会分发给所有manager，当manager管理的worker需要secret时分发下去。
所有的secrect都在/run/secrets/的目录中, 作为一个file。

如果在`docker service`中使用环境变量
> docker service create -e ENV_VAR_FILE=/run/secrets/SECRET_NAME SERVICE_NAME

两种secret注入swarm的方法：
1）文件注入:
在当前目录创建包含secret的文件，运行
> docker secret create SECRET_NAME SECRET_FILE.txt

坏处：密码文件在服务器中，非常危险

2）command line注入
> echo "SECRET_NAME" | docker secret create SECRET_NAME -

坏处：如果有人进去root，可以通过bash history查找到明文密码


查看密码
> docker secret inspect SECRET_NAME

#在stack中
stack yml file的version需要大于等于3.1

```yml
version: "3.1"
  services:
    psql:
      image: postgres
      secrets:
        - psql_user
        - psql_password
      enviroment:
        POSTGRES_PASSWORD_FILE: /run/secrets/psql_password
        POSTGRES_USER_FILE: /run/secrets/psql_user
secrets:
  psql_user:
    file: ./psql_user.txt
  psql_password:
    file: ./psql_password.txt
```
stack中secret同样有两种注入方法，一种是用file，第二中是先用command line提前注入
如果用CLI注入，需要用`external:`标签标明secrets来源
```yml
secrets:
  psql_user: 
    external:
  psql_password:
    external:
```

secrets中还可以自定义permission，可以指定某系统用户才能使用secrets

**当deploy完成之后，要及时清理bash history或secret file**


```
TCP port 2376 for secure Docker client communication. This port is required for Docker Machine to work. Docker Machine is used to orchestrate Docker hosts.
TCP port 2377. This port is used for communication between the nodes of a Docker Swarm or cluster. It only needs to be opened on manager nodes.
TCP and UDP port 7946 for communication among nodes (container network discovery).
UDP port 4789 for overlay network traffic (container ingress networking).
```

CentOS7中防火墙默认关闭

查看防火墙状态
> systemctl status firewalld

开启防火墙
> systemctl start firewalld

修改为默认开机启动
> systemctl enable firewalld

【如果】在Manager的node上打开下列接口
```
firewall-cmd --add-port=2376/tcp --permanent
firewall-cmd --add-port=2377/tcp --permanent
firewall-cmd --add-port=7946/tcp --permanent
firewall-cmd --add-port=7946/udp --permanent
firewall-cmd --add-port=4789/udp --permanent
```

【如果】
在worker的node上打开下列接口
```
firewall-cmd --add-port=2376/tcp --permanent
firewall-cmd --add-port=7946/tcp --permanent
firewall-cmd --add-port=7946/udp --permanent
firewall-cmd --add-port=4789/udp --permanent
```

重新加载防火墙
> firewall-cmd --reload

重启Docker
> systemctl restart docker



## Docker 18.09 版本更新

18.09以上的版本提供了ssh到docker的功能，具体方法是 `docker -H ssh://user@server` 然后再输入你想操作的docker指令。

比如运行: 
`docker -H ssh://user@server run -it --rm busybox`

这样我们就可以直接从本地SSH到服务器的docker，并把Secret传进去从而实现目前最安全的secret部署方法。

首先授权给当前用户docker的使用权，我们就不需要每次都敲sudo了
> sudo usermod -aG docker USER_NAME

然后打开terminal通过本地传secrete:
> echo "SECRET_NAME" | docker -H ssh://USER_NAME@YOUR_HOST secret create secret_name -
如果用file的当做secret的话
> docker -H ssh://USER_NAME@YOUR_HOST secret create secret_name.txt

成功之后会打印出secret的ID`xtgwhpfr6cyvqp3gnmeevorws`，也可以ssh进服务器使用`docker secret ls`查看是否存在刚才注入的secret

这样就在服务器中完全不留痕迹的注入了secret。