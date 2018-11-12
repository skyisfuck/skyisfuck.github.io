---
title: Centos7安装Xen
date: 2018-11-09 19:00:23
tags: xen
---
#### 1、安装xen
```bash
# 1、下载安装xen源
yum install centos-release-xen -y

# 2、修改xen源
sed -i "s/enabled=1/enabled=0/g" /etc/yum.repos.d/CentOS-Xen.repo
sed -i "1,8s/\[centos-virt-xen.*\]/[centos-virt-xen]/" /etc/yum.repos.d/CentOS-Xen.repo

# 3、更新centos内核
yum --enablerepo=centos-virt-xen -y update kernel

# 4、安装xen
yum --enablerepo=centos-virt-xen -y install xen

# 5、修改Dom0的内存大小,内存太小，开不了机
vi /etc/default/grub
GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=4096M,max:4096M cpuinfo com1=115200,8n1 ....."

# 6、运行grub-bootxen.sh脚本，将xen添加到开机启动项中
/bin/grub-bootxen.sh

# 7、重启进入xen系统
reboot

# 8、查看xen的信息，检查是否安装成功
xl info

# 9、关闭selinux 和 NetworkManager
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
setenforce 0
systemctl disable NetworkManager
systemctl stop NetworkManager
```

#### 2、新建网桥设备
```bash
cd /etc/sysconfig/network-scripts/
cp ifcfg-eth0 ifcfg-br0 

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

#### 3、选用半虚拟化（PV）或 PV-on-HVM（PVHVM） 或 全虚拟化(HVM)
###### PV-on-HVM（PVHVM）
这里有 [PV-on-HVM](https://wiki.xen.org/wiki/PV_on_HVM) 的详细讨论，及[实例](https://wiki.xen.org/wiki/Xen_Linux_PV_on_HVM_drivers)。简单来说，你只需在上述 HVM 配置文件加入 xen_platform_pci=1 便能采用 PVHVM。

###### 半虚拟化（PV）
<font color=red>CentOS-6 及 CentOS-7 的内核缺省是不兼容半虚拟化的。你可以在 DomU 内采用 Dom0 内核，并于安装完成后通过修改配置文件启用 PV。

要是你有意这样做，以下是一些注意事项：</font>
```bash
# 1.你 不能 进行缺省安装，因为 Xen PV 机器的开机分区不能采用 XFS 文件系统……这是 CentOS 的缺省值。请在 CentOS 安装程序内手动创建一个 ext4 的 /boot 分区。
# 2.切勿在 CentOS 安装程序内以 LVM 分区作为开机分区……请采用标准分区。
# 3.要是你完成了上述事情，你便能创建一个 PV 配置文件，然后以 PV 模式从该分区开机。下面是一个供 test.pv DomU 用的样例，名为 
vi /etc/xen/config.d/test.pv.cfg
	name = "test"
	kernel = "/source/vmlinuz"
	ramdisk = "/source/initrd.img"
	#bootloader = "/usr/lib64/xen/bin/pygrub"
	memory = 2048
	vcpus = 1
	vif = [ 'bridge=br0' ]
	disk = [ '/xen/disk/test.qcow2,qcow2,xvda,rw' ]
	on_reboot = "destroy"
假如你遵从上述所有规则（以 xen dom0 内核替换 CentOS-7 安装的内核，/boot 禁用 xfs 文件系统，/boot 禁用 LVM，等……），
该机器将会以 PV 模式引导。当然，由于碟盘是共享的，你可以 同时 运行 HVM 及 PV 实例。你也可单独执行它们。
# 4.创建镜像硬盘
mkdir -p /xen/disk
qemu-img create -f qcow2 /xen/disk/test.qcow2 10G
# 5.要引导你的 PV DomU，请用此指令：
xl create -c /etc/xen/config.d/test.pv.cfg
# 6.根据提示安装完系统后，选择reboot
# 7.修改test.pv配置文件
vi /etc/xen/config.d/test.pv.cfg
	name = "test"
	#kernel = "/source/vmlinuz"
	#ramdisk = "/source/initrd.img"
	bootloader = "/usr/lib64/xen/bin/pygrub"
	memory = 2048
	vcpus = 1
	vif = [ 'bridge=br0' ]
	disk = [ '/xen/disk/test.qcow2,qcow2,xvda,rw' ]
	#on_reboot = "destroy"
