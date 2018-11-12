---
title: centos7_kvm
date: 2018-11-12 18:46:08
tags: kvm
---
#### 安装KVM环境
```bash
# 1、检测是否支持KVM
[root@localhost ~]# cat /proc/cpuinfo | egrep "vmx|svm"
# 2、关闭SELinux
[root@localhost ~]# sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
setenforce 0
# 3、安装kvm 管理工具
[root@localhost ~]# yum -y install qemu-kvm libvirt virt-install virt-manager virt-viewer

# qemu-kvm: KVM模块

# virt-install: 包含工具（virt-install，virt-clone和virt-image），
# 用于安装和克隆虚拟机使用libvirt。 它完全支持paravirtulized半虚拟化和全虚拟化。 
# 支持的虚拟机管理程序是Xen，qemu（QEMU）和kvm

# libvirt: 虚拟管理模块
# virt-manager: 图形界面管理虚拟机

# 4、重启宿主机，以便加载 kvm 模块
# ------------------------
[root@localhost ~]# reboot

# 5、查看KVM模块是否被正确加载
# ------------------------
[root@localhost ~]# lsmod | grep kvm

kvm_intel             162153  0
kvm                   525259  1 kvm_intel
# 6、开启libvirtd服务，并且设置其开机自动启动
[root@localhost ~]# systemctl start libvirtd.service
[root@localhost ~]# systemctl enable libvirtd.service
```

#### 安装虚拟机
安装前要设置环境语言为英文LANG="en_US.UTF-8"，如果是中文的话某些版本可能会报错。CentOS 7 在这里修改 /etc/locale.conf。
kvm创建虚拟机，特别注意.iso镜像文件一定放到/home 或者根目录重新创建目录，不然会因为权限报错，无法创建虚拟机。
```bash
# 字符界面安装
virt-install \
--virt-type=kvm \
--name=centos78 \
--vcpus=2 \
--memory=4096 \
--location=/tmp/CentOS-7-x86_64-Minimal-1511.iso \
--disk path=/home/vms/centos78.qcow2,size=40,format=qcow2 \
--network bridge=br0 \
--graphics none \
--extra-args='console=ttyS0' \
--force

# --location 里面可以是本地，也可以是nfs，http
# 如 --location=http://mirrors.aliyun.com/centos/7/os/x86_64/

# 图形化 vnc 安装
virt-install \
--virt-type=kvm \
-n centos7 \
--vcpu 2 \
-r 2048 \
--disk /kvm/disk/centos.qcow2,format=qcow2,size=10 \
--network bridge=br0 \
--cdrom /opt/centos.iso \
--vnc --vncport=5910 --vnclisten=0.0.0.0
```

#### 连接虚拟机
通过 virsh console <虚拟机名称> 命令来连接虚拟机
```bash
# 查看虚拟机
[root@localhost ~]# virsh list              # 查看在运行的虚拟机
[root@localhost ~]# virsh list –all         # 查看所有虚拟机

 Id    Name                           State
----------------------------------------------------
 7     centos7                        running

#连接虚拟机
virsh console centos7
```

#### 配置宿主机网络
1.KVM 虚拟机是基于 NAT 的网络配置；
2.只有同一宿主机的虚拟键之间可以互相访问，跨宿主机是不能访问；
3.虚拟机需要和宿主机配置成桥接模式，以便虚拟机可以在局域网内可见；

