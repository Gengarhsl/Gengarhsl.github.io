---
title: git-jenkins-harbor-k8s部署笔记
date: 2026-02-06
---



1.3 软硬件环境
角色	主机名	ip地址	备注
master	k8s-master	192.168.163.151	
k8s主节点、8核CPU、4GB(CD)

node	k8s-node1	192.168.163.152	
k8s从节点、8核CPU、4GB

node	k8s-node2	192.168.163.153	
k8s从节点、8核CPU、4GB

Gitlab+Jenkins+Docker	jenkins	192.168.163.155	
 8核CPU、12GB，Gitlab最少需要4G(CI)

docker镜像仓库:harbor	harbor	192.168.163.156	
8核CPU、4GB

软件	版本
docker	26.1.3
k8s	1.22.2
harbor	2.0.6
Jenkins	2.457(尽量别用latest版本安装 可能导致插件安装失败)
gitlab	10.7.5
二、环境准备
在所有节点操作

2.1 关闭NetworkManager
NetworkManager会和network启动是发生冲突，导致network启动失败，不能远程访问虚拟机；

systemctl stop NetworkManager                #临时关闭
systemctl disable NetworkManager             #永久关闭网络管理命令
systemctl start network.service              #开启网络服务
2.2 设置静态ip
  2.2.1查看网络配置

ifconfig #查看网络配置
2.2.2 修改网络配置文件

vi /etc/sysconfig/network-scripts/ifcfg-ens33
1）选择动态分配或者静态分配

BOOTPROTO=static    #dhcp：自动分配ip ，static：静态ip
2）开启自动打开网卡

ONBOOT=yes     #开启启动必须是yes


TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static    #dhcp：自动分配ip ，static：静态ip
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=9c922cc8-b3ed-419a-88bd-6fd756e04880
DEVICE=ens33
ONBOOT=yes  #开启启动必须是yes
IPADDR=192.168.163.152 #静态ip
NETMASK=255.255.255.0  #
GATEWAY=192.168.163.2  #网关
DNS1=192.168.163.0   #DNS
DNS2=114.114.114.114
2.2.3重启服务

systemctl restart network
2.2.4查看网络状态


systemctl status network.service


### 2.3 修改主机名 

```bash
hostnamectl set-hostname k8s-master #k8s-master 为主机名称
```

### 2.3 所有机器关闭防火墙 

```bash
 systemctl stop firewalld	#关闭



 systemctl disable firewalld		#开机不自启



 systemctl status firewalld		#查看状态
```

### 2.4 所有机器关闭selinux 

```bash
 sed -i 's/enforcing/disabled/' /etc/selinux/config 



 setenforce 0



  systemctl status firewalld
```

### 2.5 所有机器关闭swap 

```bash
 swapoff -a # 临时关闭



 sed -ri 's/.*swap.*/#&/' /etc/fstab  #永久关闭



 swapon -s #查看swapon分区
```

2.6 为所有节点安装docker 
yum install wget.x86_64 -y
rm -rf /etc/yum.repos.d/*
wget -O /etc/yum.repos.d/centos7.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install docker-ce-20.10.11 -y
systemctl start docker
systemctl enable docker
docker version

配置docker加速器

 mkdir -p /etc/docker
 tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"]
}
EOF
 systemctl daemon-reload
 systemctl restart docker
三、部署k8s集群 
3.1 添加主机名与ip的对应关系（k8s-master、k8s-node1、k8s-node2）
cat >> /etc/hosts << EOF
192.168.163.151 k8s-master
192.168.163.152 k8s-node1
192.168.163.153 k8s-node2
EOF

source /etc/profile

3.2 将桥接的ipv4流量传递到iptables的链（k8s-master、k8s-node1、k8s-node2） 
cat >> /etc/sysctl.d/k8s.conf << EOF
 net.bridge.bridge-nf-call-ip6tables = 1
 net.bridge.bridge-nf-call-iptables = 1
EOF

3.3 安装kubeadm、kubelet、kubectl（k8s-master、k8s-node1、k8s-node2） 
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install kubelet-1.22.2 kubeadm-1.22.2 kubectl-1.22.2 -y
systemctl enable kubelet && systemctl start kubelet
kubectl version

3.4 修改docker的配置（k8s-master、k8s-node1、k8s-node2）  
#镜像加速 registry-mirrors
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"] 
}
EOF
systemctl daemon-reload
systemctl restart docker.service
systemctl restart kubelet.service
systemctl status kubelet.service

这里从节点的kubelet.service状态报code=exited, status=1/FAILURE是正常的 集群没有初始化导致的

3.5 部署master节点（主节点k8s-master）
3.5.1 集群初始化
kubeadm init \
--apiserver-advertise-address=192.168.71.129 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.23.0 \
--control-plane-endpoint k8s-master \
--service-cidr=172.16.0.0/16 \
--pod-network-cidr=10.244.0.0/16
这段命令是用于将一个工作节点（worker node）加入到已存在的 Kubernetes 集群中的过程。

kubeadm init 是用来初始化 Kubernetes 集群的命令。下面是一些常用的参数及其说明：

--apiserver-advertise-address: 指定用于通告的 API 服务器的 IP 地址。

--apiserver-bind-port: 指定 API 服务器绑定的端口。

--control-plane-endpoint: 指定控制平面的端口。

--image-repository: 设置 Kubernetes 镜像仓库的地址。

--kubernetes-version: 设置 Kubernetes 版本。

--pod-network-cidr: 设置 Pod 网络的 CIDR 范围。

--service-cidr: 设置服务的 CIDR 范围。

1).遇到报错： 



2).解决办法：关闭swap

 swapoff -a # 临时关闭
 sed -ri 's/.*swap.*/#&/' /etc/fstab  #永久关闭
