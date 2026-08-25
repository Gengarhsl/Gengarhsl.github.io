---
title: Docker企业级部署Seata分布式事务（Nacos + MySQL + Oracle支持）
date: 2026-02-06
tags:
- Docker
- Seata
- 分布式事务
- 微服务中间件
categories:
- 中间件实战
---

# 一、Seata简介

Seata 是一款开源的分布式事务解决方案，支持：

- AT 模式（最常用）
- TCC 模式
- SAGA 模式
- XA 模式

核心组件：

| 组件 | 作用                         |
| ---- | ---------------------------- |
| TC   | 事务协调器（Seata Server）   |
| TM   | 事务管理器（发起事务的服务） |
| RM   | 资源管理器（数据库操作服务） |

核心流程：

1. TM 开启全局事务
2. TC 生成 XID
3. XID 在服务间传播
4. RM 注册分支事务
5. TM 决议提交或回滚
6. TC 通知所有RM执行

---

# 二、环境准备

## 1、软件环境

Docker >= 23
Nacos >= 2.2
MySQL 或 Oracle
JDK 8+


---

## 2、目录规划（企业标准）

/data/docker/seata
/data/docker/seata/resources
/data/docker/seata/logs


创建目录：

mkdir -p /data/docker/seata/resources
mkdir -p /data/docker/seata/logs


---

# 三、拉取Seata镜像

## 1、查看镜像

docker search seata


推荐官方镜像：

seataio/seata-server


---

## 2、拉取指定稳定版本（推荐）

docker pull seataio/seata-server:1.6.1


查看：

docker images


---

# 四、创建Seata数据库

下载官方脚本：

https://github.com/apache/incubator-seata/tree/master/script/server/db


创建数据库：

CREATE DATABASE seata;


执行：

mysql.sql


---

# 五、初始化配置文件

## 1、先启动一次容器

docker run -d --name seata-temp
-p 7091:7091
seataio/seata-server:1.6.1


---

## 2、复制配置

docker cp seata-temp:/seata-server/resources /data/docker/seata/


---

## 3、删除临时容器

docker rm -f seata-temp


---

# 六、修改 application.yml（企业版）

server:
port: 7091

spring:
application:
name: seata-server

console:
user:
username: seata
password: seata

seata:
config:
type: nacos
nacos:
server-addr: 192.168.1.10:8848
group: SEATA_GROUP
username: nacos
password: nacos
data-id: seata.properties

registry:
type: nacos
nacos:
application: seata-server
server-addr: 192.168.1.10:8848
group: SEATA_GROUP
cluster: default
username: nacos
password: nacos

store:
mode: db
db:
datasource: druid
db-type: mysql
driver-class-name: com.mysql.jdbc.Driver
url: jdbc:mysql://192.168.1.20:3306/seata
user: root
password: 123456

security:
secretKey: SeataSecretKey
tokenValidityInMilliseconds: 1800000


---

# 七、Nacos配置中心配置

创建：

DataId: seata.properties
Group: SEATA_GROUP


内容：

store.mode=db
store.db.dbType=mysql
store.db.url=jdbc:mysql://192.168.1.20:3306/seata
store.db.user=root
store.db.password=123456


---

# 八、正式启动Seata（生产版）

docker run -d
--name seata
--restart=always
-p 8091:8091
-p 7091:7091
-e SEATA_IP=192.168.1.30
-v /data/docker/seata/resources:/seata-server/resources
-v /data/docker/seata/logs:/root/logs
seataio/seata-server:1.6.1


---

# 九、验证部署

## 查看容器

docker ps


## 查看日志

docker logs -f seata


## 查看Nacos

服务列表出现 seata-server


---

# 十、微服务接入Seata

入口方法：

@GlobalTransactional


普通方法：

@Transactional


---

# 十一、Oracle数据库 undo_log

CREATE TABLE undo_log (
id NUMBER(19) NOT NULL,
branch_id NUMBER(19) NOT NULL,
xid VARCHAR2(100) NOT NULL,
context VARCHAR2(128) NOT NULL,
rollback_info BLOB NOT NULL,
log_status NUMBER(10) NOT NULL,
log_created DATE NOT NULL,
log_modified DATE NOT NULL,
ext VARCHAR2(100),
CONSTRAINT pk_undo_log PRIMARY KEY (id)
);


---

# 十二、企业级避坑指南

1. 不支持复杂SQL
2. 不支持MP批量操作
3. Feign必须检查返回值
4. 每个业务库必须有 undo_log
5. AT模式入口必须使用 GlobalTransactional
6. Oracle建议驱动19+
7. Druid建议1.2.20+
8. 异常不能吞
9. 不支持事务嵌套
10. 低版本Seata序列化问题严重

---

# 十三、企业部署架构建议

Nacos集群
MySQL主从
Seata集群
Docker部署
日志集中收集


---

# 十四、生产环境建议

开启restart=always
开启日志挂载
禁止使用latest版本
配置独立数据库
使用内网IP注册


---