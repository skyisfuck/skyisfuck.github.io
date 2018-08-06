---
title: iptables分析
date: 2018-01-16 11:25:48
tags:
---
### 一、iptables基本概念
匹配（match）：符合指定的条件，比如指定的 IP 地址和端口。
丢弃（drop）：当一个包到达时，简单地丢弃，不做其它任何处理。
接受（accept）：和丢弃相反，接受这个包，让这个包通过。
拒绝（reject）：和丢弃相似，但它还会向发送这个包的源主机发送错误消息。这个错误消息可以指定，也可以自动产生。
目标（target）：指定的动作，说明如何处理一个包，比如：丢弃，接受，或拒绝。
跳转（jump）：和目标类似，不过它指定的不是一个具体的动作，而是另一个链，表示要跳转到那个链上。
规则（rule）：一个或多个匹配及其对应的目标。
链（chain）：每条链都包含有一系列的规则，这些规则会被依次应用到每个遍历该链的数据包上。每个链都有各自专门的用途， 这一点我们下面会详细讨论。
表（table）：每个表包含有若干个不同的链，比如 filter 表默认包含有 INPUT，FORWARD，OUTPUT 三个链。iptables有四个表，分别是：raw，nat，mangle和filter，每个表都有自己专门的用处，比如最常用filter表就是专门用来做包过滤的，而nat 表是专门用来做NAT的。
策略（police）：我们在这里提到的策略是指，对于 iptables 中某条链，当所有规则都匹配不成功时其默认的处理动作。
连接跟踪（connection track）：又称为动态过滤，可以根据指定连接的状态进行一些适当的过滤，是一个很强大的功能，但同时也比较消耗内存资源。

### 二、iptables的数据包流程
![iptables数据包流程](/photo/iptables-1.png)
上图表达了数据包经过iptables的基本流程，从图中可将数据包报文的处理过程分为三种类型。

