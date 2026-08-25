---
title: Docker企业级部署Elasticsearch与Kibana实践
date: 2026-02-06
---

# Docker 企业级部署 Elasticsearch + Kibana + IK 分词器

> 适用于：生产环境 / SpringCloud 微服务 / 日志平台 / 搜索服务

特点： - docker-compose 一键部署 - 启用安全认证 - 持久化数据 -
插件标准化安装 - 容器网络隔离 - 可直接扩展为集群

------------------------------------------------------------------------

# 一、目录规划（生产推荐）

``` bash
mkdir -p /data/es/{data,plugins,config,certs}
mkdir -p /data/kibana/{config,data}
```

权限：

``` bash
chmod -R 777 /data/es
chmod -R 777 /data/kibana
```

------------------------------------------------------------------------

# 二、docker-compose 企业级部署

创建：

``` bash
mkdir -p /opt/es-stack
cd /opt/es-stack
vim docker-compose.yml
```

## docker-compose.yml

``` yaml
version: '3.8'

services:

  elasticsearch:
    image: elasticsearch:8.6.0
    container_name: es
    restart: always
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=Elastic@123
    volumes:
      - /data/es/data:/usr/share/elasticsearch/data
      - /data/es/plugins:/usr/share/elasticsearch/plugins
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - es-net

  kibana:
    image: kibana:8.6.0
    container_name: kibana
    restart: always
    environment:
      - ELASTICSEARCH_HOSTS=http://es:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=Elastic@123
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - es-net

networks:
  es-net:
    driver: bridge
```

------------------------------------------------------------------------

# 三、启动服务

``` bash
docker compose up -d
```

查看状态：

``` bash
docker ps
```

------------------------------------------------------------------------

# 四、访问测试

Elasticsearch：

``` text
http://服务器IP:9200
```

账号：

``` text
elastic
Elastic@123
```

Kibana：

``` text
http://服务器IP:5601
```

------------------------------------------------------------------------

# 五、企业级 IK 分词器安装

进入容器：

``` bash
docker exec -it es bash
```

安装插件：

``` bash
./bin/elasticsearch-plugin install https://get.infini.cloud/elasticsearch/analysis-ik/8.6.0
```

退出并重启：

``` bash
docker restart es
```

------------------------------------------------------------------------

# 六、SpringBoot 企业连接配置

``` yaml
spring:
  elasticsearch:
    uris: http://服务器IP:9200
    username: elastic
    password: Elastic@123
```

------------------------------------------------------------------------

# 七、生产环境优化建议

## JVM优化

``` yaml
ES_JAVA_OPTS=-Xms2g -Xmx2g
```

原则： - 不超过物理内存 50% - Xms = Xmx

## 内核参数（必须）

``` bash
vim /etc/sysctl.conf
```

``` text
vm.max_map_count=262144
```

生效：

``` bash
sysctl -p
```

------------------------------------------------------------------------

# 八、日志查看

``` bash
docker logs -f es
docker logs -f kibana
```

------------------------------------------------------------------------

# 九、集群扩展（企业进阶）

后续可扩展：

-   ES 三节点集群
-   专用 master 节点
-   专用 data 节点
-   接入 Filebeat / Logstash
-   对接 SpringCloud 微服务日志中心
-   对接 MQTT / IoT 数据检索

------------------------------------------------------------------------

# 十、企业部署最佳实践总结

-   使用 docker-compose 管理服务
-   必须开启安全认证
-   数据必须挂载宿主机
-   禁止 root 运行容器
-   JVM 内存按机器资源合理配置
-   插件版本必须与 ES 一致
-   生产环境建议至少三节点
