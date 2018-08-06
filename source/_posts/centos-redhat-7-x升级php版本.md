---
title: centos/redhat 7.x升级php版本
date: 2018-01-15 02:55:57
tags:
---
**我的是centos 7.1系统**

<font color=green>**卸载重装方式:(安全)**</font>
<font color=black>**CentOS/RHEL 7.x:**</font>
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

<font color=black>**CentOS/RHEL 6.x:**</font>
rpm -Uvh https://mirror.webtatic.com/yum/el6/latest.rpm
Now you can install PHP 5.6 (along with an opcode cache) by doing:

yum install php56w php56w-opcache

<font color=red>**直接升级:(有风险)**</font>
yum install yum-plugin-replace
yum replace php-common --replace-with=php56w-common
yum install php56w-opcache