#### 1) 目的为本机的报文
报文以本机为目的地址时，其经过iptables的过程为：
1.     数据包从network到网卡
2.     网卡接收到数据包后，进入raw表的PREROUTING链。这个链的作用是在连接跟踪之前处理报文，能够设置一条连接不被连接跟踪处理。(注：不要在raw表上添加其他规则)
3.     如果设置了连接跟踪，则在这条连接上处理。
4.     经过raw处理后，进入mangle表的PREROUTING链。这个链主要是用来修改报文的TOS、TTL以及给报文设置特殊的MARK。(注：通常mangle表以给报文设置MARK为主，在这个表里面，千万不要做过滤/NAT/伪装这类的事情)
5.     进入nat表的PREROUTING链。这个链主要用来处理 DNAT，应该避免在这条链里面做过滤，否则可能造成有些报文会漏掉。(注：它只用来完成源/目的地址的转换)
6.     进入路由决定数据包的处理。例如决定报文是上本机还是转发或者其他地方。(注：此处假设报文交给本机处理)
7.     进入 mangle 表的 INPUT 链。在把报文实际送给本机前，路由之后，我们可以再次修改报文。
8.     进入 filter 表的 INPUT 链。在这儿我们对所有送往本机的报文进行过滤，要注意所有收到的并且目的地址为本机的报文都会经过这个链，而不管哪个接口进来的或者它往哪儿去。
9.     进过规则过滤，报文交由本地进程或者应用程序处理，例如服务器或者客户端程序。
#### 2） 本地主机发出报文
数据包由本机发出时，其经过iptables的过程为：
1.     本地进程或者应用程序（例如服务器或者客户端程序）发出数据包。
2.     路由选择，用哪个源地址以及从哪个接口上出去，当然还有其他一些必要的信息。
3.     进入 raw 表的 OUTPUT 链。 这里是能够在连接跟踪生效前处理报文的点，在这可以标记某个连接不被连接跟踪处理。
4.     连接跟踪对本地的数据包进行处理。
5.     进入 mangle 表的 OUTPUT 链，在这里我们可以修改数据包，但不要做过滤(以避免副作用)。
6.     进入 nat 表的 OUTPUT 链，可以对防火墙自己发出的数据做目的NAT(DNAT) 。
7.     进入 filter 表的 OUTPUT 链，可以对本地出去的数据包进行过滤。
8.     再次进行路由决定，因为前面的 mangle 和 nat 表可能修改了报文的路由信息。
9.     进入 mangle 表的 POSTROUTING 链。这条链可能被两种报文遍历，一种是转发的报文，另外就是本机产生的报文。
10.        进入 nat 表的 POSTROUTING 链。在这我们做源 NAT（SNAT），建议你不要在这做报文过滤，因为有副作用。即使你设置了默认策略，一些报文也有可能溜过去。
11.        进入出去的网络接口。
#### 3）转发报文
报文经过iptables进入转发的过程为：
1.     数据包从network到网卡
2.     网卡接收到数据包后，进入raw表的PREROUTING链。这个链的作用是在连接跟踪之前处理报文，能够设置一条连接不被连接跟踪处理。(注：不要在raw表上添加其他规则)
3.     如果设置了连接跟踪，则在这条连接上处理。
4.     经过raw处理后，进入mangle表的PREROUTING链。这个链主要是用来修改报文的TOS、TTL以及给报文设置特殊的MARK。(注：通常mangle表以给报文设置MARK为主，在这个表里面，千万不要做过滤/NAT/伪装这类的事情)
5.     进入nat表的PREROUTING链。这个链主要用来处理 DNAT，应该避免在这条链里面做过滤，否则可能造成有些报文会漏掉。(注：它只用来完成源/目的地址的转换)
6.     进入路由决定数据包的处理。例如决定报文是上本机还是转发或者其他地方。(注：此处假设报文进行转发)
7.     进入 mangle 表的 FORWARD 链，这里也比较特殊，这是在第一次路由决定之后，在进行最后的路由决定之前，我们仍然可以对数据包进行某些修改。
8.     进入 filter 表的 FORWARD 链，在这里我们可以对所有转发的数据包进行过滤。需要注意的是：经过这里的数据包是转发的，方向是双向的。
9.     进入 mangle 表的 POSTROUTING 链，到这里已经做完了所有的路由决定，但数据包仍然在本地主机，我们还可以进行某些修改。
10.        进入 nat 表的 POSTROUTING 链，在这里一般都是用来做 SNAT ，不要在这里进行过滤。
11.        进入出去的网络接口。
### 三、iptables的表、链、和规则
![iptables的表、链和规则](/photo/iptables-2.png)
<font color=black>规则（rules）</font>其实就是网络管理员预定义的条件，规则一般的定义为“如果数据包头符合这样的条件，就这样处理这个数据包”。规则存储在内核空间的信息包过滤表中，这些规则分别指定了源地址、目的地址、传输协议（如TCP、UDP、ICMP）和服务类型（如HTTP、FTP和SMTP）等。当数据包与规则匹配时，iptables就根据规则所定义的方法来处理这些数据包，如放行（accept）、拒绝（reject）和丢弃（drop）等。配置防火墙的主要工作就是添加、修改和删除这些规则。
<font color=black>链（chains）</font>是数据包传播的路径，每一条链其实就是众多规则中的一个检查清单，每一条链中可以有一条或数条规则。当一个数据包到达一个链时，iptables就会从链中第一条规则开始检查，看该数据包是否满足规则所定义的条件。如果满足，系统就会根据该条规则所定义的方法处理该数据包；否则iptables将继续检查下一条规则，如果该数据包不符合链中任一条规则，iptables就会根据该链预先定义的默认策略来处理数据包。
<font color=black>表（tables）</font>提供特定的功能，iptables内置了4个表，即filter表、nat表、mangle表和raw表，分别用于实现包过滤，网络地址转换、包重构(修改)和数据跟踪处理。
       从上图中可看出，表与链的关系，raw、mangle、nat和filter四个表所含的链是不同的：
