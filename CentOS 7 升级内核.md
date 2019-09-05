# CentOS 7 升级内核

[TOC]

RHEL7 及 CentOS7 现在使用的 kernel 仍然是 3.10 版本，如果需要使用新版 kernel 才有的功能，便需要升级 kernel。除了手动编译 kernel 外，以下提供使用 yum 命令，通过 ELRepo repository 升级 kernel 的方法。



## 一、启用 ELRepo repository

ELRepo 仓库是基于社区的用于企业级 Linux 仓库，提供对 RedHat Enterprise（RHEL） 和 其它基于 RHEL 的 Linux 发行版（CentOS、Scientific、Fedora 等）的支持。

ELRepo 聚焦于和硬件相关的软件包，包括文件系统驱动、显卡驱动、网络驱动、声卡驱动和摄像头驱动等。

```shell
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
```

## 二、查看可升级的 kernel 版本

启用 ELRepo repository 后，可以执行以下指令查看可升级的 kernel 版本

```shell
# yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
```

## 三、安装 kernel

```shell
# yum --enablerepo=elrepo-kernel install kernel-ml
```

## 四、设置开机从新内核启动

```shell
# grub2-set-default 0
```

## 五、重启机器

```shell
# reboot
```

## 六、安装内核源码文件

在升级内核完成并重启机器后执行，也可以不用安装。

在使用源码编译安装软件时，可能会使用到，所以还是安装上去好一点。

```shell
# yum --enablerepo=elrepo-kernel install kernel-ml-devel-$(uname -r) kernel-ml-headers-$(uname -r)
```

