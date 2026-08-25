---
title: springcloud项目部署为https
date: 2026-02-06
---



#  springcloud项目部署为https

##  一、环境准备

### 1、安装jdk、openssl 并配置环境变量

## 二、生成证书

### 1、注意事项

1.1、 本次部署文档  采用  自签根证书、根证书签发服务证书、配置信任库的模式

1.2、下述证书名 密码 域名 IP地址均可替

1.3、网关使用gateway时，若gateway配置https，下游服务为http，使用lb:服务名形式设置uri 可能会导致 调用下游服务时请求会沿用https，请求失败。可用uri：主机名：端口的形式配置

1.4、除网关外的所有下游服务需要保持协议一致，若跨协议，feign调用会失败

1.5、证书文件类型可采用PKCS12等，JKS类型springboot可能会读取失败(仅限于本次测试结果)

### 2、证书生成

2.1.1、 生成根证书

keytool -genkeypair -alias rootca -keyalg RSA -keysize 2048 -validity 3650 -keystore rootca.jks 
-dname "CN=Root CA, OU=OrgUnit, O=Org, L=City, ST=State, C=Country" -ext bc:c

参数说明：

alias: 证书别名

keystore: 证书文件名

dname: 描述信息

storepass：证书密钥

2.1.2 、导出根 CA 证书（如果还没有）
如果你已经有 rootca.cer，可以跳过这一步。否则，使用以下命令导出根 CA 证书：
keytool -exportcert -alias rootca -file rootca.crt -keystore rootca.jks -storepass 123456 -rfc

2.1.3、 提取根 CA 的私钥
由于 keytool 不直接支持从 JKS 提取私钥，我们可以先将 JKS 转换为 PKCS#12 格式，然后再使用 OpenSSL 提取私钥。
2.1.4、 将 JKS 转换为 PKCS#12 格式
keytool -importkeystore -srckeystore rootca.jks -destkeystore rootca.p12 -srcstoretype JKS -deststoretype PKCS12 -srcstorepass 123456 -deststorepass 123456 -alias rootca

2.1.5、 使用 OpenSSL 提取私钥
openssl pkcs12 -in rootca.p12 -nocerts -out rootca.key -nodes -passin pass:123456

2.2、生成服务证书

2.2.1、创建网关服务的密钥库并生成密钥对
使用 keytool 创建网关服务的密钥库并生成密钥对：
keytool -genkeypair -alias dist -keyalg RSA -keysize 2048 -keystore dist.jks -storepass 123456 -keypass 123456 -dname "CN=gateway.example.com, OU=OrgUnit, O=Organization, L=City, ST=State, C=CN"



2.2.2、从密钥库中导出证书签名请求 (CSR)
从网关密钥库中导出 CSR：
keytool -certreq -alias dist -file dist.csr -keystore dist.jks -storepass 123456

2.2.3、创建 OpenSSL 配置文件
创建一个名为 openssl.cnf 的配置文件，用于指定证书的详细信息和扩展字段：

#openssl.cnf

[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
prompt             = no

[ req_distinguished_name ]
C  = CN
ST = State
L  = City
O  = Organization
OU = OrgUnit
CN = gateway.example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.168.3.166
DNS.1 = gateway.example.com

2.2.4、 使用 OpenSSL 签发证书
使用 OpenSSL 签发证书并将 SAN 扩展添加到证书中：
openssl x509 -req -in dist.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial -out dist.crt -days 365 -extfile openssl.cnf -extensions req_ext

2.2.5、导入根 CA 证书到网关密钥库
将根 CA 证书导入到网关密钥库中：
keytool -importcert -alias rootca -file rootca.crt -keystore dist.jks -storepass 123456 -noprompt

2.2.6、导入网关的签发证书到网关密钥库
将网关的签发证书导入到网关密钥库中：
keytool -importcert -alias dist -file dist.crt -keystore dist.jks -storepass 123456

2.2.7、将 JKS 转换为 PKCS#12 格式（可选）
keytool -importkeystore -srckeystore dist.jks -destkeystore dist.p12 -srcstoretype JKS -deststoretype PKCS12 -srcstorepass 123456 -deststorepass 123456  -alias dist

2.3、配置信任库

2.3.1、创建信任库

keytool -storetype JKS -keystore truststore.jks -storepass 123456 -deststoretype PKCS12

2.3.2  导入根 CA 证书到 truststore.p12
keytool -importcert -v -alias dist -file dist.cer -keystore truststore.p12 -storetype PKCS12 -storepass 123456 -noprompt

根证书 以及每个服务的证书都需要导入到信任库

## 三、配置服务

## 1、springcloud 项目配置

1.1、将项目证书和信任库放到指定位置，（一般是项目resources下)

配置文件示例：

server:

  port: 10084

  servlet:

​    context-path:



  ssl:

​    enabled: false

​    key-store: classpath:bizz.p12  # 新服务的 keystore 文件路径

​    #key-store: file:/bizz.p12   # 绝对路径写法

​    key-store-password: 123456         # keystore 密码

​    key-alias: bizz   # 服务证书别名

​    key-store-type: PKCS12                # 证书文件类型

​    trust-store: classpath:truststore.p12  # 共享的信任库文件路径

​    trust-store-password: 123456        # 信任库密码

​    trust-store-type: PKCS12

## 2、前端配置(项目中采用ngxin部署)

2.1、证书生成

按照上述方法生成前端证书，详见第二点

2.2、配置示例

#监听 HTTP 请求并重定向到 HTTPS

server {
    listen 80;
    server_name 192.168.3.166;  # 替换为 IP 地址

    # 将 HTTP 请求重定向到 HTTPS
    return 301 https://$host:5173$request_uri;
}

#监听 HTTPS 请求

server {
    listen 5173 ssl;
    server_name 192.168.3.166;  # 替换为 IP 地址

    ssl_certificate E:/nginx-1.18.0/conf/ssl/dist.crt;
    ssl_certificate_key E:/nginx-1.18.0/conf/ssl/dist.key;


    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    # 静态文件服务配置
    location / {
        root E:/kunteng/qianduan/OA/dist;  # 确保路径正确
        index index.html;
    }
    
    # 反向代理到你的 Spring Cloud Gateway
    location /api/ {
        proxy_pass https://192.168.3.166:38760;  # 替换为实际的网关地址和端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}