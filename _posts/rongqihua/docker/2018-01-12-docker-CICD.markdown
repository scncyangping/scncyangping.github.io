---
layout:     post
title:      "CI/CD"
subtitle:   ""
date:       2018-01-12 12:03:00
author:     "YaPi"
header-img: ""
tags:
    - docker
---

#### gitlab、gitlab-ci 服务器搭建

略

#### CI/CD

1. 注册runner
>suudo gitlab-ci-multi-runner register

>按要求输入gitlab服务器的地址、tags等信息

2. 增加.gitlab-ci.yml文件

```

stages:
    - style
    - test
    - deploy
pep8:
    stage: style
    script:
        - pip install tox
        - tox -e pep8
    tags:
        - python2.7 # gitlab-ci服务器上创建runner指定的tags

unittest-python27:
    stage: test
    script:
        - pip install tox
        - tox -e py27
    tags:
        - python2.7

# 自动部署 (部署到ci服务器)
docker-deploy:
    stage: deploy
    script:
        - docker build -t flask-demo . # 从当前目录找到Dockerfile文件build一个image
        - docker run -d -p 5000:5000 flask-demo

    tages:
        - demo # 选择一个shell类型的runner

    only: # 只有master分支修改了才会部署
        - master

docker-image-release:
    state: release
    script:
        - docker build -t registry.com/flask-demo:$CI_COMMIT_TAG .
        - docker push registry.com/flask-demo:$CI_COMMIT_TAG
        tags:
          - demo
        only:
          - tags

```


若想实现自动部署

- 需要有gitlab服务器

- 需要有gitlab-ci服务器
    1. 此服务器能够ping通docker-registry服务器
    2. 需要配置
         >/etc/docker/daemon.json
            >添加{"insecure-registries":{"ip:端口"}}
                > service docker restart

- 需要有docker-registry服务器
    > gitlab-ci 选择shell环境时,可以将打包的镜像提交到自己的docker仓库内，后续其他节点可以手动更新
- 并且能够互通


gitlab-ci 的缺点是过于臃肿,只有部署了gitlab-ci runner的节点才能去部署更新项目，安装不是很方便

由此可使用drone + gitlab 。优点是drone的server和clien都是采用容器的方式部署的，比较方便

#### 搭建drone 的服务端和客户端

搭建drone

```
version: '2'
services:
    drone-server:
        image: drone/drone:0.8

        ports:
            - 8000:8000
            - 9000
        volumes:
            - /home/data1/drone/drone-data:/var/lib/drone/
        restart: always
        environment:
            - DRONE_OPEN=true
            - DRONE_GITLAB=true
            - DRONE_GITLAB_CLIENT=3b026a57ef85bd05181a670ad32ded6b0d58e711e81c79864fd4801569e1c580 # 这个在Gitlab UserSettings/Application 中生成
            - DRONE_GITLAB_SECRET=2bb590a29eb0de0ad345811da5253527d33c2c166ecf8bff17480396c8e7dff5  # 这个在Gitlab UserSettings/Application 中生成
            - DRONE_GITLAB_URL=http://gitlab.dev # gitlab地址
            - DRONE_HOST=http://drone.dev # drone地址
            - DRONE_SECRET=123456

    drone-agent: # 可与server分开部署
        image: drone/agent:0.8

        restart: always
        depends_on:
            - drone-server
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        environment:
            - DRONE_SERVER=drone-server:9000
            - DRONE_SECRET=123456
```

然后在gitlab项目目录中添加 drone.yml文件,则会自动进行其他操作

test

```
clone: # 设置克隆地址，不设置默认为本地
  git:
    image: plugins/git
    recursive: true # 是否递归查找
    skip_verify: true # 跳过https检测
    submodule_override: # 指定项目url 本地环境可用
      helloworld: http://192.168.0.87/demo/helloworld.git

pipeline:
  build:
    image: golang
    commands:
      # - go get
      - go build
      - go test

```


