原文地址： https://blogs.oracle.com/ronen/entry/diving_into_openstack_network_architecture
### 前言   
openstack网络功能强大同时也相对更复杂。本系列文章通过Oracle OpenStack Tech
Preview介绍openstack的配置，通过各种场景和例子说明openstack各种不同的网络组件。
本文的目的在于提供openstack网络架构的全景图并展示各个模块是如何一起协作的。这对openstack的初学者以及希望理解openstack网络原理的人会非常有帮助。
首先，我们先讲解下一些基础并举例说明。

根据最新的icehouse版用户调查，基于open vswitch插件的Neutron在生产环境和POC环境都被广泛使用，所以在这个系列的文章中我们主要分析这种openstack网络的配置。当然，我们知道openstack网络支持很多种配置，尽管neutron+open vswitch是最常用的配置，但是我们从未说它是最好或者最高效的一种方式。Neutron+open vswitch仅仅是一个例子，对任何希望理解openstack网络的人是一个很好的切入点。即使你打算使用其他类型的网络配置比如使用不同的neutron插件或者根本不使用neutron，这篇文章对你理解openstack网络仍是一个很好的开始。

我们在例子中使用的配置是Oracle OpenStack Tech Preview所提供的一种配置。安装它非常简单，并且它是一个很好的参考。在这种配置中，我们在所有服务器上使用eth2作为虚拟机的网络，所有虚拟机流量使用这个网卡。Oracle OpenStack Tech Preview使用VLAN进行L2隔离，进而提供租户和网络隔离，下图展示了我们如何进行配置和部署：
![setup2](https://blogs.oracle.com/ronen/resource/setup2.jpg)

第一篇文章会略长，我们将聚焦于openstack网络的一些基本概念。我们将讨论open vswitch、network namespaces、linux bridge、veth pairs等几个组件。注意这里不打算全面介绍这些组件，只是为了理解openstack网络架构。可以通过网络上的其他资源进一步了解这些组件。

### Open vSwitch (OVS)   
在Oracle OpenStack Tech Preview中用于连接虚拟机和物理网口（如上例中的eth2)，就像上边部署图所示。OVS包含bridages和ports，OVS bridges不同于与linux bridge（使用brctl命令创建）。让我们先看下OVS的结构，使用如下命令：
<pre><code>
# ovs-vsctl show
7ec51567-ab42-49e8-906d-b854309c9edf
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
        Port "int-br-eth2"
            Interface "int-br-eth2"
    Bridge "br-eth2"
        Port "br-eth2"
            Interface "br-eth2"
                type: internal
        Port "eth2"
            Interface "eth2"
        Port "phy-br-eth2"
            Interface "phy-br-eth2"
ovs_version: "1.11.0"
</code></pre>
我们看到标准的部署在compute node上的OVS，拥有两个网桥，每个有若干相关联的port。上边的例子是在一个没有任何虚拟机的计算节点上。我们可以看到eth2连接到个叫br-eth2的网桥上，我们还看到两个叫“int-br-eth2"和”phy-br-eth2“的port，事实上是一个veth pair，作为虚拟网线连接两个bridages。我们会在后边讨论veth paris。

当我们创建一个虚拟机，br-int网桥上会创建一个port，这个port最终连接到虚拟机（我们会在后边讨论这个连接）。这里是启动一个虚拟机后的OVS结构：
<pre><code>
# ovs-vsctl show
efd98c87-dc62-422d-8f73-a68c2a14e73d
    Bridge br-int
        Port "int-br-eth2"
            Interface "int-br-eth2"
        Port br-int
            Interface br-int
                type: internal
        Port "qvocb64ea96-9f"
            tag: 1
            Interface "qvocb64ea96-9f"
    Bridge "br-eth2"
        Port "phy-br-eth2"
            Interface "phy-br-eth2"
        Port "br-eth2"
            Interface "br-eth2"
                type: internal
        Port "eth2"
            Interface "eth2"
ovs_version: "1.11.0"
</code></pre>
”br-int“网桥现在有了一个新的port"qvocb64ea96-9f" 连接VM，并且被标记为vlan1。虚拟机的每个网卡都需要对应在"br-int”网桥上创建一个port。

OVS中另一个有用的命令是dump-flows，以下为例子：  
<pre><code>
# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=735.544s, table=0, n_packets=70, n_bytes=9976,idle_age=17, priority=3,in_port=1,dl_vlan=1000 actions=mod_vlan_vid:1,NORMAL
cookie=0x0, duration=76679.786s, table=0, n_packets=0, n_bytes=0,idle_age=65534, hard_age=65534, priority=2,in_port=1 actions=drop
cookie=0x0, duration=76681.36s, table=0, n_packets=68, n_bytes=7950,idle_age=17, hard_age=65534, priority=1 actions=NORMAL
</code></pre>
如上所述，VM相连的port使用了Vlan tag 1。然后虚拟机网络（eth2)上的port使用tag1000。OVS会修改VM和物理网口间所有package的vlan。在openstack中，OVS
 agent 控制open vswitch中的flows，用户不需要进行操作。如果你想了解更多的如何控制open vswitch中的流，可以参考http://openvswitch.org中对ovs-ofctl的描述。

