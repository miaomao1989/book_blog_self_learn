# GRE与Vxlan网络详解

## 1. `GRE`

### 1.1 概念

`GRE`全称是`Generic Routing Encapsulation`，是一种协议封装的格式，具体内容请见：https://tools.ietf.org/html/rfc2784

协议封装指的是用一种格式的协议分装另一种格式的协议。我们熟悉的`TCP/IP`协议可以看成是一种封装： `TCP`传输层协议被网络层的`IP`协议封装，通过`IP`协议来进行点到点的传输。还有比如很有用的`IP SAN`，就是通过`IP`协议封装`scsi`协议，使得我们可以直接通过`IP`网络来进行磁盘数据的传输。对于这两个例子来说，前一种封装的目的是通过分层来严格区分协议的设计，使得具体的协议设计/实现的时候可以更加地清晰；而后者则是为了使用现有的设施，方便厂商推广自己的产品，同时通过两种协议的结合产生更多的功能。对于`GRE`来说，应是偏向后者的一种封装。

`GRE`的目的是设计一种通用的封装格式，所以如果将它与一些为特定目的进行设计的封装协议比较，那么`GRE`是没有太多优势的。

 A GRE encapsulated packet has the form:

|:---------------------------------:|
|                                   |
|          Delivery Header          |
|                                   |
| ：-----------------------------： |
|                                   |
|            GRE Header             |
|                                   |
| :------------------------------:  |
|                                   |
|          Payload packet           |
|                                   |
| :------------------------------:  |

在`GRE`中，需要被传输和封装的报文称之为`payload packet`，而用于封装和传输的协议则成为`delivery protocol`。 `GRE`在封装的时候，除了`payload`和`delivery`协议之外，会生成一个`GRE header`。 `GRE header + payload` 一起被`deliver`协议封装用于传输。 `GRE header`会包含`payload`的一些信息，包括`checksum`，`version`，`payload`的协议类型等等。可以看到,通过这个`GRE header`的协议类型字段，我们可以做很多事情。既然底层的`delivery`协议是用于传输的，那么`A`和`B`通信的时候`delivery`协议可以看成是个邮局送信的火车，虽然很重要，但是对于业务来说没有其运送的信重要。当脱开这一层`delivery`层后，我们怎么知道信的格式呢？通过`GRE header`中的协议类型我们就能知道协议的类型了，既然知道了协议类型，那么就有能力解析了。

由于`GRE`是一种通用的格式，我们可以使用`GRE`进行很多不同种类的封装。比如我们可以使用`PPTP`协议来进行`VPN`，可以使用`IPv4`来包裹`IPv6`，比较常见的`delivery`协议一般是`IP`协议。

不过`GRE`在设计的时候有一个问题，那就是没有考虑加密。因此现在常见的需要加密的封装一般是用`IPsec`协议。

比如说：`A`主机在公司，`B`主机在家，`A`的网络地址为`192.168.1.1`， `B`的网络地址为`192.168.2.1`，`A`如果要和`B`通信，则需要通过互联网，所以在`A`连接的路由器`RA`上，会配置一个`tunnel`口，`tunnel`口的信息是(1.1.1.1->2.2.2.2)，在`B`连接的路由器`RB`上也会配置一个`tunnel`口，`tunnel`口的信息是（2.2.2.2 -> 1.1.1.1）。同时在设置好路由的情况下，`A`发送一个报文给`B`，报文会首先到`RA`，`RA`发现报文需要走互联网，于是通过`tunnel`口封装，封装的`delivery`协议的目的地址写的是配置的`RB`地址`2.2.2.2`。报文到了`RB`后`delivery`协议被脱去，然后`RB`根据路由信息转发给`B`。`（http://assafmuller.com/2013/10/10/gre-tunnels/）`

