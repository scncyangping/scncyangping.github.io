---
layout:     post
title:      "docker网络 + docker-swarm"
subtitle:   ""
date:       2018-01-12 12:01:00
author:     "YaPi"
header-img: ""
tags:
    - docker
---

### Docker 网络

#### ping telnet
ping 验证IP的可达性

telnet 检查服务的可用性

安装telnet
1. yum install telnet-server
2. yum install telnet

telnet 127.0.0.1 8080

#### Linux 网络命名空间

```
添加

ip netns add test1

查看

ip netns list

删除

ip netns delete

查看信息

ip netns exec test1 ip a （包含端口信息）

ip netns exec test1 ip link (link信息)

启动

ip netns exec test1 ip link set dev lo up

```

#### 通讯模式


- 第一步：

    创建
    ip netns add test1
    ip netns add test2

- 第二步：

    本机添加一对link

    ip link add veth-test1 type veth peer name veth-test2


- 第三步：

    将生成的link添加到前面创建的test1和test2

    ip link set veth-test1 netns test1

    ip link set veth-test2 netns test2

- 第四步：

    为test1 和 test2 添加 ip地址

    ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1

    ip netns exec test2 ip addr add 192.168.1.2/24 dev veth-test2

- 第五步：

    启动test1 和 test2

    ip netns exec test1 ip link set dev veth-test1 up

    ip netns exec test2 ip link set dev  veth-test2 up

- 第六步：

    ip netns exec test1 ping 192.168.0.2

    能够连接成功 这个大概就是docker 容器间通讯的方式


#### docker 网络(bridge host none)

查看网络情况

docker network ls

bridge 为内部使用

host 为外部使用


docker network inspect brige网络id

安装
yum install bridge-utils

brctl show

有一个容器 就会有一个连到docker 的线路

所有的容器都是痛殴docker0进行通讯的

创建连接 (不经常使用)

docker run -d --name=test2 --link test1  busybox /bin/sh -c "while true;do sleep 3600;done"


这里 --link test1 就可以直接使用

#### 让容器之间能够不加link直接ping通

1.创建自动网络集
    docker network create -d bridge my-bridge

2.启动的时候指定

>docker run -d --name=test3 --network my-bridge  busybox /bin/sh -c "while true;do sleep 3600;done"

或者将已经建立了默认的网络连接到自定义的网络

>docker network connect my-bridge test2



#### 端口映射

通过 -p  指定

>docker run --name web -d -p 9069:80 nginx



#### none、host 网络

docker run -d --name test --network none busybox /bin/sh -c "while true;do sleep 3600;done"


和本机完全一样的网络
docker run -d --name test --network host busybox /bin/sh -c "while true;do sleep 3600;done"

#### 启动时指定环境变量 -e
docker run -d --link redis --name flsk-redis -e REDIS_HOST=redis yangping/flask-redis


#### 多机器通信 (etcd)

除bridge、host、none之外的网络 overlay

使用 --net 指定网络



#### Docker 持久化数据


1. 基于本地文件系统的Volume
    1.1 受管理的data Volume 由docker后台自动创建
    1.2 绑定挂载的Volume，挂载位置由用户指定
2. 第三方存储 如aws


Volume 使用

在dockerfile 中 使用关键字 volume 指定本地映射路径


查看所有的volume

docker volume ls

创建的volume的名字不太友好，需指定

-v 自定义名称:volume映射的本地路径

```
docker run -d -v mysql:/var/lib/mysql --name mysql2 -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql
```

把当前容器删除掉，后续创建新的容器 -v 指定的和上述相同，则会数据共享

Bind Mouting(不需要在dockerfile指定volume)

docker run -v /home/aaa:/root/aaa


### Docker Swarm

#### docker swarm

- 创建节点
    1. 创建maneger
    docker swarm init --advertise-addr=192.168.0.2

    2. 会生成一个token，在需要设置为worker的节点上执行 docker swarm join -token SWMTKEN-1-3xxxx 192.168.0.0:2377


- 查看当前swarm的节点  docker node  ls

- 创建service
```
docker service create --name demo busybox sh -c "while true;do sleep 3600;done"

查看service

docker service ls

docker service ps demo


横向扩展

docker service scale demo=5

会在节点中分配并启动5个服务，若某一服务挂掉了，会重新启动一个


删除

docker service rm demo

```

==只有manager节点才能用 docker service 的命令==

#### 构建 wordpress

- 创建overlay网络
    1.docker network create -d overlay demo
- 创建mysql服务
    >docker service create --name mysql --env MYSQL_ROOT_PASSWORD=root --env MYSQL_DATABASE=wordpress --network demo --mount type=volume,source=mysql-data,destination=/var/lib/mysql mysql

- 创建wordpress服务
    >docker service create --name wordpress -p 80:80 --env WORDPRESS_DB_PASSWORD=root --env WORDPRESS_DB_HOST=mysql --network demo wordpress

创建过后 访问manager或节点的ip都能访问到服务

#### docker swarm 网络

- 创建一个返回服务ip的服务
    > docker service create --name whoami -p 8000:8000 --network demo -d jwilder/whoami

- 查看DNS    nslookup www.baidu.com



#### 类似docker-compose部署

配置文件

```
 version: '3'  # order 必须要3.4以上

services:

  web:
    image: wordpress
    ports:
      - 8080:80
    secrets:
      - my-pw
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/my-pw
    networks:
      - my-network
    depends_on: # 依赖于某个服务，需要先创建此服务
      - mysql
    deploy:
      mode: replicated  #可扩展
      replicas: 3  # 启动3个服务
      restart_policy: # 重启的规则
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config: # 更新的规则
        parallelism: 1
        delay: 10s

  mysql:
    image: mysql
    secrets:
      - my-pw
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/my-pw
      MYSQL_DATABASE: wordpress
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - my-network
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

volumes:
  mysql-data:

networks:
  my-network:
    driver: overlay

# secrets:
#   my-pw:
#    file: ./password
```

docker stack 命令

- deploy
>docker stack deploy 服务名称 --compose-file=文件名称
- ls
- ps
- rm
- services


#### docker secret

应用docker manager 节点的分布式存储里面

- 存在swarm manager 节点的Raft database 里面
- secret 可以assign给一个service，这个service就能看到这个secret
- 在container内部secret看起来就像文件，但是实际在内存中


- 创建 docker secret create
    1. 从文件中创建
        >docker secret create my-pw 文件名

    2. 标准输入中创建
        >echo 'adminadmin' | docker secret create my-pw2 -

- 查看 docker secret ls
- 删除 docker secret rm 名称


-使用 启动service的时候指定 --secret mypwd
    >进入到节点中在 /run/secrets里面就会有这个文件，内容就是密码