# 8.启动进入系统
xl cretae -c /etc/xen/config.d/test.pv.cfg
```

###### 选用全虚拟化（HVM）
<font color=green>参数详解可以man 5 xl.cfg</font>
```bash
# 全虚拟化的硬盘不能用qemu-img创建，大小显示有误，用dd创建
dd if=/dev/zero of=/xen/disk/c6-x8664.img oflag=direct seek=10239 bs=1M count=1

# CentOS-6配置文件
vi /etc/xen/config.d/c6-x8664.hvm.cfg

builder = "hvm"
name = "c6-x8664.hvm"
memory = 4096
vcpus = 2
serial='pty'
vif = [ 'mac=00:16:3E:29:00:00,bridge=xenbr0' ]
disk = [ 'phy:/dev/vg_c6xendom0/c68-x8664-hvm,xvda,rw', 'file:/opt/isos/CentOS-6.8-x86_64-minimal.iso,xvdb:cdrom,r' ]
boot = "dc"
sdl = 0
vnc = 1
vnclisten  = "192.168.0.9"
vncdisplay = 0
vncpasswd  = "supersecret"
stdvga=1
videoram = 64

# CentOS-7配置文件
vi etc/xen/config.d/c7-x8664.hvm.cfg

builder = "hvm"
name = "c7-x8664.hvm"
memory = 4096
vcpus = 2
serial='pty'
vif = [ 'mac=00:16:3E:29:00:01,bridge=xenbr0' ]
disk = [ 'phy:/dev/vg_c6xendom0/c73-x8664-hvm,xvda,rw', 'file:/opt/isos/CentOS-7-x86_64-Minimal-1611.iso,xvdb:cdrom,r' ]
boot = "dc"
sdl = 0
vnc = 1
vnclisten  = "192.168.0.9"
vncdisplay = 1
vncpasswd  = "supersecret"
stdvga=1
videoram = 64

```
* vnc listen 的 IP 地址是网桥的 IP，在此样例中为 192.168.0.9
* boot 可用磁盘机（a）、硬盘（c）、网络（n）或光盘（d）……因此 dc 代表以光盘然后以硬盘开机。完成安装后我们会将它改为 boot="c"
* 我们可利用 vnc 客端连接到 192.168.0.9:5900（centos-6）及 192.168.0.9:5901（centos-7）。
* 步骤同半虚拟化，安装完系统后把配置文件中的boot = "dc"改成boot = "c"

#### 4、使用通用工具libvirtd来管理xen
* virsh (CLI)
* virt-manager (GUI) --->可在xshell中使用，需要安装xmanager
* virt-install (CLI) -->创建虚拟机

###### virt-install安装系统
```bash
# 1、安装Libvirt
yum --enablerepo=centos-virt-xen -y install libvirt libvirt-daemon-xen virt-install virt-manager
# 2、启动Libvirtd
systemctl start libvirtd
systemctl enable libvirtd
# 3、创建虚拟机镜像目录
mkdir -p /xen/disk
# 4、开始安装虚拟机
virt-install --connect xen:/// --paravirt --name centos7 --ram 4096 --disk path=/xen/disk/centos7.img,size=10 --vcpus 2 --os-type linux --os-variant rhel7 --network bridge=br0 --graphics none --location '/home/centos/' --extra-args 'text console=com1 utf8 console=hvc0'
# --location 后面接的是iso镜像挂载点(可本地/nfs/http/ftp)
```

###### virt-manager安装系统
```bash
# 1、在xshell命令行中输入virt-manager & 开启图形化安装系统
virt-manager &
```



#### 5、启动自制的linux系统 busybox(pv)
在配置busybox前，讲一下，相关的配置知识
```bash

[root@node1 ~]# uname -r
4.9.75-30.el6.x86_64
可以查看当前系统的内核版本

[root@node1 ~]# xl list
Name                                        ID   Mem VCPUs  State   Time(s)
Domain-0                                     0  1024     1     r-----     547.3

#xl list 可以查看显示Domain的相关信息

关于xen虚拟机的状态
    r: running
    b: 阻塞
    p: 暂停
    s: 停止
    c: 崩溃
    d: dying,正在关闭中的过程中

