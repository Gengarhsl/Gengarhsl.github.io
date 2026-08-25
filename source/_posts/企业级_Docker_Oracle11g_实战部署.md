---
date: 2026-02-06
title: Docker企业级实战部署 Oracle11g
---

# Docker 企业级实战部署 Oracle11g（11.2.0.3）

适用场景：

-   本地开发环境
-   微服务数据库测试
-   Oracle 老系统迁移
-   内网测试数据库

特点：

-   一键部署
-   持久化建议
-   企业实战流程
-   启动监听与数据库
-   常见问题解决

------------------------------------------------------------------------

# 一、环境要求（生产建议）

  项目     建议配置
  -------- ----------
  CPU      ≥2核
  内存     ≥4G
  磁盘     ≥20G
  Docker   ≥20.x

检查 Docker：

``` bash
docker -v
```

------------------------------------------------------------------------

# 二、拉取 Oracle11g 镜像

## 方式1：DockerHub（慢）

``` bash
docker pull lhrbest/oracle_11g_ee_lhr_11.2.0.3:1.0
```

## 方式2：阿里云镜像（推荐）

``` bash
docker pull registry.cn-hangzhou.aliyuncs.com/lhrbest/oracle_11g_ee_lhr_11.2.0.3:1.0
```

重新打tag：

``` bash
docker tag registry.cn-hangzhou.aliyuncs.com/lhrbest/oracle_11g_ee_lhr_11.2.0.3:1.0 lhrbest/oracle_11g_ee_lhr_11.2.0.3:1.0
```

查看镜像：

``` bash
docker images | grep oracle
```

------------------------------------------------------------------------

# 三、创建 Oracle 容器（实战标准）

``` bash
docker run -itd --name oracle11g -h oracle11g --privileged=true -p 1521:1521 -p 1158:1158 -p 222:22 lhrbest/oracle_11g_ee_lhr_11.2.0.3:1.0 init
```

端口说明：

  端口   作用
  ------ ------------
  1521   数据库端口
  1158   EM管理
  222    SSH

------------------------------------------------------------------------

# 四、启动数据库（核心步骤）

进入容器：

``` bash
docker exec -it oracle11g bash
```

切换用户：

``` bash
su - oracle
```

启动监听：

``` bash
lsnrctl start
```

进入数据库：

``` bash
sqlplus / as sysdba
```

启动实例：

``` sql
startup;
```

------------------------------------------------------------------------

# 五、连接数据库

连接信息：

  项目   内容
  ------ ------------
  IP     服务器IP
  端口   1521
  SID    LHR11G
  用户   system/sys

连接示例：

``` bash
sqlplus system/oracle@服务器IP:1521/LHR11G
```

------------------------------------------------------------------------

# 六、企业级持久化（强烈建议）

创建目录：

``` bash
mkdir -p /data/oracle11g
chmod -R 777 /data/oracle11g
```

推荐运行：

``` bash
docker run -itd --name oracle11g --privileged=true -p 1521:1521 -v /data/oracle11g:/u01 lhrbest/oracle_11g_ee_lhr_11.2.0.3:1.0 init
```

------------------------------------------------------------------------

# 七、SpringBoot连接配置

``` yaml
spring:
  datasource:
    driver-class-name: oracle.jdbc.OracleDriver
    url: jdbc:oracle:thin:@服务器IP:1521:LHR11G
    username: system
    password: oracle
```

------------------------------------------------------------------------

# 八、常用管理命令

查看监听：

``` bash
lsnrctl status
```

查看实例：

``` sql
select status from v$instance;
```

查看用户：

``` sql
select username from dba_users;
```

------------------------------------------------------------------------

# 九、日志查看

``` bash
docker logs -f oracle11g
```

------------------------------------------------------------------------

# 十、常见问题（实战踩坑）

## 1. 无法连接

检查：

``` bash
netstat -ntlp | grep 1521
```

## 2. 数据库未启动

``` sql
startup;
```

## 3. 内存不足

修改 Docker 启动内存限制。

------------------------------------------------------------------------

# 十一、企业最佳实践

-   不要用于生产正式业务
-   建议用于开发测试
-   建议使用 Oracle19c 替代
-   建议加数据卷持久化
-   定期备份 /u01 目录