![example](https://assafmuller.files.wordpress.com/2013/10/gre1.png?w=696)

当`A（192.168.1.1） ping B（192.168.2.1）`时，报文是如下形式的：

从`A`到`RA`:

![apingb](https://assafmuller.files.wordpress.com/2013/10/gre-encapsulation-before.png?w=696)

从`RA`到`RB`:

![RA2RB](https://assafmuller.files.wordpress.com/2013/10/gre-encapsulation-after.png?w=696)

### 1.2 `Neutron`中的`GRE`
`　（http://assafmuller.com/2013/10/14/gre-tunnels-in-openstack-neutron/）`

- br-tun网桥信息

```
[root@NextGen1 ~]# ovs-vsctl show
911ff1ca-590a-4efd-a066-568fbac8c6fb
[... Bridge br-int omitted ...]
    Bridge br-tun
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
        Port "gre-2"
            Interface "gre-2"
                type: gre
                options: {in_key=flow, local_ip="192.168.1.100", out_key=flow, remote_ip="192.168.1.101"}
        Port "gre-1"
            Interface "gre-1"
                type: gre
                options: {in_key=flow, local_ip="192.168.1.100", out_key=flow, remote_ip="192.168.1.102"}
```

- VM1(10.0.0.1) pings VM2(10.0.0.2) in different nodes :
Before VM1 can create an ICMP echo request message, VM1 must send out an ARP request for VM2’s MAC address. A quick reminder about ARP encapsulation – It is encapsulated directly in an Ethernet frame – No IP involved (There exists a base assumption that states that ARP requests never leave a broadcast domain therefor IP packets are not needed). The Ethernet frame leaves VM1’s tap device into the host’s br-int. br-int, acting as a normal switch, sees that the destination MAC address in the Ethernet frame is FF:FF:FF:FF:FF:FF – The broadcast address. Because of that it floods it out all ports, including the patch cable linked to br-tun. br-tun receives the frame from the patch cable port and sees that the destination MAC address is the broadcast address. Because of that it will send the message out all GRE tunnels (Essentially flooding the message). But before that, it will encapsulate the message in a GRE header and an IP packet. In fact, two new packets are created: One from 192.168.1.100 to 192.168.1.101, and the other from 192.168.1.100 to 192.168.1.102. The encapsulation over the GRE tunnels looks like this:

![](https://assafmuller.files.wordpress.com/2013/10/gre-encapsulation-arp1.png?w=696)

　　To summarize, we can conclude that the flow logic on br-tun implements a learning switch but with a GRE twist. If the message is to a multicast, broadcast, or unknown unicast address it is forwarded out all GRE tunnels. Otherwise if it learned the destination MAC address via earlier messages (By observing the source MAC address, tunnel ID and incoming GRE port) then it forwards it to the correct GRE tunnel.

- br-tun 流表
　　To summarize, we can conclude that the flow logic on br-tun implements a learning switch but with a GRE twist. If the message is to a multicast, broadcast, or unknown unicast address it is forwarded out all GRE tunnels. Otherwise if it learned the destination MAC address via earlier messages (By observing the source MAC address, tunnel ID and incoming GRE port) then it forwards it to the correct GRE tunnel.

```
[root@NextGen1 ~]# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=182369.287s, table=0, n_packets=5996, n_bytes=1481720, idle_age=52, hard_age=65534, priority=1,in_port=3 actions=resubmit(,2)
 cookie=0x0, duration=182374.574s, table=0, n_packets=14172, n_bytes=3908726, idle_age=5, hard_age=65534, priority=1,in_port=1 actions=resubmit(,1)
 cookie=0x0, duration=182370.094s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,in_port=2 actions=resubmit(,2)
 cookie=0x0, duration=182374.078s, table=0, n_packets=3, n_bytes=230, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x0, duration=182373.435s, table=1, n_packets=3917, n_bytes=797884, idle_age=52, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x0, duration=182372.888s, table=1, n_packets=10255, n_bytes=3110842, idle_age=5, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,21)
 cookie=0x0, duration=182103.664s, table=2, n_packets=5982, n_bytes=1479916, idle_age=52, hard_age=65534, priority=1,tun_id=0x1388 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x0, duration=182372.476s, table=2, n_packets=14, n_bytes=1804, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x0, duration=182372.099s, table=3, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x0, duration=182371.777s, table=10, n_packets=5982, n_bytes=1479916, idle_age=52, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
 cookie=0x0, duration=116255.067s, table=20, n_packets=3917, n_bytes=797884, hard_timeout=300, idle_age=52, hard_age=52, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:1f:19:55 actions=load:0->NXM_OF_VLAN_TCI[],load:0x1388->NXM_NX_TUN_ID[],output:3
 cookie=0x0, duration=182371.623s, table=20, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=resubmit(,21)
 cookie=0x0, duration=182103.777s, table=21, n_packets=10235, n_bytes=3109310, idle_age=5, hard_age=65534, priority=1,dl_vlan=1 actions=strip_vlan,set_tunnel:0x1388,output:3,output:2
 cookie=0x0, duration=182371.507s, table=21, n_packets=20, n_bytes=1532, idle_age=65534, hard_age=65534, priority=0 actions=drop
 ```

- `table`表流程

![table](https://assafmuller.files.wordpress.com/2014/01/flow-table-flow-chart.png)

- 在`openstack`中主要是通过`ovs`，`ovs`支持`GRE`。通过`GRE`，`VM`之间的`ARP`、`IP`报文都能在`GRE`的封装下通过`IP`网络进行传递。不同的网络通过`GRE header`中的`tunnel id`号区别。由于`ovs`支持`openflow`协议，为了效率和性能，`mac`和`GRE`的路由关系会存放在`ovs`的流表中。从链接中的文章可以看出，`GRE`有一个缺点，那就是每新增一个计算节点，都需要其和所有其他计算节点以及`network`控制器建立`GRE`链接。在计算节点很多的时候会有性能问题。

## 2. VXLAN
### 2.1 概念

相比于`GRE`的通用性，`VXLAN`主要用于封装/转发二层报文。`VXLAN`全称`Virutal eXtensible Local Area Network`，简单地说就是扩充了的VLAN，其使得多个通过三层连接的网络可以表现的和直接一台一台物理交换机连接配置而成的网络一样处在一个`LAN`中。其将二层报文加上个`vxlan header`，封装在一个`UDP`包中进行传输。`vxlan header`会包括一个24位的`ID`（称为`VNI`），含义类似于`VLAN id`或者上面提到的`GRE tunnel id`。在上面`GRE`的例子中，是通过路由器来进行`GRE`协议的封装和解封，在`VXLAN`中这类封装和解封的组件有一个专有的名字叫做`VTEP`。相比`VLAN`来说，好处在于其突破了`VLAN`只有4094个子网的限制。同时假设在`UDP`协议上后其扩展性提高了不少（因为`UDP`是高层协议，屏蔽了底层的差异，换句话说屏蔽了二层的差异。）

表面上看，`VXLAN`和`GRE`区别不大，只是对`delivery`协议做了限定，使用`UDP`。但是实际上在协议的交互动作上面还是有区别的。和上面的例子一样，例如主机`A`和主机`B`想通信，那么`A`的报文会被`VTEP`封装，然后发往连接`B`的`VTEP`。在上面的例子中类似于`VTEP`的角色是由路由器来充当的，而且路由器的两端的地址是配置好的，所以`RA`知道`RB`的地址，直接将报文发给`RB`即可。但是在`VXLAN`中，`A`的`VTEP`并不知道`B`的`VTEP`在哪里，所以需要一个发现过程。那么如何发现？`VXLAN`要求每个`VNI`都关联一个组播地址。所以对于一次`ARP`请求，`A`的`VTEP`会发送一个组播`IGMP`报文给所有同在这个网络组中的其他`VTEP`。所有的订阅了这个组播地址的`VTEP`会收到这个报文，学习发送端`A`的`MAC`地址和`VTEP`地址用于以后的使用，同时`VTEP`会将报文解析后比较`VNI`，发送给同`VNI`的主机。当某个主机`B`的`IP`和`ARP`中的一样时，其会发送`ARP`应答报文，应答报文通过`B`的`VTEP`按照类似的流程发送给`A`，但是由于`B`的`VTEP`已经学习到了`A`的`MAC`地址，因此`B`的`VTEP`就可以发送给`A`的`VTEP`，而不需要再走一边`IGMP`的组播过程了。

从这个例子可以看出，`VXLAN`屏蔽了`UDP`的存在，上层基本上感知不到这层的封装。同时`VXLAN`避免了`GRE`的点对点必须又连接的缺点。由于需要`IGMP`，对于物理交换机和路由器需要做一些配置，这点在`GRE`是不需要的。

`VXLAN`报文格式：
（http://www.borgcube.com/blogs/2011/11/vxlan-primer-part-1/）

（http://www.borgcube.com/blogs/2012/03/vxlan-primer-part-2-lets-get-physical/）

![vxlan-frame](http://i2.wp.com/www.borgcube.com/blog/wp-content/uploads/2016/03/VXLAN-Headers.png)

报文走向：

![](http://i2.wp.com/www.borgcube.com/blog/wp-content/uploads/2016/03/VXLAN-VM-to-VM-communication.png)

### 2.2  `Neutron`中的`VXLAN`
(http://www.opencloudblog.com/?p=300)
vxlan的br-tun流表与上面GRE类似。

### 2.3 需要`VXLAN`的原因
- `vlan`的数量限制：4094个`vlan`远不能满足大规模云计算数据中心的需求
- 物理网络基础设施的限制： 基于`IP`子网的区域划分限制了需要二层网络连通性的应用负载的部署。即虚拟机迁移后跨大二层网络的需求。
- `TOR`交换机`MAC`表耗尽： 虚拟化以及东西向流量导致更多的`MAC`表项。使用隧道技术后，记录的是`Delivery Header`里的`MAC` （即`VTEP`的`MAC`）。

### `vxlan`与`GRE`的主要区别

若`br-tun`之间两两点对点的连接，通信封包为`GRE`格式，那么这样的网络环境就是`OVS-GRE`网络模式。同理，若`br-tun`之间跑三层网络协议，封包方式为`VXLAN`格式，这样的网络环境就是`OpenStack-Neutron-OVS-VXLAN`网络模式。对于`GRE`和`VXLAN`网络模式而言，可以抽象地将每个`br-tun`看成隧道端点，有状态的隧道点对点连接即为`GRE`；无状态的隧道使用`UDP`协议连接则为`VXLAN`。

(http://www.sdnlab.com/11819.html)
(http://bingotree.cn/?p=654)
