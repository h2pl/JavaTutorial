OpenStack 是开源云计算平台，支持多种虚拟化环境，并且其服务组件都提供了 API接口 便于二次开发。

OpenStack通过各种补充服务提供基础设施即服务 Infrastructure-as-a-Service (IaaS)`的解决方案。每个服务都提供便于集成的应用程序接口`Application Programming Interface (API)。

### openstack 逻辑架构图

![](http://mmbiz.qpic.cn/mmbiz_png/MxvbhgSaC6GWEd8OJ3JRIWQfkhGo23icH3stE59wmxYcibF1kO8Eq288YjtayCVEIDpeDs5jN7lACVSsk3DwPibFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

OpenStack 本身是一个分布式系统，不但各个服务可以分布部署，服务中的组件也可以分布部署。 这种分布式特性让 OpenStack 具备极大的灵活性、伸缩性和高可用性。 当然从另一个角度讲，这也使得 OpenStack 比一般系统复杂，学习难度也更大。

后面章节我们会深入学习 Keystone、Glance、Nova、Neutron 和 Cinder 这几个 OpenStack 最重要最核心的服务。

openstack的核心和扩展的主要项目如下：

*   OpenStack Compute (code-name Nova) 计算服务

*   OpenStack Networking (code-name Neutron) 网络服务

*   OpenStack Object Storage (code-name Swift) 对象存储服务

*   OpenStack Block Storage (code-name Cinder) 块设备存储服务

*   OpenStack Identity (code-name Keystone) 认证服务

*   OpenStack Image Service (code-name Glance) 镜像文件服务

*   OpenStack Dashboard (code-name Horizon) 仪表盘服务

*   OpenStack Telemetry (code-name Ceilometer) 告警服务

*   OpenStack Orchestration (code-name Heat) 流程服务

*   OpenStack Database (code-name Trove) 数据库服务

OpenStack的各个服务之间通过统一的REST风格的API调用，实现系统的松耦合。上图是OpenStack各个服务之间API调用的概览，其中实线代表client 的API调用，虚线代表各个组件之间通过rpc调用进行通信。松耦合架构的好处是，各个组件的开发人员可以只关注各自的领域，对各自领域的修改不会影响到其他开发人员。不过从另一方面来讲，这种松耦合的架构也给整个系统的维护带来了一定的困难，运维人员要掌握更多的系统相关的知识去调试出了问题的组件。所以无论对于开发还是维护人员，搞清楚各个组件之间的相互调用关系是怎样的都是非常必要的。

对Linux经验丰富的OpenStack新用户，使用openstack是非常容易的，在后续`openstack系列`文章中会逐步展开介绍。

### OpenStack 项目和组件

OpenStack services

*   Dashboard     【Horizon】     提供了一个基于web的自服务门户，与OpenStack底层服务交互，诸如启动一个实例，分配IP地址以及配置访问控制。

*   Compute       【Nova】        在OpenStack环境中计算实例的生命周期管理。按需响应包括生成、调度、回收虚拟机等操作。

*   Networking    【Neutron】     确保为其它OpenStack服务提供网络连接即服务，比如OpenStack计算。为用户提供API定义网络和使用。基于插件的架构其支持众多的网络提供商和技术。

*   Object Storage    【Swift】   通过一个 RESTful,基于HTTP的应用程序接口存储和任意检索的非结构化数据对象。它拥有高容错机制，基于数据复制和可扩展架构。它的实现并像是一个文件服务器需要挂载目录。在此种方式下，它写入对象和文件到多个硬盘中，以确保数据是在集群内跨服务器的多份复制。

*   Block Storage       【Cinder】   为运行实例而提供的持久性块存储。它的可插拔驱动架构的功能有助于创建和管理块存储设备。

*   Identity service  【Keystone】    为其他OpenStack服务提供认证和授权服务，为所有的OpenStack服务提供一个端点目录。

*   Image service     【Glance】  存储和检索虚拟机磁盘镜像，OpenStack计算会在实例部署时使用此服务。

*   Telemetry服务      【Ceilometer】  为OpenStack云的计费、基准、扩展性以及统计等目的提供监测和计量。

*   Orchestration服务   【Heat服务】   Orchestration服务支持多样化的综合的云应用，通过调用OpenStack-native REST API和CloudFormation-compatible Query API，支持`HOT <Heat Orchestration Template (HOT)>`格式模板或者AWS CloudFormation格式模板

通过对这些组件的介绍，可以帮助我们在后续的内容中，了解各个组件的作用，便于排查问题，而在你对基础安装，配置，操作和故障诊断熟悉之后，你应该考虑按照生产架构来进行部署。

### 生产部署架构

建议使用自动化部署工具，例如Ansible, Chef, Puppet, or Salt来自动化部署，管理生产环境。

![](http://mmbiz.qpic.cn/mmbiz_png/MxvbhgSaC6GWEd8OJ3JRIWQfkhGo23icH6JtvNllODFno0K644yCIHEpfb8icyF7B3u6gXvgsT9AgzYgV26l9P1g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

这个示例架构需要至少2个（主机）节点来启动基础服务`virtual machine <virtual machine (VM)>`或者实例。像块存储服务，对象存储服务这一类服务还需要额外的节点。

*   网络代理驻留在控制节点上而不是在一个或者多个专用的网络节点上。

*   私有网络的覆盖流量通过管理网络而不是专用网络

#### 控制器

控制节点上运行身份认证服务，镜像服务，计算服务的管理部分，网络服务的管理部分，多种网络代理以及仪表板。也需要包含一些支持服务，例如：SQL数据库，term:消息队列, and NTP。

可选的，可以在计算节点上运行部分块存储，对象存储，Orchestration 和 Telemetry 服务。

计算节点上需要至少两块网卡。

#### 计算

计算节点上运行计算服务中管理实例的管理程序部分。默认情况下，计算服务使用 KVM。

你可以部署超过一个计算节点。每个结算节点至少需要两块网卡。

#### 块设备存储

可选的块存储节点上包含了磁盘，块存储服务和共享文件系统会向实例提供这些磁盘。

为了简单起见，计算节点和本节点之间的服务流量使用管理网络。生产环境中应该部署一个单独的存储网络以增强性能和安全。

你可以部署超过一个块存储节点。每个块存储节点要求至少一块网卡。

#### 对象存储

可选的对象存储节点包含了磁盘。对象存储服务用这些磁盘来存储账号，容器和对象。

为了简单起见，计算节点和本节点之间的服务流量使用管理网络。生产环境中应该部署一个单独的存储网络以增强性能和安全。

这个服务要求两个节点。每个节点要求最少一块网卡。你可以部署超过两个对象存储节点。

#### 网络

openstack网络是非常复杂的，并且也支持多种模式其中支持GRE，VLAN,VXLAN等，在openstack中网络是通过一个组件`Neutron`提供服务，Neutron 管理的网络资源包括如下。

*   network 是一个隔离的二层广播域。Neutron 支持多种类型的 network，包括 local, flat, VLAN, VxLAN 和 GRE。

*   local 网络与其他网络和节点隔离。local 网络中的 instance 只能与位于同一节点上同一网络的 instance 通信，local 网络主要用于单机测试。

*   flat 网络是无 vlan tagging 的网络。flat 网络中的 instance 能与位于同一网络的 instance 通信，并且可以跨多个节点。

*   vlan 网络是具有 802.1q tagging 的网络。vlan 是一个二层的广播域，同一 vlan 中的 instance 可以通信，不同 vlan 只能通过 router 通信。vlan 网络可以跨节点，是应用最广泛的网络类型。

*   vxlan 是基于隧道技术的 overlay 网络。vxlan 网络通过唯一的 segmentation ID（也叫 VNI）与其他 vxlan 网络区分。vxlan 中数据包会通过 VNI 封装成 UPD 包进行传输。因为二层的包通过封装在三层传输，能够克服 vlan 和物理网络基础设施的限制。

*   gre 是与 vxlan 类似的一种 overlay 网络。主要区别在于使用 IP 包而非 UDP 进行封装。 不同 network 之间在二层上是隔离的。以 vlan 网络为例，network A 和 network B 会分配不同的 VLAN ID，这样就保证了 network A 中的广播包不会跑到 network B 中。当然，这里的隔离是指二层上的隔离，借助路由器不同 network 是可能在三层上通信的。network 必须属于某个 Project（ Tenant 租户），Project 中可以创建多个 network。 network 与 Project 之间是 1对多关系。

*   subnet 是一个 IPv4 或者 IPv6 地址段。instance 的 IP 从 subnet 中分配。每个 subnet 需要定义 IP 地址的范围和掩码。

*   port 可以看做虚拟交换机上的一个端口。port 上定义了 MAC 地址和 IP 地址，当 instance 的虚拟网卡 VIF（Virtual Interface） 绑定到 port 时，port 会将 MAC 和 IP 分配给 VIF。port 与 subnet 是 1对多 关系。一个 port 必须属于某个 subnet；一个 subnet 可以有多个 port。

![](http://mmbiz.qpic.cn/mmbiz_png/MxvbhgSaC6GWEd8OJ3JRIWQfkhGo23icHcCr8k5VVbooGYOMC3QYqlmOW8gwneIOEicS0txC4HtEicJLIN6bEyVcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

如上图所示，为VLAN模式下，网络节点的通信方式。

在我们后续实施安装的时候，选择使用VXLAN网络模式，下面我们来重点介绍一下VXLAN模式。

![](http://mmbiz.qpic.cn/mmbiz_png/MxvbhgSaC6GWEd8OJ3JRIWQfkhGo23icHOLoLQuIYiar5Jm1YPBZQpxGJibVshiaJ6ZicLmwsT3dc6K53ibfHNwBsw5Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

VXLAN网络模式，可以隔离广播风暴，不需要交换机配置chunk口，解决了vlan id个数限制，解决了gre点对点隧道个数过多问题，实现了大2层网络，可以让vm在机房之间无缝迁移，便于跨机房部署。缺点是，vxlan增加了ip头部大小，需要降低vm的mtu值，传输效率上会略有下降。

### 涉及的 Linux 网络技术

Neutron 的设计目标是实现“网络即服务”，为了达到这一目标，在设计上遵循了基于“软件定义网络”实现网络虚拟化的原则，在实现上充分利用了 Linux 系统上的各种网络相关的技术。理解了 Linux 系统上的这些概念将有利于快速理解 Neutron 的原理和实现。

*   bridge：网桥，Linux中用于表示一个能连接不同网络设备的虚拟设备，linux中传统实现的网桥类似一个hub设备，而ovs管理的网桥一般类似交换机。

*   br-int：bridge-integration，综合网桥，常用于表示实现主要内部网络功能的网桥。

*   br-ex：bridge-external，外部网桥，通常表示负责跟外部网络通信的网桥。

*   GRE：General Routing Encapsulation，一种通过封装来实现隧道的方式。在openstack中一般是基于L3的gre，即original pkt/GRE/IP/Ethernet

*   VETH：虚拟ethernet接口，通常以pair的方式出现，一端发出的网包，会被另一端接收，可以形成两个网桥之间的通道。

*   qvb：neutron veth, Linux Bridge-side

*   qvo：neutron veth, OVS-side

*   TAP设备：模拟一个二层的网络设备，可以接受和发送二层网包。

*   TUN设备：模拟一个三层的网络设备，可以接受和发送三层网包。

*   iptables：Linux 上常见的实现安全策略的防火墙软件。

*   Vlan：虚拟 Lan，同一个物理 Lan 下用标签实现隔离，可用标号为1-4094。

*   VXLAN：一套利用 UDP 协议作为底层传输协议的 Overlay 实现。一般认为作为 VLan 技术的延伸或替代者。

*   namespace：用来实现隔离的一套机制，不同 namespace 中的资源之间彼此不可见。

### 总结

openstack是一个非法复杂的分布式软件，涉及到很多底层技术，我自己对一些网络的理解也是非常有限，主要还是应用层面的知识，所以本章内容写的比较浅显一些，有问题请留言？在下一章节我们会进入生产环境如何实施规划openstack集群，至于openstack底层的技术，我也没有很深入研究，如果有任何不恰当的地方可以进行留言，非常感谢！