1).遇到报错：

Here is one example how you may list all Kubernetes containers running in docker:
                - 'docker ps -a | grep kube | grep -v pause'
                Once you have found the failing container, you can inspect its logs with:
                - 'docker logs CONTAINERID'

error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher
2).解决办法：

rm -rf /etc/containerd/config.toml
systemctl restart containerd
 如果初始化失败，可以重新初始化

kubeadm reset
此命令将删除当前集群的状态信息，并使其回到初始状态

初始化成功

记下红框中内容，添加节点时需要

3.5.2 按照指示执行
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
初始化时 该命令有输出



3.5.3 查看kubelet.service状态 
systemctl status kubelet.service

3.5.4 查看节点状态为notready 
kubectl get nodes


3.5.5 安装网络插件
官方文档：https://github.com/flannel-io/flannel

# 最好手动提前拉取所需镜像
docker pull quay.io/coreos/flannel:v0.14.0
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml 
kubectl get pods --all-namespaces


3.5.6 添加node节点 
在k8s-node1和k8s-node1节点执行

docker pull quay.io/coreos/flannel:v0.14.0
kubeadm join k8s-master:6443 --token yt11th.b5wzq4lkjxcsljpg \
	--discovery-token-ca-cert-hash sha256:f36824a14f1acb9413246e8f5f0cdf5f6060066f7e4eec6d27edd69004499f9f




在主节点k8s-master查看节点状态为notready 



 四、部署Gitlab（192.168.163.155）
4.1 配置docker加速器
 mkdir -p /etc/docker
 tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"]
}
EOF
 systemctl daemon-reload
 systemctl restart docker


4.2 安装并配置Gitlab
docker pull beginor/gitlab-ce


4.2.1 创建共享卷目录
mkdir -p /data/gitlab/etc/ /data/gitlab/log/ /data/gitlab/data
chmod 777 /data/gitlab/etc/ /data/gitlab/log/ /data/gitlab/data/
4.2.2 创建 gitlab 容器
docker run -itd --name=gitlab --restart=always --privileged=true   -p 8443:443  -p 80:80 -p 222:22 -v  /data/gitlab/etc:/etc/gitlab -v  /data/gitlab/log:/var/log/gitlab -v  /data/gitlab/data:/var/opt/gitlab  beginor/gitlab-ce
docker ps
 切记:这里的端口要设置成80，要不push项目会提示没有报错，如果宿主机端口被占用，需要把这个端口腾出来



4.2.3 关闭容器修改配置文件 
docker stop gitlab
cat /data/gitlab/etc/gitlab.rb |grep external_url
sed -i "/external_url 'GENERATED_EXTERNAL_URL'/a external_url\t'http://192.168.163.155' "  /data/gitlab/etc/gitlab.rb
cat /data/gitlab/etc/gitlab.rb |grep external_url
 external_url ‘http://192.168.163.155’

 gitlab_rails[‘gitlab_ssh_host’] = ‘192.168.163.155’’