###### Bridge模式配置
Bridge方式即虚拟网桥的网络连接方式，是客户机和子网里面的机器能够互相通信。可以使虚拟机成为网络中具有独立IP的主机。桥接网络（也叫 物理设备共享）被用作把一个物理设备复制到一台虚拟机。网桥多用作高级设置，特别是主机多个网络接口的情况。
```
┌─────────────────────────┐      ┌─────────────────┐
│          HOST           │      │Virtual Machine 1│
│ ┌──────┐      ┌───────┐ │      │    ┌──────┐     │
│ │ br0  │──┬───│ vnet0 │─│─ ─ ─ │    │ br0  │     │
│ └──────┘  │   └───────┘ │      │    └──────┘     │
│     │     │             │      └─────────────────┘
│     │     │   ┌───────┐ │      ┌─────────────────┐
│ ┌──────┐  └───│ vnet1 │─│─     │Virtual Machine 2│
│ │ eno0 │      └───────┘ │ │    │    ┌──────┐     │
│ └──────┘                │  ─ ─ │    │ br0  │     │
│ ┌──────┐                │      │    └──────┘     │
│ │ eno1 │                │      └─────────────────┘
│ └──────┘                │
└─────────────────────────┘

# 新建网桥设备

[root@localhost ~]# cd /etc/sysconfig/network-scripts/
[root@localhost ~]# cp ifcfg-eth0 ifcfg-br0 

vi ifcfg-eth0
	TYPE=Ethernet
	BOOTPROTO=none
	NAME=eth0
	DEVICE=eth0
	ONBOOT=yes
	BRIDGE=br0
	#IPADDR=192.168.5.137
	#PREFIX=24
	#GATEWAY=192.168.5.2
	#DNS1=192.168.5.2

vi ifcfg-br0
	TYPE=Bridge
	BOOTPROTO=none
	NAME=br0
	DEVICE=br0
	ONBOOT=yes
	IPADDR=192.168.5.137
	PREFIX=24
	GATEWAY=192.168.5.2
	DNS1=192.168.5.2
	STP=yes

systemctl restart network

[root@localhost ~]# brctl show 
bridge name	bridge id		STP enabled	interfaces
br0		8000.000c29db5f75	yes		eth0
```
###### NAT模式
NAT(Network Address Translation网络地址翻译)，NAT方式是kvm安装后的默认方式。它支持主机与虚拟机的互访，同时也支持虚拟机访问互联网，但不支持外界访问虚拟机。
```bash
[root@localhost ~]# virsh net-list --all

 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     no            no
# default是宿主机安装虚拟机支持模块的时候自动安装的。
[root@localhost ~]# ip a;                                                                                         
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP group default qlen 1000
    link/ether 00:0c:29:e3:3d:4a brd ff:ff:ff:ff:ff:ff
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:e3:3d:4a brd ff:ff:ff:ff:ff:ff
    inet 192.168.160.172/24 brd 192.168.160.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fee3:3d4a/64 scope link 
       valid_lft forever preferred_lft forever
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:4d:79:b9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
5: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:4d:79:b9 brd ff:ff:ff:ff:ff:ff
6: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:4f:3a:89 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe4f:3a89/64 scope link 
       valid_lft forever preferred_lft forever
# 其中virbr0是由宿主机虚拟机支持模块安装时产生的虚拟网络接口，也是一个switch和bridge，负责把内容分发到各虚拟机。几个虚拟机管理模块产生的接口关系如下图:
┌───────────────────────┐                      
│         HOST          │                      
│ ┌──────┐              │   ┌─────────────────┐
│ │ br0  │─┬──────┐     │   │Virtual Machine 1│
│ └──────┘ │      │     │   │   ┌──────┐      │
│     │    │  ┌───────┐ │ ─ │   │ br0  │      │
│     │    │  │ vnet0 │─│┘  │   └──────┘      │
│ ┌──────┐ │  └───────┘ │   └─────────────────┘
│ │virbr0│ │  ┌───────┐ │   ┌─────────────────┐
│ │ -nic │ └──│ vnet1 │─│┐  │Virtual Machine 2│
│ └──────┘    └───────┘ │   │                 │
│ ┌──────┐              │└ ─│   ┌──────┐      │
│ │ eth0 │              │   │   │ br0  │      │
│ └──────┘              │   │   └──────┘      │
└───────────────────────┘   └─────────────────┘
# 从图上可以看出，虚拟接口和物理接口之间没有连接关系，所以虚拟机只能在通过虚拟的网络访问外部世界，无法从网络上定位和访问虚拟主机。
# virbr0是一个桥接器，接收所有到网络192.168.122.*的内容。从下面命令可以验证：
[root@localhost ~]# brctl show                                                                                    
bridge name     bridge id               STP enabled     interfaces
br0             8000.000c29e33d4a       yes             eth0
                                                        vnet0
virbr0          8000.5254004d79b9       yes             virbr0-nic
[root@localhost ~]# ip r                                                                                          
default via 192.168.160.2 dev br0 proto static metric 425 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 
192.168.160.0/24 dev br0 proto kernel scope link src 192.168.160.172 metric 425
# 同时，虚拟机支持模块会修改iptables规则，通过命令可以查看：
[root@localhost ~]# iptables -t nat -L -nv
[root@localhost ~]# iptables -t filter -L -nv
# 如果没有default的话，或者需要扩展自己的虚拟网络，可以使用命令重新安装NAT。
[root@localhost ~]# virsh net-define /usr/share/libvirt/networks/default.xml
# 此命令定义一个虚拟网络，default.xml的内容：
<network>
  <name>default</name>
  <bridge name="virbr0" />
  <forward/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254" />
    </dhcp>
  </ip>
</network>
# 也可以修改xml，创建自己的虚拟网络。
# 标记为自动启动：

[root@localhost ~]# virsh net-autostart default
# Network default marked as autostarted
启动网络：
[root@localhost ~]# virsh net-start default
# Network default started

网络启动后可以用命令brctl show 查看和验证。
修改vi /etc/sysctl.conf中参数，允许ip转发，CentOS7是在vi /usr/lib/sysctl.d/00-system.conf 这里面修改

net.ipv4.ip_forward=1

通过 sysctl -p 查看修改结果
 ```
