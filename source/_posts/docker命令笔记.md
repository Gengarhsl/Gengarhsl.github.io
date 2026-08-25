---
title: docker命令笔记
date: 2026-02-06
---



docker version        # 显示docker的版本信息
docker info              # 显示docker的系统信息，包括镜像和容器的数量
docker 命令 --help         # 帮助命令
docker images	#查看镜像
docker search	#搜索镜像
docker pull	#下载镜像
docker pull  mysql:5.7	#指定版本下载
docker rmi 	#删除镜像
[root@JWei_0124 //]# docker rmi -f 镜像id                    # 删除指定的镜像
[root@JWei_0124 //]# docker rmi -f 镜像id 镜像id 镜像id    # 删除多个镜像（空格分隔）
[root@JWei_0124 //]# docker rmi -f $(docker images -aq)    # 删除全部的镜像
docker run [可选参数] image

# 参数说明
--name="name"        容器名字：用来区分容器
-d                    后台方式运行：相当于nohup
-it                    使用交互式运行：进入容器查看内容
-p                    指定容器的端口（四种方式）小写字母p
    -p ip:主机端口：容器端口
    -p 主机端口：容器端口
    -p 容器端口
    容器端口
-P                     随机指定端口（大写字母P）

docker ps	 	#列出当前正在运行的容器
# 命令参数可选项
-a        # 列出当前正在运行的容器+历史运行过的容器
-n=?    # 显示最近创建的容器（可以指定显示几条，比如-n=1）
-q        # 只显示容器的编号

docker run -it centos /bin/bash #交互式进入
docker rm 容器ID	#删除容器（不能删除正在运行的容器）
docker rm -f 	#强制删除
docker ps -a -q|xargs docker rm 	#删除所有容器
docker start 容器id        # 启动容器
docker restart 容器id    # 重启容器
docker stop 容器id        # 停止当前正在运行的容器
docker kill 容器id        # 强制停止当前容器

docker logs -tf --tail 容器id

-tf                        # 显示日志
--tail number    # 要显示的日志条数

查看容器中进程的信息
docker top id
查看镜像的元数据
docker inspect id
进入当前正在运行的容器
docker exec -it 容器id /bin/bash

从容器内拷贝文件到主机
docker cp 容器id:容器内路径 目的主机的路径
# 如何确定是具名挂载，还是匿名挂载，还是指定路径挂载
-v 容器内的路径                # 匿名挂载
-v 卷名:容器内的路径        # 具名挂载
-v /宿主机路径:容器内路径    # 指定路径挂载

# 提交容器成为一个新的副本
docker commit
# 命令和git原理类似
docker commit -m="提交的描述信息" -a="作者" 容器id 目标镜像名:[TAG]


# 我们直接启动的命令--net bridge（这个就是我们的docker0）；默认带上这个参数的，以下两种启动方式效果一致。
docker run -d -P --name tomcat01 tomcat
docker run -d -P --name tomcato1 --het bridge tomcat
# docker0特点：默认，域名不能访问，--1ink可以打通连接！
# 我们可以自定义一个网络！
[root@iZbp13qr3mm4ucsjumrlgqZ ~]# docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet
[root@iZbp13qr3mm4ucsjumrlgqZ ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
7254ffccbdf5   bridge    bridge    local
45610891738f   host      host      local
266acd66473c   mynet     bridge    local
7795cbc2686c   none      null      local