7moor

```
workspace: # 指定pipeline的工作目录,相对于镜像来说
  base: /nodejs
  path: src/wcj-server
pipeline:


#  build:
#    image: registry.7moor.com/base/node:8.11
#    commands:
#      - tsc
  publish:
    image: plugins/docker
    dockerfile: Dockerfile
    username: 7m_docker
    password: wjU6k4iuuT
    repo: registry.7moor.com/wyx/wcj-server # 仓库下项目的名称
    registry: registry.7moor.com # 自建docker仓库地址
    tags: "${DRONE_BRANCH}"
    when:
      branch: master
      event: [push, pull_request]
  deploy:
      image: appleboy/drone-ssh
      host: 123.206.78.95
      username: dockerPull
      password: YjVjNjEzNTQzMzg0ZDc3MGI3Zjc3
      port: 65022
      command_timeout: 600
      script:
        - sudo docker login -u 7m_docker -p wjU6k4iuuT open-registry.7moor.com
        - sudo docker pull open-registry.7moor.com/wyx/wcj-server:${DRONE_BRANCH}
        - sudo docker stop wcj-server
        - sudo docker rm wcj-server
        - sudo docker run -d  -p 3000:3000 -v /opt/wyx/wcj-server/logs:/workspace/logs -v /tmp/app:/tmp/app  -v /opt/wyx/wcj-server/conf:/workspace/conf --name wcj-server open-registry.7moor.com/wyx/wcj-server:${DRONE_BRANCH}

```


基本流程

项目流程为：


- git push 提交代码到版本控制系统（GitHub、GitLab、Gogs 等）
- 版本控制系统通过 webhook 触发 Drone 的 pipeline
- Drone 执行 pipeline，build 构建项目
- 构建 Docker 镜像（需要 Dockerfile 文件）
- 将镜像 push 到 registry（Harbor 等）


#### drone.yml 文件基本愈发
https://blog.csdn.net/kikajack/article/details/80503786#%E5%8C%85%E5%90%AB%E8%B7%B3%E8%BF%87%E5%88%86%E6%94%AF-branches

- branches
    1. 使用branches可以跳过或者指定某些分支进行pipline
    >
    ```
    跳过
    branches:
        exclude: [ develop, feature/* ]

    包含

    branches:
         include: [ master, feature/* ]

    匹配
    branches: [ master, develop ]
    ```

- pipelines
>pipelines 定义了工作流（或叫流水线），包含代码构建，代码测试和代码部署等一系列步骤。工作流根据各个步骤的定义位置按顺序执行。如果一个步骤返回了非 0 的退出代码，工作流将立即停止并返回一个错误状态。


```
定义两个工作流

pipeline:
  backend:
    image: golang
    commands: # 这些命令会在docker创建时执行类似entrypoint
      - go build
      - go test
  frontend:
    image: node
    commands:
      - npm install
      - npm run test
      - npm run build
```

- 条件执行 when
    ```
    pipeline:
  slack:
    image: plugins/slack
    channel: dev
+   when:
+     branch: master
    ```



#### drone 报错

- 如果是本地搭建服务器需要用管理员登陆gitlab服务器

```
 gitlab 10.6 版本以后为了安全，不允许向本地网络发送webhook请求，如果想向本地网络发送webhook请求，则需要使用管理员帐号登录，默认管理员帐号是admin@example.com，密码就是你gitlab搭建好之后第一次输入的密码，登录之后， 点击Configure Gitlab

即可进入Admin area，在Admin area中，在settings标签下面，找到OutBound Request，勾选上Allow requests to the local network from hooks and services ，保存更改即可解决问题

```


#### 重要错误提示！！！！

若修改了gitlab地址,则相应的配置drone服务器的配置文件中的volume地址也需要改变，否则，git拉取代码的地址还是以前的地址！切记切记！


#### drone server client分开安装

https://mritd.me/2018/03/30/set-up-drone-ci/