###### 自定义NAT模式
```bash
# 创建名为management的NAT网络，vi /usr/share/libvirt/networks/management.xml
<network>
  <name>management</name>
  <bridge name="virbr1"/>
  <forward/>
  <ip address="192.168.123.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.123.2" end="192.168.123.254"/>
    </dhcp>
  </ip>
</network>
# 启用新建的NAT网络
[root@localhost ~]# virsh net-define /usr/share/libvirt/networks/management.xml
[root@localhost ~]# virsh net-start management
[root@localhost ~]# virsh net-autostart management
# 验证

[root@localhost ~]# brctl show
bridge name bridge id   STP enabled interfaces
br0   8000.3863bb44cf6c no    eno1
							  vnet0
virbr0    8000.525400193f0f yes   virbr0-nic
virbr1    8000.52540027f0ba yes   virbr1-nic

[root@localhost ~]# virsh net-list --all
  Name                 State      Autostart     Persistent
----------------------------------------------------------
  default              active     no            no
  management           active     yes           yes
```
###### 退出虚拟机
```bash
exit # 退出系统到登录界面
Ctrl+5 # 从虚拟机登录页面，退出到宿主机命令行页面
Ctrl+] # 或者下面
```
###### 修改虚拟机配置信息
```bash
# 直接通过vim命令修改
[root@localhost ~]# vim  /etc/libvirt/qemu/centos72.xml
# 通过virsh命令修改
[root@localhost ~]# virsh edit centos72
```
###### 克隆虚拟机
```bash
# 暂停原始虚拟机
virsh shutdown centos72
[root@localhost ~]# virt-clone -o centos72 -n centos.112 -f /home/vms/centos.112.qcow2 -m 00:00:00:00:00:01
[root@localhost ~]# virt-clone -o centos88 -n centos.112 --file /home/vms/centos.112.qcow2 --nonsparse

virt-clone 参数介绍

    --version 查看版本。
    -h，--help 查看帮助信息。
    --connect=URI 连接到虚拟机管理程序 libvirt 的URI。
    -o 原始虚拟机名称 原始虚拟机名称，必须为关闭或者暂停状态。
    -n 新虚拟机名称 –name 新虚拟机名称。
    --auto-clone 从原来的虚拟机配置自动生成克隆名称和存储路径。
    -u NEW_UUID, --uuid=NEW_UUID 克隆虚拟机的新的UUID，默认值是一个随机生成的UUID。
    -m NEW_MAC, --mac=NEW_MAC 设置一个新的mac地址，默认为随机生成 MAC。
    -f NEW_DISKFILE, --file=NEW_DISKFILE 为新客户机使用新的磁盘镜像文件地址。
    --force-copy=TARGET 强制复制设备。
    --nonsparse 不使用稀疏文件复制磁盘映像。
```
###### 通过镜像创建虚拟机
```bash
---------1、创建虚拟机镜像文件--------
# 复制第一次安装的干净系统镜像，作为基础镜像文件，
# 后面创建虚拟机使用这个基础镜像
[root@localhost ~]# cp /home/vms/centos.88.qcow2 /home/vms/centos7.base.qcow2

# 使用基础镜像文件，创建新的虚拟机镜像
[root@localhost ~]# cp /home/vms/centos7.base.qcow2 /home/vms/centos7.113.qcow2

---------2、创建虚拟机配置文件--------
# 复制第一次安装的干净系统镜像，作为基础配置文件。
[root@localhost ~]# virsh dumpxml centos.88 > /home/vms/centos7.base.xml

# 使用基础虚拟机镜像配置文件，创建新的虚拟机配置文件
[root@localhost ~]# cp /home/vms/centos7.base.xml /home/vms/centos7.113.xml

# 编辑新虚拟机配置文件
[root@localhost ~]# vi /home/vms/centos7.113.xml
# 主要是修改虚拟机文件名，UUID，镜像地址和网卡地址，其中 UUID 在 Linux 下可以使用 uuidgen 命令生成
<domain type='kvm'>
  <name>centos7.113</name>
  <uuid>1e86167a-33a9-4ce8-929e-58013fbf9122</uuid>
  <devices>
    <disk type='file' device='disk'>
      <source file='/home/vms/centos7.113.img'/>
    </disk>
    <interface type='bridge'>
      <mac address='00:00:00:00:00:04'/>
    </interface>    
    </devices>
</domain>
[root@localhost ~]# virsh define /home/vms/centos7.113.xml
# Domain centos.113 defined from /home/vms/centos7.113.xml

# 创建磁盘
[root@localhost ~]# mkdir /home/vms
# 创建 guest 所需的磁盘
# create 表示创建，-f qcow2 表示创建一个格式为 qcow2 的磁盘， 
# /home/vms/centos78.qcow2 表示创建的磁盘名称及磁盘文件，40G 表示该磁盘可用大小。
[root@localhost ~]# qemu-img create -f qcow2 -o preallocation=metadata /home/vms/centos78.qcow2 40G
```
###### 常用命令说明
**virt-install**
```bash
# 常用参数说明
–name指定虚拟机名称
–memory分配内存大小。
–vcpus分配CPU核心数，最大与实体机CPU核心数相同
–disk指定虚拟机镜像，size指定分配大小单位为G。
–network网络类型，此处用的是默认，一般用的应该是bridge桥接。
–accelerate加速
–cdrom指定安装镜像iso
–vnc启用VNC远程管理，一般安装系统都要启用。
–vncport指定VNC监控端口，默认端口为5900，端口不能重复。
–vnclisten指定VNC绑定IP，默认绑定127.0.0.1，这里改为0.0.0.0。
–os-type=linux,windows
–os-variant=rhel6

--name      指定虚拟机名称
--ram       虚拟机内存大小，以 MB 为单位
--vcpus     分配CPU核心数，最大与实体机CPU核心数相同
–-vnc       启用VNC远程管理，一般安装系统都要启用。
–-vncport   指定VNC监控端口，默认端口为5900，端口不能重复。
–-vnclisten  指定VNC绑定IP，默认绑定127.0.0.1，这里改为0.0.0.0。
--network   虚拟机网络配置
  # 其中子选项，bridge=br0 指定桥接网卡的名称。

–os-type=linux,windows
–os-variant=rhel7.2

--disk 指定虚拟机的磁盘存储位置
  # size，初始磁盘大小，以 GB 为单位。

--location 指定安装介质路径，如光盘镜像的文件路径。
--graphics 图形化显示配置
  # 全新安装虚拟机过程中可能会有很多交互操作，比如设置语言，初始化 root 密码等等。
  # graphics 选项的作用就是配置图形化的交互方式，可以使用 vnc（一种远程桌面软件）进行链接。
  # 我们这列使用命令行的方式安装，所以这里要设置为 none，但要通过 --extra-args 选项指定终端信息，
  # 这样才能将安装过程中的交互信息输出到当前控制台。
--extra-args 根据不同的安装方式设置不同的额外选项
```
**virsh**
```bash
virsh list                 # 查看在运行的虚拟机
virsh dumpxml vm-name      # 查看kvm虚拟机配置文件
virsh start vm-name        # 启动kvm虚拟机
virsh shutdown vm-name     # 正常关机

virsh destroy vm-name      # 非正常关机，强制关闭虚拟机（相当于物理机直接拔掉电源）
virsh undefine vm-name     # 删除vm的配置文件

ls  /etc/libvirt/qemu
# 查看删除结果，Centos-6.6的配置文件被删除，但磁盘文件不会被删除

virsh define file-name.xml # 根据配置文件定义虚拟机
virsh suspend vm-name      # 挂起，终止
virsh resumed vm-name      # 恢复被挂起的虚拟机
virsh autostart vm-name    # 开机自启动vm
virsh console <虚拟机名称>   # 连接虚拟机
```
**错误解决**
```bash
console test
Connected to domain test
Escape character is ^]
# 如果出现上面字符串使用 CTRL+Shift+5 CTRL+Shift+]
```
