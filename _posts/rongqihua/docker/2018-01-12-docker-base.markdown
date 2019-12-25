---
layout:     post
title:      "docker基础"
subtitle:   ""
date:       2018-01-12 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - docker
---
### 基础

#### Docker Platform
Docker 提供了一个开发、打包、运行app的平台,把app和底层infrastructure隔离开来

#### Docker Engine

- 后台进程
- Rest API Server
- CLI接口

#### Docker Image 概述

- 文件和meta data 的集合
- 分层的，并且每一层都可以添加改变删除文件，成为一个新的image
- 不同image可以共享相同的layer
- Image 本身是read-only

查看所有images

```
docker image ls
或
docker images
```

##### image 的获取

1. Build from Dockerfile
2. Pull from Registry

#### docker container

- 通过image创建
- 在image layer之上建立一个container layer(可读可写 image只能读)
- 类比面向对象：类 和实例
- image负责app的存储和分发，container负责运行app

查看container

>docker container ls 查看运行中的

>docker container ls -a 查看所有的包括运行完成的

类似 : docker ps 、 docker ps -a

交互式运行

```
docker  run -it 相关image


列举所有的container id
docker container ls -aq


docker container commit == docker commit

docker image build == docker build

```

#### docker file
- FROM
>FROM scratch # 制作base image

>FROM centos # 使用 base image

- LABEL  （有点像注释）
> LABEL maintainer="yangping"

> LABEL version="1.0"

> LABEL description="miaoshu"

- RUN (执行命令)

每run一次都会生成一层

```
RUN yum update && yuum install -y vim\
    python-dev  # 反斜杠换行
```

- WORKDIR (指定文件目录)

```
WORKDIR /test # 如果没有回自动创建test目录
WORKDIR demo
RUN pwd
```
>尽量使用WORKDIR 不要使用RUN cd 尽量使用绝对目录


- ADD and COPY
将当前目录添加到环境中去

```
ADD test.tar.gz / # 添加到跟目录并解压

ADD hello test/ # 添加hello文件到test下

注:
大部分情况下 COPY 优于 ADD
ADD 除了COPY还有额外功能(解压)
添加远程文件/目录使用 curl 或者wget


```

- ENV

```
ENV MYSQL_VERSION 5.6 # 设置常量
RUN apt-get install -y mysql-server = "${MYSQL_VERSION}" && rm -rf /var/lib/apt/lists/* # 引用常量
```

```
Run :执行命令并且创建新的image layer

cmd ：设置容器启动后默认执行的命令和参数

EntryPoint : 设置容器启动时运行的命令
最佳实践：写一个shell脚本作为entrypoint

COPY docker-entrypotin.sh /usr/local/bin

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 27017
CMD ["mongod"]

```

==docker build -t yangping/centos-entrypoint-shell .==


- CMD

```
多个cmd 只有最后一个运行

docker run 加上了 -it 等命令的时候cmd不会执行
```

#### push 自己的image到docker hup

1. 登陆 docker login
2. docker push image名称
3.


#### docker 命令

```
对运行总的容器执行某个命令

docker exec 容器id 命令

进入到容器

docker exec -it 80ae7f46635a /bin/bash

运行容器内部的python

docker exec -it 80ae7f46635a python

删除退出状态的容器

docker rm $(docker ps -aq)

指定容器名称(具有唯一性)

docker run -d --name=demo yangping/flasktest

docker stop demo

docker start demo

显示container 详细信息

docker inspect 容器id

显示日志信息

docker logs 492cb216161e

```


#### 安装stress工具

1. sudo yum install -y epel-release
2. sudo yum install -y stress


#### 可执行程序加上运行参数

```
FROM centos
RUN yum install -y epel-release && yum install -y stress
ENTRYPOINT ["/usr/bin/stress"]
CMD []


一个空的CMD 可以接受需要传入的参数

```

#### 对容器进行资源限制

docker run  -memory

docker run --memory=200M yangping/centos-stress


设置容器优先级  获取cpu的时候按照设置的权重进行获取

docker run -cpu-shares=10 yangping/cento..



### Docker Compose

#### 构建wordpress

1. docker run -d --name mysql -v mysql-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=wordpress mysql

2. docker run -d -e WORDRESS_DB_HOST=mysql:3306 --link mysql -p 8080:80 wordpress

#### 什么是docker compose

- Docker Compose 是一个工具
- 这个工具通过一个yml文件定义多容器的docker应用
- 通过一条命令就可以根据yml文件的定义去创建或者管理这多个容器

#### docker-compose.yml

##### 三大概念
1. services
2. networks
3. volumes

##### service

一个service代表一个container

service的启动类似docker run 指定network 和 volume

```
第一种 从dockerhub上拉取
services:
    db:
      image:postgres:9.4
      volumes:
       - "db-data:/var/lib/postgresql/data"
      networks:
       - back-tier

第二种 本地build
services:
    db:
      build: ./worker
      links:
        - db
        - redis
      networks:
       - back-tier
```

例子

```
version: '3'

services:

  wordpress:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD: root
    networks:
      - my-bridge # 设置网络为下面自定义的网络

  mysql:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - my-bridge

volumes:
  mysql-data:

networks:
  my-bridge:
    driver: bridge
```


#### docker-compose 安装
- mac windows 安装docker的时候自动安装好了

- linux
  1. sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  2. sudo chmod +x /usr/local/bin/docker-compose
  3. docker-compose --version



#### docker-compose 命令

指定启动的yml文件并启动

docker-compose -f docker-test.yml up

不加 -d 会打印log  加上-d 后台执行

查看启动的service

docker-compose ps

停止docker-compose启动的容器
docker-compose stop 关闭
docker-compose start 启动

docker-compose down 关闭并清除container和network

docker-compose images 显示使用的images

docker-compose exec service名字
>docker-compose exec mysql bash

同一个service启动多个服务
docker-compose up --scale web=3


将一个service启动多个，并且绑定的端口为80

利用 haproxy 将80映射到外部8080

后续访问这个服务的时候则会轮训访问每个服务
```
version: "3"

services:

  redis:
    image: redis

  web:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      REDIS_HOST: redis

  lb:
    image: dockercloud/haproxy
    links:
      - web
    ports:
      - 8080:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

```
