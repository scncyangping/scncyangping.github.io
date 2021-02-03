---
layout:     post
title:      "CentOS 8 安装 Gitlab 13.8.1"
subtitle:   ""
date:       2020-01-28 12:20:00
author:     "YaPi"
header-img: ""
tags:
    - 运维
---

#### 安装过程
略

#### 配置自有nginx

```text
vim /etc/gitlab/gitlab.rb

# 配置自有nginx访问
external_url 'http://gitlab.scncys.cn'
# 禁用自带nginx
nginx['enable'] = false

# 添加gitlab nginx配置文件

upstream gitlab {
  server unix:/var/opt/gitlab/gitlab-workhorse/sockets/socket fail_timeout=0;
}

server {
  listen *:80;

  server_name gitlab.scncys.cn;   # 请修改为你的域名

  server_tokens off;     # don't show the version number, a security best practice
  root /opt/gitlab/embedded/service/gitlab-rails/public;

  # Increase this if you want to upload large attachments
  # Or if you want to accept large git objects over http
  client_max_body_size 250m;

  # individual nginx logs for this gitlab vhost
  access_log  /var/log/gitlab/gitlab_access.log;
  error_log    /var/log/gitlab/gitlab_error.log;

  location / {
    # serve static files from defined root folder;.
    # @gitlab is a named location for the upstream fallback, see below
    try_files $uri $uri/index.html $uri.html @gitlab;
  }

  # if a file, which is not found in the root folder is requested,
  # then the proxy pass the request to the upsteam (gitlab unicorn)
  location @gitlab {
    # If you use https make sure you disable gzip compression
    # to be safe against BREACH attack

    proxy_read_timeout 300; # Some requests take more than 30 seconds.
    proxy_connect_timeout 300; # Some requests take more than 30 seconds.
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Frame-Options   SAMEORIGIN;

    proxy_pass http://gitlab;
  }

  # Enable gzip compression as per rails guide: http://guides.rubyonrails.org/asset_pipeline.html#gzip-compression
  # WARNING: If you are using relative urls do remove the block below
  # See config/application.rb under "Relative url support" for the list of
  # other files that need to be changed for relative url support
  location ~ ^/(assets)/  {
    root /opt/gitlab/embedded/service/gitlab-rails/public;
    # gzip_static on; # to serve pre-gzipped version
    expires max;
    add_header Cache-Control public;
  }

  error_page 502 /502.html;
}

# 重置配置
sudo gitlab-ctl reconfigure
# 重启
sudo gitlab-ctl restart
```