cat /data/gitlab/etc/gitlab.rb |grep gitlab_ssh_host
sed -i "/gitlab_ssh_host/a gitlab_rails['gitlab_ssh_host'] = '192.168.163.155' "  /data/gitlab/etc/gitlab.rb
cat /data/gitlab/etc/gitlab.rb |grep gitlab_ssh_host


gitlab_rails[gitlab_shell_ssh_port] = 222

cat /data/gitlab/etc/gitlab.rb | grep gitlab_shell_ssh
sed -i "/gitlab_shell_ssh_port/a gitlab_rails['gitlab_shell_ssh_port'] = 222" /data/gitlab/etc/gitlab.rb
cat /data/gitlab/etc/gitlab.rb | grep gitlab_shell_ssh
vim /data/gitlab/data/gitlab-rails/etc/gitlab.yml


4.2.4 修改完配置文件之后。直接启动容器 
docker start gitlab
docker ps
在宿主机所在的物理机访问，http://192.168.163.155/ ，会自动跳转到修改密码(root用户),如果密码设置的没有满足一定的复杂性，则会报500，需要从新设置





 

 



4.2.5 新建SSH密钥
  1.进入gitlab容器

docker exec -it gitlab /bin/sh


  2.生成密钥

ssh-keygen
一路回车就可以完成 创建SSH 密钥，可以看到 ~/.ssh目录下生成了 id_rsa 和 id_rsa.pub 两个文件

3.查看密钥

cat ~/.ssh/id_rsa.pub


4.gitlab配置密钥





 

 5.推出容器

exit


五、部署harbor（192.168.163.156） 
5.1 安装docker-compose
yum install -y docker-compose
配置docker加速器

mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload
systemctl restart docker
5.2 使用安装包安装 harbor
harbor安装包：harbor





 解压安装harbor

tar -zxvf harbor-offline-installer-v2.0.6.tgz
docker load -i harbor/harbor.v2.0.6.tar.gz


5.3 修改配置文件
cd harbor/
cp harbor.yml.tmpl harbor.yml
vim harbor.yml
 harbor.yml：设置IP和用户名密码 注释写https



5.4 执行./prepare && ./install.sh 初始化harbor
./prepare
./install.sh


[root@harbor ~]# cd harbor/
[root@harbor harbor]# cp harbor.yml.tmpl harbor.yml
[root@harbor harbor]# vim harbor.yml
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# ./prepare
prepare base dir is set to /root/harbor
ERROR:root:Error: The protocol is https but attribute ssl_cert is not set
Error happened in config validation...
[root@harbor harbor]# ls
common  common.sh  harbor.v2.0.6.tar.gz  harbor.yml  harbor.yml.tmpl  input  install.sh  LICENSE  prepare
[root@harbor harbor]# prepare
bash: prepare: 未找到命令...
[root@harbor harbor]# ./prepare
prepare base dir is set to /root/harbor
Error happened in config validation...
ERROR:root:Error: The protocol is https but attribute ssl_cert is not set
[root@harbor harbor]# vim harbor.yml
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# ./prepare
prepare base dir is set to /root/harbor
Error happened in config validation...
ERROR:root:Error: The protocol is https but attribute ssl_cert is not set
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# vim harbor.yml
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# clear
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# ./prepare
prepare base dir is set to /root/harbor
WARNING:root:WARNING: HTTP protocol is insecure. Harbor will deprecate http protocol in the future. Please make sure to upgrade to https
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# 
[root@harbor harbor]# ./install.sh

[Step 0]: checking if docker is installed ...

Note: docker version: 26.1.3

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.18.0