xm与xl启动DomU使用的配置文件略有不同； 
对于xl而言，其创建DomU使用的配置指令可通过man xl.cfg获取    
常用指令：       
    1.name : 域名称，必须是唯一的         
    2.builder： 指明虚拟机的类型，generic表示pv,hvm表示hvm            
    3.vcpus: 虚拟CPU个数
	maxvcpus: 最大虚拟cpu个数
	cpus: vcpu可运行于其上物理CPU列表             
    4.memory=MBYTES: 内存大小
	maxmem=MBYTES: 可以使用的最大内存空间              
    5.on_poweroff: 指明关机时采取的action
	destroy ,restart,preserve               
    6.on_reboot="ACTION" : 指明重启时采取的action               
    7.on_crash="ACTION": 虚拟机崩溃时采取的action            
    8.disk=[ "DISK_SPEC_STRING", "DISK_SPEC_STRING", ...]:  指明磁盘设备，列表
    9.vif=[ "NET_SPEC_STRING", "NET_SPEC_STRING", ...]： 指明网络接口，列表           
    10.vfb=[ "VFB_SPEC_STRING", "VFB_SPEC_STRING", ...]： 指明virtual frame buffer 显示图形界面，列表；
    11.pci=[ "PCI_SPEC_STRING", "PCI_SPEC_STRING", ... ]： 指明pci设备接口

PV模式专用指令：           
    kernel="PATHNAME": 内核文件路径；          
    ramdisk="PATHNAME"：为kernel指定内核提供的ramdisk文件路径            
    root="STRING"： 指明根文件系统          
    extra="STRING" ：额外传递给内核引导时使用的参数 
    bootloader="PROGRAM"：如果DomU使用自己的kernel及ramdisk,此时需要一个Dom0中的应用程序来实现bootloader功能；

磁盘参数指定方式：           
    [<target>,[<format>,[vdev],[<access>]]]]

	tartet表示磁盘映像文件或设备文件路径
	format表示磁盘格式，如果映像文件，有多种 格式，如raw,qcow2
	vdev 此设备在DomU被识别为硬件设备类型，支持hd,sd[x],xvd[x]
	access访问权限 
	    ro,r : 只读
	    rw,w ：读写                
    disk=["/images/xen/linux.img,raw,xvda,rw",]

使用qemu-img管理磁盘映像
create [-f fmt] [-o options] filename [size]        
可创建sparse稀疏格式的磁盘映像文件
```
###### 示例：
```bash
(1) 准备磁盘映像文件
        qemu-img create -f raw -o size=2G  /images/xen/busybox.img
        mke2fs -t ext2 busybox.img
        mount -o /images/xen/busybox.img /mnt/

(2) 提供根文件系统
        编译busybox,并复制到busybox.img映像中
            yum groupinstall -y "Development Tools" "Server Platfrom Development"
            yum install -y glibc-static    #为了方便移值，安装glibc-static，不让它依赖其它库
            wget https://busybox.net/downloads/busybox-1.22.1.tar.bz2
            tar xf busybox-1.22.1.tar.bz2
            cd busybox-1.22.1
            make menuconfig
                #   Busybox Settings  --->   Build Options  --->  [*] Build BusyBox as a static binary (no shared libs)
            make && make install

        cp -a _install/* /mnt/
        mkdir proc sys dev etc var boot home

        到这里可以试一下

        cd
        chroot /mnt /bin/sh

(3) 提供DomU配置文件
        cd /boot/
        ln -s vmlinuz-2.6.32-431.el6.x86_64 vmlinuz
        ln -s initramfs-2.6.32-431.el6.x86_64.img initramfs.img
        cd /etc/xen
        cp xlexample.pvlinux busybox
        vim busybox
        更改如下项：
                name = "busybox-001"
                kernel = "/boot/vmlinuz"
                ramdisk = "/boot/initramfs.img"
                extra = "selinux=0 init=/bin/sh"
                memory = 256
                vcpus = 1
                disk = [ '/images/xen/busybox.img,raw,xvda,rw' ]
                root = "/dev/xvda ro"

(4) 启动实例：
                xl -v create busybox -n
                xl list
                xl -v create busybox
                xl console busybox-001
                退出ctrl+]
```
