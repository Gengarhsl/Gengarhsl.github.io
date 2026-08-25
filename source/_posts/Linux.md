---
title: Linux笔记
date: 
---

# 安装虚拟化组

### 安装图形化界面虚拟机管理工具
``` bash
$ yum groupinstall 'Virtualization Host'
```
### 启动虚拟化服务
``` bash
$ yum install virt-manager
```
### Web管理界面
``` bash
$ systemctl start libvirtd.service
$ systemclt start cockpit.service
```




# 配置网卡
``` bash
$ nmtui 图形化工具
$ /etc/sysconfig/network-scripts/ifcfg-enp1s0 网卡配置文件
$ nmcli con modify enp1s0 ipv4.method me
$ nmcli con up enp1s0启用网卡配置
```

w查看登录信息
Ss -ant查看链接信息
Ss -at
Ss -atu
``` bash
$ /etc/resolv.conf  dns记录文件 网卡会覆盖
$ /etc/hostname修改主机名
$ /etc/hosts dns解析文件
```
route -n查询网关
主机名 计算机名
Fqdn  web-server.abc.com
Netbiosname.  Web-server
nmcli con show 查看
nmcli device status 查看网卡状态

# 安装软件包 chrony

### 查看时钟同步状态
``` bash
$ chronyc sources -v
```
### 搭建内网Ntp服务器
``` bash
$ vim /etc/chrony.conf 修改配置文件
$ firewall-cmd. --add-service=ntp 防火墙放行ntp服务
```
修改时区
timedatectl. Set-timezone. Asia/Shanghai
           List-timezone  查询
Timedatectl. Set-ntp. true/false 开启互联网校准
Date -s. 修改时间

Yum list 查询仓库里所有软件包及其状态

rpm -i 安装 v查看安装信息 h显示进度
rpm -q查询 
rpm -e删除
过滤
| grep 名字
查询某个文件属于某个rpm包
rpm -qa 
Net-tools
Bash-c*
Vim-e*

# 磁盘分区

\1. /boot. 1g
\2. Swap.  2 Ram
\3. /.  All

# ssh配置
``` bash
$ /etc/ssh/sshd_config
$ Port 22
$ Permitrootlogin yes 允许 root远程登录 no禁止
```

### 关闭selinux安全插件
``` bash
setenforce 0
```
### 使用秘钥登陆
客户端生成秘钥文件  公钥加密（非对称加密）公钥，私钥


客户端
``` bash
$ ssh-keygen 生成公钥
上传公钥到目标服务器的root账号
$ ssh-copy-id 主机
```

scp 文件 主机:位置
SCP -R 文件夹

# 

# 搭建DHCP服务器

### 安装dhcp软件包
``` bash
$ yum install dhcp-server
修改配置文件 
$ vim /etc/DHCP/dhcpd.conf
```
10yy复制
P粘贴

查看磁盘 
fdisk -l
lsblk
分区工具
fdisk交互式分区
n p w
parted 可交互

Parted 磁盘
Print打印查看
Mkpart
Name 名字
类型 
起始 上一个磁盘的终止
终止 起始加分区大小
quit 退出
非交互式
parted 磁盘 mkpart 名字 起始 终止


RMB
Parted 磁盘 mklabel msdos
Parted 磁盘 rm 分区 删除分区
Parted 磁盘 mkpart 主/扩 格式 起始 终止
对分区进行格式化

ext4
Xfs
mkfs -t 格式 磁盘
mkfs.格式 磁盘
如果硬盘是Ada 插入U盘 设备名为 sdb
mount | tail -n 数字 查看文件的最后几行
lsblk -f 查看挂载
开机自动挂载
/etc/fstab
设备名建议使用UUID
blkid 磁盘 查看UUID
lsblk _f 查看UUID
udevadm settle 更新文件 rhel8
Parts -a 磁盘 更新文件. rhel7之前

扩充交换分区
mkswap 磁盘 格式化为swap类型
挂载为了swap空间
swapon 磁盘
swapon -s 查看
swapoff 卸载挂载点

dd if=/dev/zero of=/ten/swap1 bs=1M count=2048 复制磁盘块为文件
格式化
挂载



Man 命令 查看命令手册 

# 搭建ftp服务器

``` bash
$ yum install vsftpd -y
 修改FTP配置文件
$ /etc/vsftpd/vsftpd.conf
$anonymous_enable=yes
 启动FTP服务 开机自启
$systemctl enable vsftpd.service --now
 Ftp 默认家目录 匿名访问
$ /var/ftp
默认命令行下是主动FTP 21 20建立传输
被动高端端口
限制FTP被动端口范围
Pasv_min_port=10000
Pasv_max_port=11000
Ss -ant查看 
```
# LVM  逻辑卷管理

传统的磁盘分区  分区固定后不能更改

### 将磁盘，分区转换为PV逻辑卷
``` bash
$ PVcreate 磁盘
$ Pvs. Pvdisplay 磁盘 查看信息
```
### 将物理卷组合成卷组 VG
``` bash
$ vgcreate 名字 磁盘
$ vgs. Vgdispaly 磁盘 查看
$ VGremove 名称 删除卷组
$ -s定义块大小

Pe的尺寸可以更改
```
### 创建lv 逻辑卷
``` bash
$ lvcreate -n 名字 -L 大小/-l 多少个块 卷组
$ lvextend -L +尺寸 逻辑卷 扩展逻辑卷
将扩展的lv与挂载点 同步尺寸
xfs_growfs 挂载点
Resize2fs 逻辑卷 ext4同步
Vgextend vg pv扩展卷组
```


# dns服务器

### 安装软件包
``` bash
$ yum install bind. bind-utils 
$ /etc/named.conf
UDP53客户机找服务
Tcp53服务器找服务器

添加区域
zone “ABC.com”IN{
      type master;
      File "abc.com.zone";

};

$ /var/named/
```



mount -o loop 文件 挂载本地文件
＞覆盖 ＞＞重定向
开机自动挂载 defaults.loop
给web启用 ssl 使用数字证书
Yum install mod_ssl

# 搭建多个web站点 虚拟主机

Apache 配置文件 /etc/httpd
conf 主配置文件 conf.d辅配置文件
/usr/share/doc/httpd配置文件模板
httpd-vhosts.conf 虚拟主机模板

动态网站
asp asp.net php jsp Python 等开发
小型机 IBM Aix
      Sun solris Oracle 甲骨文
      Hp hp-unix

挂接数据库 mysql mariadb
mariadb-server
如果需要，可以做安全初始化配置
MySQL_secure_installation
清除历史命令 history -c
unzip 文件 -d 目录

Chmod o+w config/ -R
data/ -R
uc_client 
Uc_server
软件包 php-mysqlnd
Php-xml



firewall-cmd.  --list-all  查看防火墙规则
firewall-cmd --list-all
firewalld.richlanguage富规则
firewall-cmd --add-service=服务  --permanent. 写入配置文件



