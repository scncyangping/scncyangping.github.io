---
layout:     post
title:      "CentOS 8 安装 Harbor 2.0"
subtitle:   ""
date:       2020-01-28 12:20:00
author:     "YaPi"
header-img: ""
tags:
    - 运维
---

#### 安装docker及docker-compose

```text

rm -rfv /etc/yum.repos.d/*

curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo

yum install vim bash-completion net-tools gcc -y

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum -y install docker-ce

# 若报错：
# requires containerd.io >= 1.4.1, but none of the providers can be installed
# 去https://download.docker.com/linux找对应版本containerd

wge https://download.docker.com/linux/centos/8/x86_64/stable/Packages/containerd.io-1.4.3-3.1.el8.x86_64.rpm 

yum install containerd.io-1.4.3-3.1.el8.x86_64.rpm
yum -y install docker-ce

```

#### 下载harbor包

```text
# 解压下载的离线包
tar xvf harbor-offline-installer-v2.0.0.tgz

# 复制默认配置文件
cp harbor.yml.tmpl harbor.yml
# 编辑配置文件
vim harbor.yml
# 主要配置
hostname: harbor.scncys.cn # 设置访问地址，可为IP地址

http:
  port: 8001 # 设置暴露端口
  
https:
  port: 1443 # 设置ssh端口
  
https:
  port: 1443
  certificate: /root/ssh/harbor.scncys.cn.crt # ssh
  private_key: /root/ssh/harbor.scncys.cn.key # ssh

harbor_admin_password: xxxxx # admin用户密码，登陆可修改

data_volume: /root/data/harbor # 数据存储路径


log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /root/data/log/harbor  # 日志地址
```

#### 执行安装

```text
./install.sh


# 若需删除
同目录下：
docker-compose down
```



#### 若需自己配置nginx

注释相关https配置

```text
upstream harbor {
    server 159.75.40.92:8001 weight=1;
}
server {
    listen      80;
    listen 443 ssl;
    server_name harbor.scncys.cn;
    ssl_certificate   /root/ssh/harbor.scncys.cn.crt; 
    ssl_certificate_key  /root/ssh/harbor.scncys.cn.key; 
    ssl_session_timeout 5m; 
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
    ssl_prefer_server_ciphers on;
    location / {
        proxy_pass http://harbor;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 20m;    
        client_body_buffer_size 128k; 
        proxy_connect_timeout 90;   
        proxy_read_timeout 90;      
        proxy_buffer_size 4k;       
        proxy_buffers 6 32k;        
        proxy_busy_buffers_size 64k; 
        proxy_temp_file_write_size 64k; 
    }
}
```