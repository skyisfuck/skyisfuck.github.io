---
title: Centos7-自动登录
date: 2018-09-11 10:50:01
tags: centos
---
#### 1、在工作中测试中经常要登录用户名、密码，因此在网上查了一下怎么开机自动登录 root 用户

```bash

# 打开terminal，然后进入如下目录//etc/systemd/system/getty.target.wants
cd /etc/systemd/system/getty.target.wants

#在配置文件中ExecStart选项中加入 --autologin root
sed -ri 's/(ExecStart=.*)/\1 --autologin root/' getty\@tty1.service 

#重启机器ok
reboot

```