[Step 2]: loading Harbor images ...
Loaded image: goharbor/notary-server-photon:v2.0.6
Loaded image: goharbor/clair-photon:v2.0.6
Loaded image: goharbor/clair-adapter-photon:v2.0.6
Loaded image: goharbor/harbor-portal:v2.0.6
Loaded image: goharbor/harbor-core:v2.0.6
Loaded image: goharbor/harbor-db:v2.0.6
Loaded image: goharbor/harbor-jobservice:v2.0.6
Loaded image: goharbor/redis-photon:v2.0.6
Loaded image: goharbor/notary-signer-photon:v2.0.6
Loaded image: goharbor/harbor-log:v2.0.6
Loaded image: goharbor/harbor-registryctl:v2.0.6
Loaded image: goharbor/trivy-adapter-photon:v2.0.6
Loaded image: goharbor/chartmuseum-photon:v2.0.6
Loaded image: goharbor/prepare:v2.0.6
Loaded image: goharbor/nginx-photon:v2.0.6
Loaded image: goharbor/registry-photon:v2.0.6


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /root/harbor
WARNING:root:WARNING: HTTP protocol is insecure. Harbor will deprecate http protocol in the future. Please make sure to upgrade to https
Clearing the configuration file: /config/log/logrotate.conf
Clearing the configuration file: /config/log/rsyslog_docker.conf
Clearing the configuration file: /config/nginx/nginx.conf
Clearing the configuration file: /config/core/env
Clearing the configuration file: /config/core/app.conf
Clearing the configuration file: /config/registry/passwd
Clearing the configuration file: /config/registry/config.yml
Clearing the configuration file: /config/registryctl/env
Clearing the configuration file: /config/registryctl/config.yml
Clearing the configuration file: /config/db/env
Clearing the configuration file: /config/jobservice/env
Clearing the configuration file: /config/jobservice/config.yml
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Creating harbor-log ... done
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir


Creating harbor-db ... done
Creating harbor-core ... done
Creating network "harbor_harbor" with the default driver
Creating nginx ... done
Creating harbor-portal ... 
Creating redis ... 
Creating registryctl ... 
Creating registry ... 
Creating harbor-db ... 
Creating harbor-core ... 
Creating harbor-jobservice ... 
Creating nginx ... 
✔ ----Harbor has been installed and started successfully.----
[root@harbor harbor]# 

5.5 查看相关镜像
docker ps


5.6 访问测试
 http://192.168.163.156


5.7 CI服务器的docker配置（jenkins:192.168.163.155）
这里因为我们要在192.168.163.155(CI服务器)上push镜像到192.168.163.156(私仓)，所有需要修改CI服务器上的Docker配置。

cat /etc/docker/daemon.json
vim /etc/docker/daemon.json


[root@jenkins ~]# cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.163.156"]
}
[root@jenkins ~]# 
加载使其生效 

systemctl daemon-reload
systemctl restart docker
 CI机器简单测试一下

docker login 192.168.163.156
 测试上传镜像

docker tag beginor/gitlab-ce 192.168.163.156/library/gitlab-ce #beginor/gitlab-ce为本地镜像 可以自定义
docker images
docker push 192.168.163.156/library/gitlab-ce




5.8 CD服务器的docker配置（k8s-master、k8s-node1、k8s-node2） 
cat /etc/docker/daemon.json

vim /etc/docker/daemon.json
cat /etc/docker/daemon.json

加载使其生效 

systemctl daemon-reload
systemctl restart docker
CI机器简单测试一下

 

5.9 harbor开机启动
  编辑/lib/systemd/system/harbor.service

vim /lib/systemd/system/harbor.service
[Unit]
Description=Harbor
After=network.service docker.service systemd-networkd.service systemd-resolved.service 
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/bin/docker-compose -f /root/harbor/docker-compose.yml up
ExecStop=/usr/bin/docker-compose -f /root/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target


重载配置 

chmod 755 /lib/systemd/system/harbor.service
systemctl restart harbor
systemctl status harbor
systemctl enable harbor

 六、安装配置jenkins（jenkins:192.168.163.155）
6.1 镜像jenkins拉取
docker pull jenkins/jenkins:2.457


6.2 创建共享卷，修改所属组和用户,和容器里相同
mkdir /jenkins
chown 1000:1000 /jenkins
# 这里为什么要改成 1000，是因为容器里是以 jenkins 用户的身份去读写数据，而在容器里jenkins 的 uid 是 1000
6.3 创建 jenkins 容器 
docker run -dit -p 8080:8080 -p 50000:50000 --name jenkins  --privileged=true --restart=always -v /jenkins:/var/jenkins_home jenkins/jenkins:2.457
docker ps | grep jenkins
 访问jenkins  http://192.168.163.155:8080



关闭Jenkins容器，因为要修改配置

docker stop jenkins

6.4 配置jenkins环境
更换国内清华大学镜像,Jenkins下载插件特别慢，更换国内的清华源的镜像地址会快不少

cat /jenkins/hudson.model.UpdateCenter.xml


sed -i  's#updates.jenkins.io/update-center.json#mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json#g '  /jenkins/hudson.model.UpdateCenter.xml
cat /jenkins/hudson.model.UpdateCenter.xml  


 “https://www.google.com/” 替换为 “https://www.baidu.com/”

