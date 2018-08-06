---
title: linux启动过程
date: 2018-01-16 10:59:47
tags:
---
<font color=black>** Linux系统开机启动流程 **</font>
> 主板上的bios -> 第一块硬盘mbr -> grub -> 加载linux内核 -> init(1号进程) -> 读inittab, 进入相应的运行级别 ->
> rc.sysinit脚本 -> rc脚本 -> 进入rc[x].d, 运行S开头的服务启动脚本 -> rc.local ->
> /sbin/mingetty产生6个终端 -> /bin/login出现用户登录界面 -> 
> 输入正确的用户名和密码 -> /bin/bash

vi /etc/grub.conf
/etc/inittab

![linux-boot](/photo/linux-boot.jpeg)
