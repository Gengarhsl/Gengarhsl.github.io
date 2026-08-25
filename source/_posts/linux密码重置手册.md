---
title: linux重置密码手册
date: 2026-03-05
---



```
# Linux 忘记 Root 密码解决方案与安全防护指南（运维手册版）

本文系统整理 **Linux 忘记 root 密码的解决方案**、**GRUB 启动原理**、**不同发行版重置方式**以及**生产环境安全防护措施**。  
适用于 Linux 运维工程师、开发人员和系统管理员。

适用系统包括：

- **:contentReference[oaicite:0]{index=0}**
- **:contentReference[oaicite:1]{index=1}**
- **:contentReference[oaicite:2]{index=2}**
- **:contentReference[oaicite:3]{index=3}**
- **:contentReference[oaicite:4]{index=4}**

---

# 一、Linux 启动流程简述

理解 Linux 启动流程有助于理解为什么可以通过 GRUB 重置 root 密码。

Linux 启动过程大致如下：
```

BIOS / UEFI
 ↓
 BootLoader（GRUB）
 ↓
 Linux Kernel
 ↓
 init / systemd
 ↓
 用户空间

```
关键点：
```

GRUB 可以修改 Linux 内核启动参数

```
如果我们修改启动参数，就可以：

- 进入单用户模式
- 进入 emergency shell
- 修改系统文件
- 重置 root 密码

---

# 二、通过 GRUB 重置 root 密码（推荐方法）

适用于：

- CentOS 7+
- Rocky Linux
- AlmaLinux

## 1 重启服务器

```bash
reboot
```

------

## 2 进入 GRUB 菜单

系统启动时按：

```
ESC
或
Shift
```

进入 GRUB 界面。

------

## 3 编辑启动项

选中当前系统启动项，按：

```
e
```

进入编辑模式。

找到这一行：

```
linux16 /vmlinuz-xxx root=/dev/mapper/... ro
```

将：

```
ro
```

改为：

```
rw init=/sysroot/bin/sh
```

示例：

```
linux16 /vmlinuz-xxx root=/dev/mapper/centos-root rw init=/sysroot/bin/sh
```

------

## 4 启动系统

按：

```
Ctrl + X
```

进入紧急 shell。

------

## 5 切换到系统环境

```
chroot /sysroot
```

------

## 6 修改 root 密码

```
passwd root
```

输入新密码即可。

示例：

```
Changing password for user root.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

------

## 7 修复 SELinux 标签（重要）

如果系统开启了 SELinux，需要执行：

```
touch /.autorelabel
```

否则系统可能无法正常登录。

------

## 8 重启系统

```
exit
exit
reboot
```

系统启动后即可使用新密码登录。

------

# 三、使用 rd.break 重置密码（CentOS 官方方式）

该方法是 **Red Hat 推荐的方式。

## 1 进入 GRUB 编辑界面

按：

```
e
```

找到：

```
linux16
```

在行尾添加：

```
rd.break
```

示例：

```
linux16 /vmlinuz-xxx root=/dev/mapper/centos-root ro rd.break
```

------

## 2 启动系统

按：

```
Ctrl + X
```

进入 emergency mode。

------

## 3 重新挂载系统分区

```
mount -o remount,rw /sysroot
```

------

## 4 切换到系统环境

```
chroot /sysroot
```

------

## 5 修改 root 密码

```
passwd root
```

------

## 6 SELinux 修复

```
touch /.autorelabel
```

------

## 7 重启系统

```
exit
exit
reboot
```

------

# 四、Ubuntu / Debian 重置 root 密码

适用于：

- **Ubuntu**
- **Debian**

## 1 进入 GRUB

开机按：

```
ESC
```

------

## 2 编辑启动项

找到：

```
linux /boot/vmlinuz
```

在行尾添加：

```
init=/bin/bash
```

------

## 3 启动系统

按：

```
Ctrl + X
```

进入 root shell。

------

## 4 挂载为读写

```
mount -o remount,rw /
```

------

## 5 修改密码

```
passwd root
```

------

## 6 重启系统

```
exec /sbin/init
```

或

```
reboot -f
```

------

# 五、为什么可以重置 root 密码？

核心原因：

```
GRUB 可以修改 Linux 启动参数
```

只要满足以下条件：

- 能访问服务器控制台
- 能修改 GRUB 启动项
- 没有设置 GRUB 密码

就可以重置 root 密码。

------

# 六、生产环境安全防护

为了防止非法修改 root 密码，需要采取安全措施。

------

## 1 设置 GRUB 密码

```
grub2-setpassword
```

这样进入 GRUB 编辑界面需要密码。

------

## 2 禁止外部设备启动

在 BIOS 中：

- 禁止 U 盘启动
- 禁止光驱启动

------

## 3 设置 BIOS 密码

防止攻击者修改启动顺序。

------

## 4 使用磁盘加密

Linux 推荐使用：

**LUKS**

磁盘加密后：

- 没有密码无法挂载磁盘
- 无法修改系统文件
- 无法重置 root 密码

------

## 5 机房物理安全

服务器安全原则：

```
物理安全 > 系统安全
```

如果攻击者拥有：

- 服务器物理访问权限
- 控制台权限

任何系统都可能被破解。

------

# 七、云服务器为什么不能这样操作

在云环境中，例如：

- **Amazon Web Services**
- **Alibaba Cloud**
- **Tencent Cloud**

用户通常无法：

- 修改 GRUB
- 控制启动过程
- 直接访问磁盘

因此只能通过云平台提供的 **重置密码功能**。

------

# 八、运维最佳实践

建议：

```
1 定期备份服务器
2 禁止 root 远程登录
3 使用 SSH Key 登录
4 开启日志审计
5 设置 GRUB 密码
6 保护服务器物理安全
```

------

# 九、总结

Linux 忘记 root 密码时，可以通过 **GRUB 修改启动参数**进入系统重置密码。

常见方法：

| 方法              | 推荐程度 |
| ----------------- | -------- |
| GRUB init=/bin/sh | 常用     |
| rd.break          | 官方推荐 |
| 单用户模式        | 较旧     |

但前提是：

```
必须拥有服务器控制权限
```

生产环境必须做好：

- GRUB 安全
- BIOS 安全
- 磁盘加密
- 物理安全