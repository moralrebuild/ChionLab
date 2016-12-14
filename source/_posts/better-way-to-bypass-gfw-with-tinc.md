---
title: tinc VPN+策略路由：Linux下更好的科学上网方式
date: 2016-12-12 13:31:59
tags:
- linux
- openwrt
categories:
- 学习笔记
- linux
---

tinc是一个[基于网状网络的VPN软件](https://www.tinc-vpn.org/)，使用tinc架设VPN，对于远程办公、文件传输等需求都是十分方便的。
而作为VPN，我们同样可以通过redirect gateway的方式来实现科学上网。本文将介绍通过使用tinc VPN和配置策略路由的方式，实现Linux平台下的科学上网（国内请求不走代理，国外请求走代理）。

本文教程以Arch Linux和OpenWRT为例，配置思路同样适用于其他Linux发行版。

相比于[使用shadowsocks进行科学上网](https://blog.chionlab.moe/2016/01/23/openwrt-bypass-gfw-solution/)，tinc+策略路由有以下优势：
* VPN是工作在IP层（网络层）的，因此可以实现对IP层packet进行代理，比如基于ICMP的ping和traceroute命令；而shadowsocks只能代理传输层的TCP和UDP请求。
* tincd进程和服务器间的通信是基于UDP的（对于屏蔽UDP的ISP，tinc会自动failover到TCP），而该socket数量是一直固定的，对于本地发出的需要代理的连接（无论是IP层还是传输层），可实现多路复用，大大提高性能。而shadowsocks对于每一个本地TCP连接，均需要向服务器建立一次新的TCP连接，速度十分有限。
* shadowsocks服务器有可能因为同时打开太多的TCP连接而拒绝请求，需要几分钟才能恢复。这种拒绝有可能是在ASP的防火墙上发生的，修改vps的内核参数如`file-max`等也不能解决。博主的VPS服务器上的shadowsocks服务每隔几天就会遇到一次这样的情况。而tinc因为基于多路复用，则没有这个问题。

条件
---
要架设可用于科学上网的tinc服务，你需要拥有：
* 一台境外的tinc服务器。你可能需要自行搭建一台tinc服务器，要求：tun/tap设备可用，操作系统为Linux发行版。
* 一台本地的Linux机器，可以是你的PC、软路由，或者是一台OpenWRT路由器，同样要求tun/tap可用。
* 确认服务器和本地机器上已安装iptables, iproute2, ipset。
* 本文非傻瓜教程，无法涵盖全部Linux发行版的操作，你需要熟悉自己所使用的发行版，如service，systemd等基本操作，遇到问题要知道如何排查。

tinc安装及配置
------------
你可以参考[Arch Linux wiki](https://wiki.archlinux.org/index.php/Tinc)来安装和配置tinc。这里博主简要介绍快速部署方法。

1. 在服务器和本地机器上安装tinc。
  ### Arch Linux:
  ```
  # pacman -Syu
  # pacman -S tinc
  ```
  ### OpenWRT:
  ```
  # opkg update
  # opkg install tinc
  ```
2. 在服务器和本地机器上创建tinc配置文件夹。请替换`myvpn`为你喜欢的vpn名称。
  ```
  # mkdir -p /etc/tinc/myvpn
  # mkdir /etc/tinc/myvpn/hosts
  ```
3. 服务端配置文件（在服务器上操作）：
  你可以将`alpha`替换为自己喜欢的服务器标识名，下同。
  /etc/tinc/*myvpn*/tinc.conf
  ```
  Name = alpha
  Device = /dev/net/tun
  ```
  /etc/tinc/*myvpn*/tinc-up
  ```
  #!/bin/sh
  ip link set $INTERFACE up
  ip addr add  192.168.100.1/32 dev $INTERFACE
  ip route add 192.168.100.0/24 dev $INTERFACE
  ```
  /etc/tinc/*myvpn*/tinc-down
  ```
  #!/bin/sh
  ip route del 192.168.100.0/24 dev $INTERFACE
  ip addr del 192.168.100.1/32 dev $INTERFACE
  ip link set $INTERFACE down
  ```
  添加脚本执行权限：
  ```
  # chmod +x /etc/tinc/myvpn/tinc-up
  # chmod +x /etc/tinc/myvpn/tinc-down
  ```
4. 本地机配置文件（在本地机器上操作）：
  你可以将`beta`替换为自己喜欢的客户端标识名，下同。
  /etc/tinc/*myvpn*/tinc.conf
  ```
  Name = beta
  Device = /dev/net/tun
  ConnectTo = alpha
  ```
  /etc/tinc/*myvpn*/tinc-up
  ```
  #!/bin/sh
  ip link set $INTERFACE up
  ip addr add  192.168.100.100/32 dev $INTERFACE
  ip route add 192.168.100.0/24 dev $INTERFACE
  ```
  /etc/tinc/*myvpn*/tinc-down
  ```
  #!/bin/sh
  ip route del 192.168.100.0/24 dev $INTERFACE
  ip addr del 192.168.100.100/32 dev $INTERFACE
  ip link set $INTERFACE down
  ```
  添加脚本执行权限：
  ```
  # chmod +x /etc/tinc/myvpn/tinc-up
  # chmod +x /etc/tinc/myvpn/tinc-down
  ```
5. 在服务器上建立host配置文件并生成密钥：
  /etc/tinc/*myvpn*/hosts/*alpha* 请将`10.0.0.1`替换为服务器的公网IP。
  ```
  Address = 10.0.0.1
  Port = 655
  Subnet = 0.0.0.0/0
  ```
  生成密钥：
  ```
  # tincd -n myvpn -K
  ```
6. 在本地机器上建立host配置文件并生成密钥：
  /etc/tinc/*myvpn*/hosts/*beta*
  ```
  Port = 655
  Subnet = 192.168.100.100/32
  ```
  生成密钥：
  ```
  # tincd -n myvpn -K
  ```
7. 在服务器和本地机上交换host配置文件：
  复制服务器上的`/etc/tinc/myvpn/hosts/alpha`到本地机器的`/etc/tinc/myvpn/hosts/alpha`。
  复制本地机器上的`/etc/tinc/myvpn/hosts/beta`到服务器上的`/etc/tinc/myvpn/hosts/beta`。
8. 在服务器和本地机上启动tinc服务：（先启动服务器上的）
  ### Arch Linux和其他基于systemd管理的linux发行版:
  ```
  # systemctl start tinc@myvpn.service
  ```
  ### OpenWRT和其他发行版：
  ```
  # tincd -n myvpn
  ```
9. 测试VPN是否正常：
  ```
  $ ifconfig #是否找到了myvpn接口？分配的IPv4地址是否正确？
  $ ping 192.168.100.1 #ping服务器
  $ ping 192.168.100.100 #ping客户端
  ```

配置服务器路由规则
---------------
以下操作在tinc服务器上进行：

1. 开启ip_forward：
  ### Arch Linux：
  ```
  # echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-ipforword.conf
  # sysctl --system
  ```

  ### 其他Linux发行版：
  ```
  # vim /etc/sysctl.conf
  ```
  将`net.ipv4.ip_forward=0`修改为`net.ipv4.ip_forward=1`。如果文件为空或没有这行，则添加一行`net.ipv4.ip_forward=1`即可。
  随后运行这条命令使配置生效：
  ```
  # sysctl -p
  ```

2. 开启masquerade：
  首先通过`ifconfig`命令（Arch Linux上使用`ip addr`），找出外网IP对应的接口名称。这里假设外网接口是`eth0`。
  运行命令：
  ```
  # iptables -t nat -A POSTROUTING -o eth0 -s 192.168.100.0/24 -j MASQUERADE
  ```
  你可以保存当前的iptables配置：
  ```
  # iptables-save > /etc/iptables/iptables.rules
  ```
  在Arch Linux上，需要开启`iptables.service`来实现重启后保留配置：
  ```
  # systemctl enable iptables.service
  ```
  在CentOS6上，需要开启`iptables`服务来实现重启后保留配置：
  ```
  # chkconfig iptables on
  ```

配置策略路由
----------
确认VPN架设成功，客户端和服务端能够互相ping通后，我们可以进行策略路由的配置了。
为了方便各位配置，博主已经写好了配置脚本。

以下操作在本地机器上进行：

1. 下载国内IP段文件，保存至`/etc/chn_route.list`：
  ```
  # wget https://raw.githubusercontent.com/Chion82/soft-router/master/tinc_proxy/chn_route.list -O /etc/chn_route.list
  ```

2. 下载策略路由初始化和停止脚本，保存至`/usr/bin/`目录：
  ```
  # cd /usr/bin
  # wget https://raw.githubusercontent.com/Chion82/soft-router/master/tinc_proxy/init_tinc_proxy -O init_tinc_proxy
  # wget https://raw.githubusercontent.com/Chion82/soft-router/master/tinc_proxy/stop_tinc_proxy -O stop_tinc_proxy
  # chmod +x init_tinc_proxy
  # chmod +x stop_tinc_proxy
  ```
3. 修改启动脚本：
  ```
  # vim /usr/bin/init_tinc_proxy
  ```
  将第2行`VPN_SERVER=XX.XX.XX.XX`的`XX.XX.XX.XX`修改为tinc服务器的外网IP地址。
  将第5行`VPN_INTERFACE=chionvpn`的`chionvpn`修改为`myvpn`，或者是刚才你自定义的vpn名称。
  如果你的Linux发行版不是Arch Linux，请删除最后这两行：
  ```
  #Set rp_filter
  echo 2 > /proc/sys/net/ipv4/conf/$VPN_INTERFACE/rp_filter
  ```
4. 修改停止脚本：
  ```
  # vim /usr/bin/stop_tinc_proxy
  ```
  将第2行`VPN_INTERFACE=chionvpn`的`chionvpn`修改为`myvpn`，或者是刚才你自定义的vpn名称。
  如果你的Linux发行版不是Arch Linux，请删除第7和第8行：
  ```
  #Restore rp_filter
  echo 1 > /proc/sys/net/ipv4/conf/$VPN_INTERFACE/rp_filter
  ```
5. 修改`tinc-up`文件：
  ```
  # vim /etc/tinc/myvpn/tinc-up
  ```
  在最后一行添加：
  ```
  /usr/bin/init_tinc_proxy &
  ```
6. 修改`tinc-down`文件：
  ```
  # vim /etc/tinc/myvpn/tinc-down
  ```
  在第一行`#!/bin/sh`
  下方插入一行：
  ```
  /usr/bin/stop_tinc_proxy
  ```

7. 重启tinc来测试配置是否正确：
  ### Arch Linux:
  ```
  # systemctl restart tinc@myvpn.service
  ```
  ### 其他Linux发行版：
  ```
  # tincd -n myvpn -k
  # tincd -n myvpn
  ```
  进行测试:
  ```
  $ ping 172.217.27.132
  ```
  如果能够ping通，说明以上配置正确。

配置ChinaDNS和dnsmasq
--------------------
至此，我们的VPN和策略路由已经配置完成。为了避免国内DNS污染，我们需要使用[ChinaDNS](https://github.com/shadowsocks/ChinaDNS)。ChinaDNS的配置方法与之前的[OpenWRT科学上网](https://blog.chionlab.moe/2016/01/23/openwrt-bypass-gfw-solution/)类似，唯一不同处是，这里的国外上游DNS服务器我们可以直接填写`8.8.8.8`，而不需要shadowsocks的`ss-tunnel`隧道。

以下操作在本地机器上进行：

1. 安装并配置ChinaDNS
  ### Arch Linux:
  ```
  # yaourt -S chinadns
  # cp /etc/chn_route.list /etc/chnroute.txt
  # system start chinadns.service
  ```

  ### OpenWRT:
  参照[OpenWRT科学上网 #安装ChinaDNS](https://blog.chionlab.moe/2016/01/23/openwrt-bypass-gfw-solution/#安装ChinaDNS)来安装ChinaDNS，然后执行：
  ```
  # cp /etc/chn_route.list /etc/chinadns_chnroute.txt
  ```
  进入OpenWRT管理网页，进入services->ChinaDNS，勾选`Enable`，中国路由表(CHNRoute File)填`/etc/chinadns_chnroute.txt`，设置`Upstream Servers`为：`114.114.114.114,8.8.8.8`。

2. 配置dnsmasq
  ### Arch Linux：
  ```
  # pacman -S dnsmasq
  ```
  修改`/etc/dnsmasq.conf`，清空文件并填入以下内容：
  ```
  listen-address=127.0.0.1  #如果机器(如软路由)绑定了静态IP，请在这里加上静态IP，以逗号分割

  no-resolv
  server=127.0.0.1#5353
  ```
  启动dnsmasq：
  ```
  # systemctl start dnsmasq.service
  ```
  修改本机的DNS配置，使其指向`127.0.0.1`。
  如果你的网络配置文件管理器是`netctl`，在对应的配置文件(位于`/etc/netctl/`下)中设置：
  ```
  DNS=('127.0.0.1')
  ```

  ### OpenWRT：
  进入网络(Network)->DHCP and DNS。
  将DNS转发(DNS forwardings)设置为`127.0.0.1#5353`。
  还要记得勾选“忽略解析文件”(ignore resolve file)。

3. 测试
  现在应该能够ping通谷歌域名了：
  ```
  $ dig www.google.com
  $ ping www.google.com
  ```

配置自启动
--------
如果需要在机器启动时自动开启科学上网，可按照以下步骤进行：

1. 自启动tinc服务，在服务器和本地机器上操作：
  ### Arch Linux:
  ```
  # systemctl enable tinc@myvpn.service
  ```

  ### 其他Linux发行版：
  在 `/etc/rc.local` 脚本文件最后添加一行：
  ```
  tincd -n myvpn
  ```
  当然，更好的方法是编写一个init服务脚本（位于`/etc/init.d/`）。

2. 自启动ChinaDNS服务，在本地机上操作：
  ### Arch Linux:
  ```
  # systemctl enable chinadns.service
  # systemctl enable dnsmasq.service
  ```

  ### OpenWRT:
  不需要特别设置，ChinaDNS和dnsmasq服务在安装后默认是自启动的。

至此，全部配置已经完成了，你现在可以上youtube看大新闻了。

策略路由原理及常见问题
------------------
1. `init_tinc_proxy`这个脚本都做了些什么？
  * 首先，读取`/etc/chn_route.list`文件，这个文件的内容是国内IPv4的CIDR地址段。创建一个ipset集合`chn_route`，将这些国内地址段写入该集合。
  * 添加一个路由表，id为`200`，该路由表接受全部IP段（`0.0.0.0/0`，或`default`），经由接口`myvpn`，网关（下一跳）是VPN服务器`192.168.100.1`。即执行：
  ```
  # ip route add default via 192.168.100.1 dev myvpn tabel 200
  ```
  * 添加一个路由规则，将MARK为`200`的IP报使用id为`200`的路由表进行路由。即执行：
  ```
  # ip rule add fwmark 200 table 200
  ```
  * 在iptables的mangle表增加一个自定义链`tinc_proxy`，并在该链中添加如下规则：
    目的地址在`BYPASS`指定的例外IP段中的packet，采取`RETURN`处理；
    目的地址在`chn_route`集合中的packet，采取`RETURN`处理；
    恢复CONNMARK的值到MARK（CONNMARK：用于跟踪一个连接的标记值）；
    对MARK的值为`0`的packet，设置其MARK为`200`；
    将MARK的值保存到CONNMARK；
  * 在mangle表的`PREROUTING`和`OUTPUT`链中插入自定义链`tinc_proxy`
  * 在nat表的`POSTROUTING`链中，对出口接口为`myvpn`的packet，采取`MASQUERADE`处理。

2. 为什么在Arch Linux下，需要将内核参数`/proc/sys/net/ipv4/conf/$VPN_INTERFACE/rp_filter`设为`2`？
  `rp_filter`是[Reverse Path Filtering (反向路径过滤)](http://tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.kernel.rpf.html)，其原理是：
  内核对目的地址为本机的每个IP报文，先检查其来源地址，然后根据本机路由表，查找到该来源地址的路由（即反向路径查找），若查找到的路由对应的接口与该报文实际到达所经过的接口不相符，则抛弃该包。显然，这对基于fwmark的策略路由是不适用的，因此需要关闭反向路径过滤功能。
  Arch Linux下默认使用严格的反向路径过滤策略，需要将该值设置为`2`。
  而其他发行版，只需要将该值保持为默认的`0`即可。

3. tinc VPN架设成功后，服务器和客户端能够互相ping通，但是无法经由服务器科学上网？
  请逐步排查，特别注意服务器的tinc host配置文件中，`Subnet`是否正确设置为`0.0.0.0`。
  参考[Example: redirecting the default gateway to a host on the VPN](https://www.tinc-vpn.org/examples/redirect-gateway/)将全部流量都经过VPN，看看能否正常访问外网。
