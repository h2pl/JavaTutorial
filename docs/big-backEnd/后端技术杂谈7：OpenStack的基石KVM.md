# Table of Contents

* [[Qemu，KVM，Virsh傻傻的分不清](https://www.cnblogs.com/popsuper1982/p/8522535.html)](#[qemu，kvm，virsh傻傻的分不清]httpswwwcnblogscompopsuper1982p8522535html)
  * [ Kvm虚拟化技术实践](# kvm虚拟化技术实践)
    * [VMware虚拟机支持Kvm虚拟化技术？](#vmware虚拟机支持kvm虚拟化技术？)
    * [安装Kvm虚拟化软件](#安装kvm虚拟化软件)
* [ifconfig virbr0virbr0    Link encap:Ethernet  HWaddr 52:54:00:D7:23:AD            inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1          RX packets:0 errors:0 dropped:0 overruns:0 frame:0          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0          collisions:0 txqueuelen:0           RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)](#ifconfig-virbr0virbr0----link-encapethernet--hwaddr-525400d723ad------------inet-addr1921681221--bcast192168122255--mask2552552550----------up-broadcast-running-multicast--mtu1500--metric1----------rx-packets0-errors0-dropped0-overruns0-frame0----------tx-packets0-errors0-dropped0-overruns0-carrier0----------collisions0-txqueuelen0-----------rx-bytes0-00-b--tx-bytes0-00-b)
* [brctl showbridge name     bridge id               STP enabled     interfacesvirbr0          8000.525400d723ad       yes             virbr0-nic](#brctl-showbridge-name-----bridge-id---------------stp-enabled-----interfacesvirbr0----------8000525400d723ad-------yes-------------virbr0-nic)
* [iptables -nvL -t natChain PREROUTING (policy ACCEPT 304 packets, 38526 bytes) pkts bytes target     prot opt in     out     source               destination          Chain POSTROUTING (policy ACCEPT 7 packets, 483 bytes) pkts bytes target     prot opt in     out     source               destination             0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24    masq ports: 1024-65535     0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24    masq ports: 1024-65535     0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24     Chain OUTPUT (policy ACCEPT 7 packets, 483 bytes) pkts bytes target     prot opt in     out     source               destination](#iptables--nvl--t-natchain-prerouting-policy-accept-304-packets-38526-bytes-pkts-bytes-target-----prot-opt-in-----out-----source---------------destination----------chain-postrouting-policy-accept-7-packets-483-bytes-pkts-bytes-target-----prot-opt-in-----out-----source---------------destination-------------0-----0-masquerade--tcp-------------------192168122024----192168122024----masq-ports-1024-65535-----0-----0-masquerade--udp-------------------192168122024----192168122024----masq-ports-1024-65535-----0-----0-masquerade--all-------------------192168122024----192168122024-----chain-output-policy-accept-7-packets-483-bytes-pkts-bytes-target-----prot-opt-in-----out-----source---------------destination)
    * [kvm创建虚拟机](#kvm创建虚拟机)
* [netstat -ntlp|grep 5900tcp        0      0 0.0.0.0:5900                0.0.0.0:*                   LISTEN      2504/qemu-kvm](#netstat--ntlpgrep-5900tcp--------0------0-00005900----------------0000-------------------listen------2504qemu-kvm)
    * [虚拟机远程管理软件](#虚拟机远程管理软件)
    * [KVM虚拟机管理](#kvm虚拟机管理)
    * [libvirt虚拟机配置文件](#libvirt虚拟机配置文件)
* [lltotal 8-rw-------. 1 root root 3047 Oct 19  2016 Centos-6.6-x68_64.xmldrwx------. 3 root root 4096 Oct 17  2016 networks](#lltotal-8-rw--------1-root-root-3047-oct-19--2016-centos-66-x68_64xmldrwx-------3-root-root-4096-oct-17--2016-networks)
    * [监控kvm虚拟机](#监控kvm虚拟机)
    * [KVM修改NAT模式为桥接[案例]](#kvm修改nat模式为桥接[案例])
* [virsh edit Centos-6.6-x68_64  # 命令 52     <interface type='network'>     53       <mac address='52:54:00:2a:2d:60'/>     54       <source network='default'/>     55            56     </interface> 修改为：52     <interface type='bridge'>     53       <mac address='52:54:00:2a:2d:60'/>     54       <source bridge='br0'/>     55            56     </interface>](#virsh-edit-centos-66-x68_64---命令-52-----interface-typenetwork-----53-------mac-address5254002a2d60-----54-------source-networkdefault-----55------------56-----interface-修改为：52-----interface-typebridge-----53-------mac-address5254002a2d60-----54-------source-bridgebr0-----55------------56-----interface)
* [brctl showbridge name     bridge id               STP enabled     interfacesbr0             8000.000c29f824c9       no              eth0virbr0          8000.525400353d8e       yes             virbr0-nic](#brctl-showbridge-name-----bridge-id---------------stp-enabled-----interfacesbr0-------------8000000c29f824c9-------no--------------eth0virbr0----------8000525400353d8e-------yes-------------virbr0-nic)
* [virsh start CentOS-6.6-x86_64Domain CentOS-6.6-x86_64 started # brctl show                   bridge name     bridge id               STP enabled     interfacesbr0             8000.000c29f824c9       no              eth0                                                        vnet0virbr0          8000.525400353d8e       yes             virbr0-nic](#virsh-start-centos-66-x86_64domain-centos-66-x86_64-started--brctl-show-------------------bridge-name-----bridge-id---------------stp-enabled-----interfacesbr0-------------8000000c29f824c9-------no--------------eth0--------------------------------------------------------vnet0virbr0----------8000525400353d8e-------yes-------------virbr0-nic)
* [ifup eth0](#ifup-eth0)
* [ssh 192.168.2.108root@192.168.2.108's password: Last login: Sat Jan 30 12:40:28 2016](#ssh-1921682108root1921682108s-password-last-login-sat-jan-30-124028-2016)
    * [总结](#总结)


# [Qemu，KVM，Virsh傻傻的分不清](https://www.cnblogs.com/popsuper1982/p/8522535.html)

 本文转载自[Itweet](https://link.juejin.im/?target=http%3A%2F%2Fwww.itweet.cn)的博客

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本系列文章将整理到我的个人博客
> www.how2playlife.com

更多Java技术文章会更新在我的微信公众号【Java技术江湖】上，欢迎关注
该系列博文会介绍常见的后端技术，这对后端工程师来说是一种综合能力，我们会逐步了解搜索技术，云计算相关技术、大数据研发等常见的技术喜提，以便让你更完整地了解后端技术栈的全貌，为后续参与分布式应用的开发和学习做好准备。


如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系我，欢迎你参与本系列博文的创作和修订。

<!-- more -->



当你安装了一台Linux，想启动一个KVM虚拟机的时候，你会发现需要安装不同的软件，启动虚拟机的时候，有多种方法：

*   virsh start

*   kvm命令

*   qemu命令

*   qemu-kvm命令

*   qemu-system-x86_64命令

这些之间是什么关系呢？请先阅读上一篇《[白话虚拟化技术](https://blog.csdn.net/a724888/article/details/80996570)》

有了上一篇的基础，我们就能说清楚来龙去脉。

KVM（Kernel-based Virtual Machine的英文缩写）是内核内建的虚拟机。有点类似于 Xen ，但更追求更简便的运作，比如运行此虚拟机，仅需要加载相应的 kvm 模块即可后台待命。和 Xen 的完整模拟不同的是，KVM 需要芯片支持虚拟化技术（英特尔的 VT 扩展或者 AMD 的 AMD-V 扩展）。

首先看qemu，其中关键字emu，全称emulator，模拟器，所以单纯使用qemu是采用的完全虚拟化的模式。

Qemu向Guest OS模拟CPU，也模拟其他的硬件，GuestOS认为自己和硬件直接打交道，其实是同Qemu模拟出来的硬件打交道，Qemu将这些指令转译给真正的硬件。由于所有的指令都要从Qemu里面过一手，因而性能比较差

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150620008-108720261.jpg)

按照上一次的理论，完全虚拟化是非常慢的，所以要使用硬件辅助虚拟化技术Intel-VT，AMD-V，所以需要CPU硬件开启这个标志位，一般在BIOS里面设置。查看是否开启

对于Intel CPU 可用命令 grep "vmx" /proc/cpuinfo 判断

对于AMD CPU 可用命令 grep "svm" /proc/cpuinfo 判断

当确认开始了标志位之后，通过KVM，GuestOS的CPU指令不用经过Qemu转译，直接运行，大大提高了速度。

所以KVM在内核里面需要有一个模块，来设置当前CPU是Guest OS在用，还是Host OS在用。

查看内核模块中是否含有kvm, ubuntu默认加载这些模块

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150634298-628102674.png)

KVM内核模块通过/dev/kvm暴露接口，用户态程序可以通过ioctl来访问这个接口，例如书写下面的程序

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150654328-1662633336.png)

Qemu将KVM整合进来，通过ioctl调用/dev/kvm接口，将有关CPU指令的部分交由内核模块来做，就是qemu-kvm (qemu-system-XXX)

Qemu-kvm对kvm的整合从release_0_5_1开始有branch，在1.3.0正式merge到master

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150710177-1591777831.png)

qemu和kvm整合之后，CPU的性能问题解决了，另外Qemu还会模拟其他的硬件，如Network, Disk，同样全虚拟化的方式也会影响这些设备的性能。

于是qemu采取半虚拟化或者类虚拟化的方式，让Guest OS加载特殊的驱动来做这件事情。

例如网络需要加载virtio_net，存储需要加载virtio_blk，Guest需要安装这些半虚拟化驱动，GuestOS知道自己是虚拟机，所以数据直接发送给半虚拟化设备，经过特殊处理，例如排队，缓存，批量处理等性能优化方式，最终发送给真正的硬件，一定程度上提高了性能。

至此整个关系如下：

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150733042-369996016.jpg)

qemu-kvm会创建Guest OS，当需要执行CPU指令的时候，通过/dev/kvm调用kvm内核模块，通过硬件辅助虚拟化方式加速。如果需要进行网络和存储访问，则通过类虚拟化或者直通Pass through的方式，通过加载特殊的驱动，加速访问网络和存储资源。

然而直接用qemu或者qemu-kvm或者qemu-system-xxx的少，大多数还是通过virsh启动，virsh属于libvirt工具，libvirt是目前使用最为广泛的对KVM虚拟机进行管理的工具和API，可不止管理KVM。

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150801681-1586679180.jpg)

Libvirt分服务端和客户端，Libvirtd是一个daemon进程，是服务端，可以被本地的virsh调用，也可以被远程的virsh调用，virsh相当于客户端。

Libvirtd调用qemu-kvm操作虚拟机，有关CPU虚拟化的部分，qemu-kvm调用kvm的内核模块来实现

![](https://images2018.cnblogs.com/blog/635909/201803/635909-20180307150815111-1223973253.jpg)

这下子，整个相互关系才搞清楚了。

虽然使用virsh创建虚拟机相对简单，但是为了探究虚拟机的究竟如何使用，下一次，我们来解析一下如何裸使用qemu-kvm来创建一台虚拟机，并且能上网。

如果搭建使用过vmware桌面版或者virtualbox桌面版，创建一个能上网的虚拟机非常简单，但是其实背后做了很多事情，下一次我们裸用qemu-kvm，全部使用手工配置，看创建虚拟机都做了哪些事情。

本章节我们主要介绍通过VMware技术虚拟出相关的Linux软件环境，在Linux系统中，安装KVM虚拟化软件，实实在在的去实践一下KVM到底是一个什么样的技术？

##  Kvm虚拟化技术实践

### VMware虚拟机支持Kvm虚拟化技术？

在VMware创建的虚拟机中，默认不支持Kvm虚拟化技术，需要芯片级的扩展支持，幸好VMware提供完整的解决方案，可以通过修改虚拟化引擎。

VMware软件版本信息，`VMware® Workstation 11.0.0 build-2305329`

首先，你需要启动VMware软件，新建一个`CentOS 6.x`类型的虚拟机，正常安装完成，这个虚拟机默认的`虚拟化引擎`，`首选模式`为”自动”。

如果想让我们的VMware虚拟化出来的CentOS虚拟机支持KVM虚拟化，我们需要修改它支持的`虚拟化引擎`,打开新建的虚拟机，虚拟机状态必须处于`关闭`状态，通过双击`编辑虚拟机设置` > `硬件` ，选择`处理器`菜单，右边会出现`虚拟化引擎`区域，选择`首选模式`为 _Intel Tv-x/EPT或AMD-V/RVI_,接下来勾选`虚拟化Intel Tv-x/EPT或AMD-V/RVI(v)`，点击`确定`。

KVM需要虚拟机宿主（host）的处理器带有虚拟化支持（对于Intel处理器来说是VT-x，对于AMD处理器来说是AMD-V）。你可以通过以下命令来检查你的处理器是否支持虚拟化：

```
 grep --color -E '(vmx|svm)' /proc/cpuinfo

```

如果运行后没有显示，那么你的处理器不支持硬件虚拟化，你不能使用KVM。

*   注意: 如果是硬件服务器，您可能需要在BIOS中启用虚拟化支持，参考 [Private Cloud personal workstation](https://link.juejin.im/?target=http%3A%2F%2Fwww.itweet.cn%2Fblog%2F2016%2F06%2F14%2FPrivate%2520Cloud%2520personal%2520workstation)

### 安装Kvm虚拟化软件

安装kvm虚拟化软件，我们需要一个Linux操作系统环境，这里我们选择的Linux版本为`CentOS release 6.8 (Final)`，在这个VMware虚拟化出来的虚拟机中安装kvm虚拟化软件，具体步骤如下：

*   首选安装epel源

    ```
    sudo rpm -ivh http://mirrors.ustc.edu.cn/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm

    ```

*   安装kvm虚拟化软件

    ```
    sudo yum install qemu-kvm qeum-kvm-tools virt-manager libvirt

    ```

*   启动kvm虚拟化软件

    ```
    sudo /etc/init.d/libvirtd start

    ```

启动成功之后你可以通过`/etc/init.d/libvirtd status`查看启动状态，这个时候，kvm会自动生成一个本地网桥 `virbr0`，可以通过命令查看他的详细信息

```
# ifconfig virbr0virbr0    Link encap:Ethernet  HWaddr 52:54:00:D7:23:AD            inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1          RX packets:0 errors:0 dropped:0 overruns:0 frame:0          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0          collisions:0 txqueuelen:0           RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
```

KVM默认使用NAT网络模式。虚拟机获取一个私有 IP（例如 192.168.122.0/24 网段的），并通过本地主机的NAT访问外网。

```
# brctl showbridge name     bridge id               STP enabled     interfacesvirbr0          8000.525400d723ad       yes             virbr0-nic
```

创建一个本地网桥virbr0，包括两个端口：virbr0-nic 为网桥内部端口，vnet0 为虚拟机网关端口（192.168.122.1）。

虚拟机启动后，配置 192.168.122.1（vnet0）为网关。所有网络操作均由本地主机系统负责。

DNS/DHCP的实现，本地主机系统启动一个 dnsmasq 来负责管理。

```
ps aux|grep dnsmasq

```

`注意：` 启动libvirtd之后自动启动iptables，并且写上一些默认规则。

```
# iptables -nvL -t natChain PREROUTING (policy ACCEPT 304 packets, 38526 bytes) pkts bytes target     prot opt in     out     source               destination          Chain POSTROUTING (policy ACCEPT 7 packets, 483 bytes) pkts bytes target     prot opt in     out     source               destination             0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24    masq ports: 1024-65535     0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24    masq ports: 1024-65535     0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24     Chain OUTPUT (policy ACCEPT 7 packets, 483 bytes) pkts bytes target     prot opt in     out     source               destination
```

### kvm创建虚拟机

上传一个镜像文件：`CentOS-6.6-x86_64-bin-DVD1.iso`

通过`qemu`创建一个raw格式的文件(注：QEMU使用的镜像文件：qcow2与raw，它们都是QEMU(KVM)虚拟机使用的磁盘文件格式)，大小为5G。

```
qemu-img create -f raw /data/Centos-6.6-x68_64.raw 5G

```

查看创建的raw磁盘格式文件信息

```
qemu-img info /data/Centos-6.6-x68_64.raw  image: /data/Centos-6.6-x68_64.rawfile format: rawvirtual size: 5.0G (5368709120 bytes)disk size: 0
```

启动，kvm虚拟机，进行操作系统安装

```
virt-install  --virt-type kvm --name CentOS-6.6-x86_64 --ram 512 --cdrom /data/CentOS-6.6-x86_64-bin-DVD1.iso --disk path=/data/Centos-6.6-x68_64.raw --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole

```

启动之后，通过命令查看启动状态，默认会在操作系统开一个`5900`的端口，可以通过虚拟机远程管理软件`vnc`客户端连接，然后可视化的方式安装操作系统。

```
# netstat -ntlp|grep 5900tcp        0      0 0.0.0.0:5900                0.0.0.0:*                   LISTEN      2504/qemu-kvm
```

`注意`：kvm安装的虚拟机，不确定是那一台，在后台就是一个进程，每增加一台端口号+1，第一次创建的为5900！

### 虚拟机远程管理软件

我们可以使用虚拟机远程管理软件VNC进行操作系统的安装，我使用过的两款不错的虚拟机远程管理终端软件，一个是Windows上使用，一个在Mac上为了方便安装一个Google Chrome插件后即可开始使用，软件信息 `Tightvnc` 或者 `VNC[@Viewer](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2FViewer "@Viewer") for Google Chrome`

如果你和我一样使用的是`Google Chrome`提供的VNC插件，使用方式，在`Address`输入框中输入，宿主机IP:59000,`Picture Quality`选择框使用默认选项，点击`Connect`进入到安装操作系统的界面，你可以安装常规的方式进行安装，等待系统安装完成重启，然后就可以正常使用kvm虚拟化出来的操作系统了。

`Tightvnc`软件的使用，请参考官方手册。

*   Tightvnc下载地址：[www.tightvnc.com/download.ph…](https://link.juejin.im/?target=http%3A%2F%2Fwww.tightvnc.com%2Fdownload.php)
*   Tightvnc下载地址：[www.tightvnc.com/download/2.…](https://link.juejin.im/?target=http%3A%2F%2Fwww.tightvnc.com%2Fdownload%2F2.7.10%2Ftightvnc-2.7.10-setup-64bit.msi)
*   Tightvnc下载地址：[www.tightvnc.com/download/2.…](https://link.juejin.im/?target=http%3A%2F%2Fwww.tightvnc.com%2Fdownload%2F2.7.10%2Ftightvnc-2.7.10-setup-32bit.msi)

### KVM虚拟机管理

kvm虚拟机是通过virsh命令进行管理的，libvirt是Linux上的虚拟化库，是长期稳定的C语言API，支持KVM/QEMU、Xen、LXC等主流虚拟化方案。链接：[libvirt.org/](https://link.juejin.im/?target=http%3A%2F%2Flibvirt.org%2F)
virsh是Libvirt对应的shell命令。

查看所有虚拟机状态

```
virsh list --all

```

启动虚拟机

```
virsh start [NAME]

```

列表启动状态的虚拟机

```
virsh list

```

*   常用命令查看

    ```
    virsh --help|more less

    ```

### libvirt虚拟机配置文件

虚拟机libvirt配置文件在`/etc/libvirt/qemu`路径下，生产中我们需要去修改它的网络信息。

```
# lltotal 8-rw-------. 1 root root 3047 Oct 19  2016 Centos-6.6-x68_64.xmldrwx------. 3 root root 4096 Oct 17  2016 networks
```

`注意`：不能直接修改xml文件，需要通过提供的命令！

```
 virsh edit Centos-6.6-x68_64

```

kvm三种网络类型,桥接、NAT、仅主机模式，默认NAT模式,其他机器无法登陆，生产中一般选择桥接。

### 监控kvm虚拟机

*   安装软件监控虚拟机

```
yum install virt-top -y

```

*   查看虚拟机资源使用情况

```
virt-top virt-top 23:46:39 - x86_64 1/1CPU 3392MHz 3816MB1 domains, 1 active, 1 running, 0 sleeping, 0 paused, 0 inactive D:0 O:0 X:0CPU: 5.6%  Mem: 2024 MB (2024 MB by guests)    ID S RDRQ WRRQ RXBY TXBY %CPU %MEM    TIME   NAME                                                                                                     1 R    0    1   52    0  5.6 53.0   5:16.15 centos-6.8
```

### KVM修改NAT模式为桥接[案例]

在开始案例之前，需要知道的必要信息，宿主机IP是`192.168.2.200`，操作系统版本`Centos-6.6-x68_64`。

启动虚拟网卡

```
ifup eth0

```

这里网卡是NAT模式，可以上网，ping通其他机器，但是其他机器无法登陆！

宿主机查看网卡信息

```
brctl show ifconfig virbr0 ifconfig vnet0
```

_实现网桥，在kvm宿主机完成_

*   步骤1，创建一个网桥，新建网桥连接到eth0,删除eth0,让新的网桥拥有eth0的ip

```
brctl addbr br0  #创建一个网桥 brctl show       #显示网桥信息 brctl addif br0 eth0 && ip addr del dev eth0 192.168.2.200/24 && ifconfig br0 192.168.2.200/24 up brctl show      #查看结果ifconfig br0    #验证br0是否成功取代了eth0的IP
```

`注意`: 这里的IP地址为 _宿主机ip_

*   修改虚拟机桥接到br0网卡，在宿主机修改

```
virsh list --all ps aux |grep kvm virsh stop Centos-6.6-x68_64 virsh list --all
```

修改虚拟机桥接到宿主机，修改52行type为`bridge`，第54行bridge为`br0`

```
# virsh edit Centos-6.6-x68_64  # 命令 52     <interface type='network'>     53       <mac address='52:54:00:2a:2d:60'/>     54       <source network='default'/>     55            56     </interface> 修改为：52     <interface type='bridge'>     53       <mac address='52:54:00:2a:2d:60'/>     54       <source bridge='br0'/>     55            56     </interface>
```

启动虚拟机，看到启动前后，桥接变化，vnet0被桥接到了br0

启动前：

```
# brctl showbridge name     bridge id               STP enabled     interfacesbr0             8000.000c29f824c9       no              eth0virbr0          8000.525400353d8e       yes             virbr0-nic
```

启动后：

```
# virsh start CentOS-6.6-x86_64Domain CentOS-6.6-x86_64 started # brctl show                   bridge name     bridge id               STP enabled     interfacesbr0             8000.000c29f824c9       no              eth0                                                        vnet0virbr0          8000.525400353d8e       yes             virbr0-nic
```

Vnc登陆后，修改ip地址，看到dhcp可以使用，被桥接到现有的ip段，ip是自动获取,而且是和宿主机在同一个IP段.

```
# ifup eth0

```

从宿主机登陆此服务器，可以成功。

```
# ssh 192.168.2.108root@192.168.2.108's password: Last login: Sat Jan 30 12:40:28 2016
```

从同一网段其他服务器登陆此虚拟机,也可以成功,至此让kvm管理的服务器能够桥接上网就完成了，在生产环境中，桥接上网是非常必要的。

### 总结

通过kvm相关的命令来创建虚拟机，安装和调试是非常必要的，因为现有的很多私有云，公有云产品都使用到了kvm这样的技术，学习基本的kvm使用对维护`openstack`集群有非常要的作用，其次所有的`openstack image`制作也得通过kvm这样的底层技术来完成，最后上传到`openstack`的镜像管理模块，才能开始通过`openstack image`生成云主机。

到此，各位应该能够体会到，其实kvm是一个非常底层和核心的虚拟化技术，而openstack就是对`kvm`这样的技术进行了一个上层封装，可以非常方便，可视化的操作和维护`kvm`虚拟机，这就是现在`牛`上天的`云计算`技术最底层技术栈，具体怎么实现请看下图。

![Libvirt_support](https://user-gold-cdn.xitu.io/2017/6/19/69d6aeff29b1a8ff1d9a62579ad91afb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如上图，没有`openstack`我们依然可以通过，`libvirt`来对虚拟机进行操作，只不过比较繁琐和难以维护。通过openstack就可以非常方便的进行底层虚拟化技术的管理、维护、使用。