yum install -y jq
cat /jenkins/updates/default.json | jq '.connectionCheckUrl'
cat /jenkins/updates/default.json | jq 'keys'


替换

sed -i    s#https://www.google.com/#https://www.baidu.com/#g  /jenkins/updates/default.json
 重启docker，获取登录密匙

docker start jenkins
cat /jenkins/secrets/initialAdminPassword


 需要修改jenkins绑定的docker的启动参数，ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H fd:// --containerd=/run/containerd/containerd.sock

vim /lib/systemd/system/docker.service


修改镜像库启动参数后需要重启docker

systemctl daemon-reload
systemctl restart docker
6.5 配置Jenkins 安装插件










安装和配置docker插件

依此点击Manage Jenkins->Manage Plugins->AVAILABLE->Search 搜索docker、docker-build-step





 

  需要修改harbor(192.168.163.156)的docker的启动参数，ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H fd:// --containerd=/run/containerd/containerd.sock

vim /lib/systemd/system/docker.service
 修改镜像库启动参数后需要重启docker

systemctl daemon-reload
systemctl restart docker


  以上docker配置是为了Build / Publish Docker Image时使用，定义了用那个服务器的docker执行docker命令

jenkins 安全设置

后面 gitlab 要和 jenkins 进行联动，所以必须要需要对 jenkins 的安全做一些设置，依次点击 系统管理-全局安全配置-授权策略，勾选"匿名用户具有可读权限"

七、Jenkins发布打包成Docker的spring boot项目到K8s集群
7.1 创建spring boot 项目


pom.xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.7.RELEASE</version>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>demo</description>
    <properties>
        <java.version>1.8</java.version>
        <java.version>1.8</java.version>
        <!--换成你的harbor仓库地址-->
        <docker.repostory>192.168.163.156</docker.repostory>
        <!--换成你的仓库项目名称-->
        <docker.registry.name>demo</docker.registry.name>
        <docker.image.tag>0.0.1</docker.image.tag>
        <docker.maven.plugin.version>1.4.10</docker.maven.plugin.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>demo</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>${docker.maven.plugin.version}</version>
                <!--以下配置依赖docker环境 maven编译打包时会生成docker镜像 本地编译时先注释掉  上传git时在打开-->
                <executions>
                    <execution>
                        <id>default</id>
                        <goals>
                            <goal>build</goal>
                            <goal>push</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <contextDirectory>${project.basedir}</contextDirectory>
                    <useMavenSettingsForAuth>true</useMavenSettingsForAuth>
                    <repository>${docker.repostory}/${docker.registry.name}/${project.artifactId}</repository>
                    <!--换成你的harbor仓库账号密码-->
                    <username>admin</username>
                    <password>root123456</password>
                    <tag>${docker.image.tag}</tag>
                    <buildArgs>
                        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                    </buildArgs>
                </configuration>
            </plugin>
        </plugins>
        <resources>
            <!-- 指定 src/main/resources下所有文件及文件夹为资源文件 -->
            <resource>
                <directory>src/main/resources</directory>
                <targetPath>${project.build.directory}/classes</targetPath>
                <includes>
                    <include>**/*</include>
                </includes>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>
</project>

 创建测试接口



package com.example.demo.api;

import lombok.AllArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author lgp
 * @description: 测试PI
 * @date 2023/9/6
 */
    @RestController
    @AllArgsConstructor
    @RequestMapping("/api")
    public class testApi {

    @GetMapping("/test")
    public String test() {
        return "测试1112233552219122120911！！";
    }

}

启动项目



访问测试接口



添加Dockerfile文件

  该文件时docker创建容器的文件，名称不可改变



