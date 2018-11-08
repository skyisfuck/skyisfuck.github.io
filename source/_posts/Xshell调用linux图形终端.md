---
title: Xshell调用linux图形终端
date: 2018-11-09 02:44:30
tags: linux
---
#### 1、设置Xshell
![Xshell设置](/photo/xshell设置.png)

#### 2、在操作系统安装如下包
```bash
yum install xorg-x11-font* xorg-x11-xauth -y
```

---
#### 使用xshll连接linux时，报错`WARNING! The remote SSH server rejected X11 forwarding request.`
```
进行步骤2或者关闭步骤1中的转发x11连接到(X):
```

