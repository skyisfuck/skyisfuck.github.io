---
title: 修改Linux网卡名(非Udev且无需重启)
date: 2018-02-24 15:50:57
tags: linux
---

#### <font color=green>1. 一次生效</font>

默认的情况下，不同版本的CentOS的网卡名字也不一样，例如CentOS6.x的网卡名为ethN，而CentOS7.x的网卡名为emN，所以在公司里，为了统一维护所有的机器，网卡名的设置也需要一致，否则在监控网络流量这一项，如果网卡名不一致，会带来额外的工作量

网络上设置网卡的大多数方案是修改Udev配置，并重启，这种方案太重了，对于已经运行的机器不能采用这样的方式，遂决定用英文关键字modify network interface name搜索，终于找到了以下答案，满足了自己的需求。

注意：对于远程联网的机器，要确保你有两个网卡：例如一个内网一个外网，否则你在关闭网卡时，会失去与主机的连接

```bash
ifconfig peth0 down  
ip link set peth0 name eth0  
ifconfig eth0 up  
```

原文链接：https://www.jianshu.com/p/5f6e6856c30e

#### <font color=green>2. 永久生效</font>

```bash

# 打开terminal，然后进入如下目录/etc/sysconfig/network-scripts/ 
cd /etc/sysconfig/network-scripts/ 

# 修改网卡名
mv ifcfg-eno16777736 ifcfg-eth0 

# 修改配置文件中的网卡名
sed -i 's/en016777736/eth0/g' ifcfg-eth0 

# 编辑/etc/default/grub并加入“net.ifnames=0 biosdevname=0 ”
sed -ri 's/(CMDLINE_LINUX=".*)"/\1 net.ifnames=0 biosdevname=0"/' /etc/default/grub

# 运行命令grub2-mkconfig -o /boot/grub2/grub.cfg 来重新生成GRUB配置并更新内核参数，重启生效
grub2-mkconfig -o /boot/grub2/grub.cfg
```