### Network Namespaces (netns)  
网络namespace是linux上一个很cool的特性，它的用途很多。在openstack网络中被广泛使用。网络namespace是拥有独立的网络配置隔离容器，并且该网络不能被其他名字空间看到。网络名字空间可以被用于封装特殊的网络功能或者在对网络服务隔离的同时完成一个复杂的网络设置。在Oracle OpenStack Tech Preview中我们使用最新的R3企业版内核，该内核提供给了对netns的完整支持。

通过如下例子我们展示如何使用netns命令控制网络namespaces。
定义一个新的namespace:
<pre><code>
# ip netns add my-ns
# ip netns list
my-ns
</code></pre>
我们说过namespace是一个隔离的容器，我们可以在namspace中进行各种操作，比如ifconfig命令。
<pre><code>
# ip netns exec my-ns ifconfig -a
lo        Link encap:Local Loopback
          LOOPBACK  MTU:16436 Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
</code></pre>
我们可以在namespace中运行任何命令，比如对debug非常有用的tcddump命令，我们使用ping、ssh、iptables命令。
连接namespace和外部：
连接到namespace和namespace直接连接的方式有很多，我们主要聚集在openstack中使用的方法。openstack使用了OVS和网络namespace的组合。OVS定义接口，然后我们将这些接口加入namespace中。
<pre><code>
# ip netns exec my-ns ifconfig -a
lo        Link encap:Local Loopback
          LOOPBACK  MTU:65536 Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

my-port   Link encap:Ethernet HWaddr 22:04:45:E2:85:21
          BROADCAST  MTU:1500 Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
</code></pre>
现在我们可以增加更多的ports到OVS bridge，并且连接到其他namespace或者其他设备比如物理网卡。Neutron使用网络namespace来实现网络服务，如DHCP、routing、gateway、firewall、load balance等。下一篇文章我们会讨论更多细节 。

### Linux bridge and veth pairs   
Linux bridge用于连接OVS port和虚拟机。ports负责连通OVS bridge和linux bridge或者两者与虚拟机。linux bridage主要用于安全组增强。安全组通过iptables实现，iptables只能用于linux bridage而非OVS bridage。

Veth对在openstack网络中大量使用，也是debug网络问题的很好工具。Veth对是一个简单的虚拟网线，所以一般成对出现。通常Veth对的一端连接到bridge，另一端连接到另一个bridge或者留下在作为一个网口使用。

这个例子中，我们将创建一些veth对，把他们连接到bridge上并测试联通性。这个例子用于通常的Linux服务器而非openstack节点：
创建一个veth对，注意我们定义了两端的名字：
<pre><code>
# ip link add veth0 type veth peer name veth1

# ifconfig -a

.

.

veth0     Link encap:Ethernet HWaddr 5E:2C:E6:03:D0:17

          BROADCAST MULTICAST  MTU:1500 Metric:1

          RX packets:0 errors:0 dropped:0 overruns:0 frame:0

          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0

collisions:0 txqueuelen:1000

          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

veth1     Link encap:Ethernet HWaddr E6:B6:E2:6D:42:B8

          BROADCAST MULTICAST  MTU:1500 Metric:1

          RX packets:0 errors:0 dropped:0 overruns:0 frame:0

          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0

collisions:0 txqueuelen:1000

          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

.

.
</code></pre>

为了让例子更有意义，我们将创建如下配置：
<pre><code>
veth0 => veth1 =>br-eth3 => eth3 ======> eth2 on another Linux server
</code></pre>
br-eht3: 一个基本的Linux bridge，连接veth1和eth3
eth3:    一个没有设定IP的物理网口，该网口连接着斯有网络
eth2:    远端Linux服务器上的一个物理网口，连接着私有网络并且被配置了IP（50.50.50.1）
一旦我们创建了这个配置，我们将通过veth0 ping 50.50.50.1这个远端IP，从而测试网络联通性：
<pre><code>


# brctl addbr br-eth3

# brctl addif br-eth3 eth3

# brctl addif br-eth3 veth1

# brctl show

bridge name     bridge id               STP enabled     interfaces

br-eth3         8000.00505682e7f6       no              eth3

                                                        veth1

# ifconfig veth0 50.50.50.50

# ping -I veth0 50.50.50.51

PING 50.50.50.51 (50.50.50.51) from 50.50.50.50 veth0: 56(84) bytes of data.

64 bytes from 50.50.50.51: icmp_seq=1 ttl=64 time=0.454 ms

64 bytes from 50.50.50.51: icmp_seq=2 ttl=64 time=0.298 ms
</code></pre>

如果命名不像例子中这么显而易见，导致我们无法支持veth设备的两端，我们可以使用ethtool命令查询。ethtool命令返回index号，通过ip link命令查看对应的设备： 
<pre><code>
# ethtool -S veth1

NIC statistics:

peer_ifindex: 12

# ip link

.

.

12: veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
</code></pre>

### 总结  
文章中，我们快速了解了OVS/网络namespaces/Linux bridges/veth对。这些组件在openstack网络架构中大量使用，理解这些组件有助于我们理解不同的网络场景。下篇文章中，我们会了解虚拟机之间/虚拟机与外部网络之间如何进行通信。
