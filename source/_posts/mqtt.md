---
title: docker部署mqtt
date: 2026-02-06
---



Docker 部署 MQTT服务器(EMQX)

m0_46601820

于 2023-12-14 09:10:40 发布

阅读量1.4k
 收藏 12

点赞数 10
文章标签： docker 服务器 容器
版权
1.拉取镜像

# 拉取最新版本
docker pull emqx/emqx:latest
# 也可以拉取指定版本
docker pull emqx/emqx:v4.0.0
2.创建临时容器

docker run -d --name emqx -p 1883:1883 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx/emqx:latest 
端口：

1883：MQTT协议端口
8083：WebSocket协议端口
8084：HTTP协议端口
8883：MQTT over SSL协议端口
18083：EMQX Dashboard Web管理界面端口
 3.创建文件挂载目录

mkdir -p /app/emqx/etc /app/emqx/lib /app/emqx/data /app/emqx/log
4.从容器内复制配置文件到挂载目录下

docker cp emqx:/opt/emqx/etc /app/emqx/etc
 5.强制删除docker 容器

docker rm -f emqx
6.运行并挂载文件代码

docker run -d \
--restart=always \
--name emqx \
-p 1883:1883 \
-p 8883:8883 \
-p 8083:8083 \
-p 8084:8084 \
-p 8081:8081 \
-p 18083:18083 \
-v /app/emqx/etc:/opt/emqx/etc \
emqx/emqx:latest
7.登录EMQX 内置的管理控制台 

http://localhost:18083/

默认用户 admin，默认密码 public

8.设置客户端认证

在 EMQX Dashboard 页面，点击左侧导航栏的访问控制 -> 认证，在随即打开的认证页面，单击创建，依次选择认证方式为 Password-Based，数据源为 Built-in Database，进入配置参数



Docker | EMQX 文档

使用Emqx5 | FastBee文档
————————————————

                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。

原文链接：https://blog.csdn.net/m0_46601820/article/details/134969551

