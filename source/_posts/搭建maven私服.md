---
title: 搭建maven私服手册
date: 2026-02-06
---



# 搭建maven私服

## 一、docker版本

docker pull sonatype/nexus3

mkdir -p /home/nexus/data
chmod 777 -R /home/nexus/data



docker run -d --name nexus3 -p 8081:8081 --restart always -v /home/nexus/data:/nexus-data sonatype/nexus3



docker logs -f nexus3

访问地址：http:ip:8081

默认密码存放位置：

cat /home/nexus/data/admin.password



1、默认仓库说明：

maven-central：maven中央库，默认从https://repo1.maven.org/maven2/拉取jar
maven-releases：私库发行版jar，初次安装请将Deployment policy设置为Allow redeploy
maven-snapshots：私库快照（调试版本）jar
maven-public：仓库分组，把上面三个仓库组合在一起对外提供服务，在本地maven基础配置settings.xml或项目pom.xml中使用

2、nexus仓库类型说明

hosted：本地仓库，通常我们会部署自己的构件到这一类型的仓库。比如公司的第二方库。
proxy：代理仓库，它们被用来代理远程的公共仓库，如maven中央仓库。
group：仓库组，用来合并多个hosted/proxy仓库，当你的项目希望在多个repository使用资源时就不需要多次引用了，只需要引用一个group即可。

3、创建Blob stores

在创建repository之前，还需要先指定文件存储目录，便于统一管理。就需要创建Blob Stores，不创建则使用的是default



4、

jboss的maven中央仓库地址：http://repository.jboss.com/maven2/
阿里云的maven中央仓库地址：http://maven.aliyun.com/nexus/content/groups/public/
apache的maven中央仓库地址：http://repo.maven.apache.org/maven2/



Hosted有三种方式：Releases、Snapshot、Mixed

Releases: 一般是已经发布的Jar包
Snapshot: 未发布的版本
Mixed：混合的

Allow redeploy：允许同一个版本号下重复提交代码, nexus以时间区分
Disable redeploy：不允许同一个版本号下重复提交代码
Read-Only：不允许提交任何版本
原生的maven-releases库是Disable redeploy设置， maven-snapshots是Allow redeploy。

5、settings.xml 配置

<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">

    <localRepository>D:\123</localRepository>
    
    <pluginGroups>
    </pluginGroups>


    <proxies>
    </proxies>
    
    <servers>
    
        <server>
            <id>maven-public</id>
            <username>admin</username>
            <password>123456</password>
        </server>
    
    </servers>


    <mirrors>
    
        <mirror>
            <id>maven-public</id>
            <mirrorOf>*</mirrorOf>
            <name>maven-public</name>
            <url>http://192.168.3.108:8081/repository/maven-public/</url>
        </mirror>
    
    </mirrors>
    
    <profiles>
    
        <!-- 配置私服 -->
    
        <profile>
            <id>maven-public</id>
            <repositories>
                <repository>
                    <id>maven-public</id>
                    <name>Repository for nexus</name>
                    <url>http://192.168.3.108:8081/repository/maven-public/</url>
                </repository>
            </repositories>
        </profile>


    </profiles>
    
    <!-- 配置私服 -->
    
    <activeProfiles>
        <activeProfile>maven-public</activeProfile>
    </activeProfiles>
</settings>



## 二、windows版

1、下载地址

[Download](https://help.sonatype.com/en/download.html)

2、运行

D:\App\nexus-3.55.0-01-win64\nexus-3.55.0-01\bin>nexus.exe /run

用户名：admin（默认）

密码：第一次启动时生成，(在/sonatype-work/nexus3/admin.password 文件中) 