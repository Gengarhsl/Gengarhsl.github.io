---
categories:
- DevOps
date: 2026-03-12
tags:
- Tailscale
- Nginx
- Dev Gateway
- 微服务
- Nacos
title: Tailscale + Nginx Dev Gateway 部署指南
---

# Tailscale + Nginx Dev Gateway 部署指南

本文介绍如何在公司服务器上部署 **Nginx Dev Gateway**，使远程开发者通过
**Tailscale VPN** 接入公司微服务环境，并通过 **Nacos 注册中心** 与
**Gateway** 正常通信，同时不影响内网用户。

目标：

-   内网服务正常使用
-   远程开发服务可接入微服务体系
-   不需要修改每个服务配置
-   不需要关闭 SELinux

------------------------------------------------------------------------

# 一、整体架构

``` text
开发者电脑
   │
Tailscale (100.x.x.x)
   │
Dev Gateway (Nginx)
192.168.3.200
   │
公司内网
   │
Gateway
   │
Nacos
   │
内网微服务
```

说明：

  组件          作用
  ------------- ------------------
  Tailscale     远程开发网络
  Dev Gateway   统一代理远程服务
  Nacos         服务注册中心
  Gateway       微服务网关

------------------------------------------------------------------------

# 二、安装 Nginx

服务器示例：

    192.168.3.200

## 1 安装 EPEL

``` bash
yum install epel-release -y
```

## 2 安装 Nginx

``` bash
yum install nginx -y
```

## 3 启动服务

``` bash
systemctl start nginx
systemctl enable nginx
```

查看状态：

``` bash
systemctl status nginx
```

------------------------------------------------------------------------

# 三、Nginx 目录说明

常见路径：

  路径                        说明
  --------------------------- ------------
  /etc/nginx/nginx.conf       主配置文件
  /etc/nginx/conf.d/          子配置目录
  /var/log/nginx/access.log   访问日志
  /var/log/nginx/error.log    错误日志

建议：

    /etc/nginx/conf.d/dev/

专门存放远程开发代理配置。

------------------------------------------------------------------------

# 四、配置 Dev Gateway

创建配置文件：

``` bash
mkdir /etc/nginx/conf.d/dev
vi /etc/nginx/conf.d/dev/order-service.conf
```

示例配置：

``` nginx
server {
    listen 10001;

    location / {

        proxy_pass http://100.86.39.109:10001;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    }
}
```

说明：

  参数         说明
  ------------ ----------------------
  listen       Dev Gateway 监听端口
  proxy_pass   开发者 Tailscale IP
  Host         转发请求头

请求流程：

    Gateway
       │
    192.168.3.200:10001
       │
    Nginx
       │
    100.86.39.109:10001
       │
    开发者本地服务

------------------------------------------------------------------------

# 五、测试配置

检查 nginx 配置：

``` bash
nginx -t
```

输出：

    syntax is ok
    test is successful

重启 nginx：

``` bash
systemctl restart nginx
```

------------------------------------------------------------------------

# 六、防火墙开放端口

查看防火墙状态：

``` bash
firewall-cmd --state
```

开放端口：

``` bash
firewall-cmd --permanent --add-port=10001/tcp
firewall-cmd --reload
```

开放端口段：

``` bash
firewall-cmd --permanent --add-port=10000-10100/tcp
firewall-cmd --reload
```

------------------------------------------------------------------------

# 七、SELinux 配置

生产环境 **不建议关闭 SELinux**。

查看状态：

``` bash
getenforce
```

输出示例：

    Enforcing

表示 SELinux 正在生效。

------------------------------------------------------------------------

## 1 安装管理工具

``` bash
yum install policycoreutils-python -y
```

------------------------------------------------------------------------

## 2 允许 nginx 使用自定义端口

例如开放 10001 端口：

``` bash
semanage port -a -t http_port_t -p tcp 10001
```

查看端口列表：

``` bash
semanage port -l | grep http
```

输出示例：

    http_port_t  tcp  80,443,8080,10001

------------------------------------------------------------------------

## 3 允许 nginx 访问网络

反向代理必须开启：

``` bash
setsebool -P httpd_can_network_connect 1
```

否则会出现：

    502 Bad Gateway

------------------------------------------------------------------------

## 4 推荐 SELinux 配置

``` bash
semanage port -a -t http_port_t -p tcp 10000-10100
setsebool -P httpd_can_network_connect 1
```

这样可以一次性开放开发端口段。

------------------------------------------------------------------------

# 八、Nacos 注册配置

开发者服务注册：

``` yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.3.201:8848
        ip: 192.168.3.200
```

Nacos 实例：

    order-service
    192.168.3.200:10001

------------------------------------------------------------------------

# 九、调用流程

远程调试：

    Postman
       │
    Gateway
       │
    192.168.3.200
       │
    Nginx Dev Gateway
       │
    Tailscale
       │
    开发者服务

内网用户：

    Gateway
       │
    192.168.3.x

不会经过 Dev Gateway。

------------------------------------------------------------------------

# 十、最佳实践

推荐配置：

  组件          服务器
  ------------- ---------------
  Nacos         192.168.3.201
  Gateway       192.168.3.201
  Dev Gateway   192.168.3.200

------------------------------------------------------------------------

# 十一、总结

通过 **Tailscale + Nginx Dev Gateway** 可以实现：

-   远程开发接入公司微服务环境
-   内网服务完全不受影响
-   不需要修改每个服务配置
-   SELinux 保持开启

该方案适合：

-   Spring Cloud
-   Nacos
-   微服务架构远程开发