#添加依赖环境，前提是将Java8的Docker镜像从官方镜像仓库pull下来，然后上传到自己的Harbor私有仓库中
FROM 192.168.163.156/library/java:8
#指定镜像制作作者，可自己随意设置
MAINTAINER admin
#运行目录
VOLUME /tmp
#将本地的文件拷贝到容器，一般在项目的target目录下，要根据项目自己修改
ADD target/*jar app.jar
#启动容器后自动执行的命令
ENTRYPOINT [ "java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/app.jar" ]
添加demo.yaml文件

该文件是k8s创建pod的配置文件



apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  labels:
    app: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      imagePullSecrets: 				#使用imagePullSecrets参数指定镜像拉取秘钥
        - name: harbor-registry		#使用我们的secret，即harbor-registry
      containers:
      - name: demo
        image: 192.168.163.156/demo/demo:0.0.1
        ports:
        - containerPort: 8080   #修改成你项目的端口
        imagePullPolicy: Always #镜像拉取策略：每次都拉去最新的镜像

---
apiVersion: v1
kind: Service
metadata:
  name: demo
  labels:
    app: demo
spec:
  ports:
    - name: http
      port: 8080			#修改成你项目的端口
      nodePort: 30001      ##容器暴露的访问端口，要确定无冲突，否则需修改
  type: NodePort
  selector:
    app: demo

项目上传到gitlab 







7.2 拉取java8镜像 
 在jenkins服务器(192.168.163.155)上拉取java8镜像，然后推送至harbor仓库，一会打包构建docker镜像的时候会用到。

docker pull java:8
docker tag java:8 192.168.163.156/library/java:8
docker login 192.168.163.156
docker push 192.168.163.156/library/java:8


7.3 配置 jenkins容器到其他服务的ssh免密登录
jenkins执行构建命令时，需要ssh访问jenkins容器所在服务器（192.168.163.155）和k8s主节点服务器（192.168.163.151），所以需要配置ssh免密登录

在jenkins容器所在服务器（192.168.163.155）上进入jenkins容器

docker exec -it jenkins bash


生产密钥

ssh-keygen -t rsa
将公钥复制到jenkins容器所在服务器（192.168.163.155）

ssh-copy-id root@192.168.163.155


 将公钥复制到k8s主节点服务器（192.168.163.151)

7.4 k8s 创建私有镜像仓库Secret（k8s主节点服务器192.168.163.151）
k8s需要从harbor私有进行仓库拉取镜像时，需要创建Secret，配置harbor私有镜像仓库地址、账号密码

#docker-registry是关键字 harbor-registry是自定义的名称与demo.yaml中的imagePullSecrets.name值一致
kubectl -n default create secret docker-registry harbor-registry --docker-email=baidu.com@example --docker-username=root --docker-password=root123456 --docker-server=192.168.163.156	


kubectl get secret harbor-registry #查看secret


7.5 安装jdk和maven(jenkins:192.168.163.155)
 安装mvn

cd /usr/local/
wget https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar -zxvf apache-maven-3.6.3-bin.tar.gz
安装jdk

yum install -y java-1.8.0-openjdk-devel
配置环境变量

vim /etc/profile


MAVEN_HOME=/usr/local/apache-maven-3.6.3
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.412.b08-1.el7_9.x86_64  ##这里的路径注意使用你安装的jdk的版本号
CLASS_PATH=.:$JAVA_HOME/lib
PATH=$MAVEN_HOME/bin:$JAVA_HOME/bin:$PATH
 刷新环境变量

source /etc/profile
检查是否安装成功

mvn -v
java -version


7.6 使用jenkins构建发布项目






#使用jenkins宿主机的maven打包 /jenkins/workspace是jenkins 拉取git项目的目录 创建jenkins容器时已经做好了映射
ssh -tt root@192.168.163.155 'docker rmi 192.168.163.156/demo/demo:0.0.1
	/usr/local/apache-maven-3.6.3/bin/mvn -f /jenkins/workspace/demo/pom.xml clean install -Dmaven.test.skip=true'
ssh -tt root@192.168.163.155 'docker login 192.168.163.156
	docker push  192.168.163.156/demo/demo:0.0.1'
scp /var/jenkins_home/workspace/demo/demo.yaml root@192.168.163.151:/
ssh -tt root@192.168.163.151 '/usr/bin/kubectl delete -f /demo.yaml' #第一次执行时 由于没有该pod 会报错 所以第一次先注释掉
ssh -tt root@192.168.163.151 '/usr/bin/kubectl apply -f /demo.yaml'




镜像仓库



查看k8s pod

kubectl get pod --all-namespaces


访问应用：192.168.163.151:30001/api/test



方式二 构建 （比较麻烦 依赖于 6.5中的安装和配置docker插件）

  分为三步：

      1.执行 shell maven编译生成jar包



        2.Build / Publish Docker Image 生成镜像并上传至镜像仓库



   3.执行 shell 远程k8s更新pod



以上项目部署CI/CD 还需要手动在jenkins中构建项目，下面整合Jenkins和gitlab，使用gitlab的webhook功能，实现代码提交到gitlab后 自动构建部署项目









1、master节点
1.1、在master节点执行下面reset命令：
kubeadm reset

//过程会询问是否重置，输入y然后回车

1.2、手动清除配置信息，这一步很关键：
cd ~ 进入根目录

ll -a 查看是否存在.kube文件
rm -rf /root/.kube

systemctl restart docker ## 重启docker 
systemctl restart kubelet ## 重启kubelet

rm -rf /etc/cni/net.d
————————————————

                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。

原文链接：https://blog.csdn.net/weixin_43025151/article/details/135536233





### 2、work节点

#### 2.1、重置工作节点

`kubeadm reset`
//过程会询问是否重置，输入y然后回车

#### 2.2、手动删除目录

```
rm -rf /root/.kube
rm -rf /etc/cni/net.d
rm -rf /etc/kubernetes/*
```





### kubectl命令报错 “The connection to the server localhost:8080 was refused - did you specify the right host or port?”

解决方法：
1.将主节点（master）中的“/etc/[kubernetes](https://so.csdn.net/so/search?q=kubernetes&spm=1001.2101.3001.7020)/admin.conf”文件拷贝到从节点相同目录下
2.配置环境变量

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
1
```

3.立即生效

```bash
source ~/.bash_profile
```

常用命令

查看pod

kubectl get pods -A   /  kubectl get pods -All-namespaces #查看所有命令空间pod

kubectl get nodes #查看所有节点信息

kubectl logs  <yourpodname> -n <yournamespace>  #查看pod日志

kubectl logs <pod-name> -c <container-name> #查看pod中指定容器

kubectl logs <pod-name> --tail=20  #查看最后20行
kubectl logs <pod-name> --since=1h #查看最近一小时内的日志

kubectl describe pod <pod_name>  #描述pod

kubectl describe node <node_name>  #描述node

kubectl describe service <service_name> #描述service

kubectl cluster-info #查看集群信息

kubectl get <resource_type> #获取资源清单（pods、services、deployment)

kubectl create -f <filename> #创建资源

kubectl apply -f <filename> #更新资源

kubectl delete <resource_type> <resource_name> #删除资源

kubectl exec -it <pod_name> *-- /bin/bash* #进入pod交互式终端

kubectl scale --replicas=<num> deployment/<deployment_name>  #**扩展/缩放 Deployment**

**查看 ConfigMap 和 Secret：**

```shell
Copy Codekubectl get configmaps
kubectl get secrets
```

- **查看和修改节点标签：**

  ```shell
  Copy Codekubectl get nodes --show-labels
  kubectl label node <node_name> <label_key>=<label_value>
  ```

**查看 API 资源：**

```shell
Copy Codekubectl api-resources
```

**查看 API 版本：**

```shell
Copy Codekubectl api-versions
```





yaml解释

apiVersion: v1 # 资源版本
kind: Service # 资源类型
metadata: # 元数据
  name: gateway # 资源名称
  namespace: peric718 # 命名空间
spec: # 详细配置
  ports: # 端口配置
  - port: 10000 # 内部访问端口
    name: gateway # 端口名称
    nodePort: 30001 # 对外暴露端口
    selector: # 选择器，用于指定当前资源作用于哪些pod
    app: gateway
    type: NodePort # 类型为NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: peric718
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gateway
  template: # pod副本创建模板。属性和Pod的属性一样
    metadata:
      labels: # 给要创建的pod打标签
        app: gateway 
    spec:
      containers: # 使用的容器信息
      - name: gateway
        image: 192.168.1.180:5000/aengus/gateway:v1.6
        # 拉取镜像策略：本地镜像不存在，从仓库取。有Always：中从仓库取。Never：总从本地取。
        imagePullPolicy: IfNotPresent 
        ports:
        - protocol: TCP
          containerPort: 10000 # 镜像映射出的端口
          env: # 镜像环境变量设置
          - name: NACOS_URL # nacos注册网址
            value: 192.168.1.181:8848
          - name: NACOS_DISCOVERY_NAMESPACE # nacos服务注册命名空间的环境变量
            value: 2a20a28f-6345-4ec7-ac28-ae525796736a
          - name: NACOS_CONFIG_NAMESPACE # nacos服务配置命名空间的环境变量
            value: 2a20a28f-6345-4ec7-ac28-ae525796736a
          - name: NACOS_DISCOVERRY_IP
            valueFrom:
              fieldRef:
                 fieldPath: status.podIP