raw表有PREROUTING链和OUTPUT链；
mangle表有PREROUTING链、POSTROUTING链、INPUT链、OUTPUT链和FORWARD链；
nat表有PREROUTING链、POSTROUTING链和OUTPUT链四个链；
filter表有INPUT链、FORWARD链和OUTPUT链。

### 四、常用iptables过滤规则
#### 1） iptables规则添加的命令
<font color=green size=4>iptables [-t table] command [match] [target/jump]</font>
以图形简略表示如下，
![命令图](/photo/iptables-3.png)
iptables命令中的command参数，match参数以及target/jump参数的具体含义可参考linux中的man手册或iptables指南。
#### 2) 常用iptables过滤规则
<font color=#006000>** 1.删除现有规则 **</font>
iptables -F    或者  iptables --flush
<font color=#006000>** 2.     设置默认链策略 **</font>
ptables的filter表中有三种链：INPUT, FORWARD和OUTPUT。默认的链策略是ACCEPT，可以将它们设置成DROP，命令如下：
iptables -P INPUT DROP              修改INPUT链的默认策略为DROP
iptables -P FORWARD DROP       修改FORWARD链
iptables -P OUTPUT DROP           修改OUTPUT链
<font color=#006000>** 3 .     屏蔽指定的IP地址 **</font>
以下规则将屏蔽BLOCK_THIS_IP所指定的IP地址访问本地主机：
BLOCK_THIS_IP="x.x.x.x"
iptables -A INPUT -i eth0 -s "$BLOCK_THIS_IP" -j DROP
(或者仅屏蔽来自该IP的TCP数据包）
iptables -A INPUT -i eth0 -p tcp -s "$BLOCK_THIS_IP" -j DROP
<font color=#006000>** 4.     屏蔽来自外部的ping **</font>
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
iptables -A OUTPUT -p icmp --icmp-type echo-reply -j DROP
<font color=#006000>** 5.     屏蔽从本机ping外部主机 **</font>
iptables -A OUTPUT -p icmp --icmp-type echo-request -j DROP
iptables -A INPUT -p icmp --icmp-type echo-reply -j DROP
<font color=#006000>** 6.     屏蔽环回(loopback)访问 **</font>
iptables -A INPUT -i lo -j DROP
iptables -A OUTPUT -o lo -j DROP
<font color=#006000>** 7.     允许所有SSH连接请求 **</font>
本规则允许所有来自外部的SSH连接请求，也就是说，只允许进入eth0接口，并且目的端口为22的数据包。
iptables -A INPUT -i eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 8.     允许从本地发起的SSH连接 **</font>
本规则允许本机发起SSH连接：
iptables -A OUTPUT -o eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 9.     仅允许来自指定网络的SSH连接请求 **</font>
以下规则仅允许来自172.16.132.0/24的网络：
iptables -A INPUT -i eth0 -p tcp -s 172.16.132.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 10.   仅允许从本地发起到指定网络的SSH连接请求 **</font>
以下规则仅允许从本地主机连接到172.16.1132.0/24的网络：
iptables -A OUTPUT -o eth0 -p tcp -d 172.16.132.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 11.   允许HTTP/HTTPS连接请求 **</font>
\# 1.允许HTTP连接：80端口
iptables -A INPUT -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
 
\# 2.允许HTTPS连接：443端口
iptables -A INPUT -i eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 12.   允许从本地发起HTTPS连接 **</font>
本规则可以允许用户从本地主机发起HTTPS连接，从而访问Internet。
iptables -A OUTPUT -o eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 13.   -m multiport：指定多个端口 **</font>
通过指定-m multiport选项，可以在一条规则中同时允许SSH、HTTP、HTTPS连接：
iptables -A INPUT -i eth0 -p tcp -m multiport --dports 22,80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp -m multiport --sports 22,80,443 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 14.   允许IMAP与IMAPS **</font>
IMAP：143
iptables -A INPUT -i eth0 -p tcp --dport 143 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 143 -m state --state ESTABLISHED -j ACCEPT
 
\# IMAPS：993
iptables -A INPUT -i eth0 -p tcp --dport 993 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 993 -m state --state ESTABLISHED -j ACCEPT
15.   允许POP3与POP3S
\# POP3：110
iptables -A INPUT -i eth0 -p tcp --dport 110 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 110 -m state --state ESTABLISHED -j ACCEPT
 
\# POP3S：995
iptables -A INPUT -i eth0 -p tcp --dport 995 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 995 -m state --state ESTABLISHED -j ACCEPT
<font color=#006000>** 16.   防止DoS攻击 **</font>
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
-m limit: 启用limit扩展
–limit 25/minute: 允许最多每分钟25个连接
–limit-burst 100: 当达到100个连接后，才启用上述25/minute限制
<font color=#006000>** 17.   允许路由 **</font>
如果本地主机有两块网卡，一块连接内网(eth0)，一块连接外网(eth1)，那么可以使用下面的规则将eth0的数据路由到eht1：
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
<font color=#006000>** 18.   DNAT与端口转发 **</font>
以下规则将会把来自422端口的流量转发到22端口，这意味着来自422端口的SSH连接请求与来自22端口的请求等效。
\# 1.启用DNAT转发
iptables -t nat -A PREROUTING -p tcp -d 172.16.132.17 --dport 422 -j DNAT --to-destination 172.16.132.17:22
 
\# 2.允许连接到422端口的请求
iptables -A INPUT -i eth0 -p tcp --dport 422 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 422 -m state --state ESTABLISHED -j ACCEPT
 
假设现在外网网关是xxx.xxx.xxx.xxx，那么把HTTP请求转发到内部的某一台计算机的规则如下：
iptables -t nat -A PREROUTING -p tcp -i eth0 -d xxx.xxx.xxx.xxx --dport 8888 -j DNAT --to 192.168.0.2:80
iptables -A FORWARD -p tcp -i eth0 -d 192.168.0.2 --dport 80 -j ACCEPT
<font color=#006000>** 19.   SNAT与MASQUERADE **</font>
如下命令表示把所有192.168.1.0网段的数据包SNAT成172.132.16.99的ip然后发出去：
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j snat --to-source 172.132.16.99
 
对于snat，不管是几个地址，必须明确的指定要snat的IP。假如我们的计算机使用ADSL拨号方式上网，那么外网IP是动态的，这时候我们可以考虑使用MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.1.0/255.255.255.0 -o eth0 -j MASQUERADE
<font color=#006000>** 20.   自定义的链 **</font>
记录丢弃的数据包：
\# 1.新建名为LOGGING的链
iptables -N LOGGING
 
\# 2.将所有来自INPUT链中的数据包跳转到LOGGING链中
iptables -A INPUT -j LOGGING
 
\# 3.指定自定义的日志前缀"IPTables Packet Dropped: "
iptables -A LOGGING -m limit --limit 2/min -j LOG --log-prefix "IPTables Packet Dropped: " --log-level 7
 
\# 4.丢弃这些数据包
iptables -A LOGGING -j DROP
 
<font color=#006000>** 21.   IP范围匹配(IP range match options) **</font>
源：  iptables -A INPUT -p tcp -m iprange --src-range 192.168.1.13-192.168.2.19  -j DROP
目的  iptables -A INPUT -p tcp -m iprange --dst-range 192.168.1.13-192.168.2.19  -j DROP
 
<font color=#006000>** 22.   MAC匹配 **</font>
iptables -A INPUT -m mac --mac-source 00:00:00:00:00:01 -j DROP
