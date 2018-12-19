# 华为云在 K8S 大规模场景下的 Service 性能优化实践

[![Kuberneteschina](https://pic3.zhimg.com/v2-6d5cb96771aa1aeb440d4954fdac1b59_xs.jpg)](https://www.zhihu.com/people/kuberneteschina)

[Kuberneteschina](https://www.zhihu.com/people/kuberneteschina)

致力于提供最权威的 Kubernetes 技术、案例与Meetup！



12 人赞了该文章

> 讲师：王泽锋 / 华为云 Kubernetes 开源负责人
> 编辑：夏天

Kubernetes 原生的 Service 负载均衡基于 Iptables 实现，其规则链会随 Service 的数量呈线性增长，在大规模场景下对 Service 性能影响严重。本次分享介绍了华为云在 Kubernetes service 性能优化方面的探索与实践。

大家好，今天给大家带来我们在 Kubernetes Service 上的一些优化实践，这是一个网络相关的话题。首先，我将给大家介绍 Kubernetes 的 Service 机制。现在 Kubernetes 中 Service 的三种模式，包括原来的 Userspace 和 Iptables，以及后来我们贡献的 IPVS；第二部分会介绍原来社区是如何使用 Iptables 来实现 Service 负载平衡的；第三部分主要是 Iptables 实现中存在的一些问题；接下来是如何使用 IPVS 来做 Service 的负载实现的；最后是一个对比。

![img](https://pic2.zhimg.com/80/v2-d4bdb2fddbf3df1005704c060d300b39_hd.jpg)

**Kubernetes 的 Service 机制**

先看一下 Kubernetes 里面的 Service。在用 Kubernetes 之前，当我们有了容器网络之后，访问一个应用最直接的做法，就是客户端直接去访问一个 Backend Container。这种做法最直观和容易，同时问题也是显而易见的。**当应用有多个后端容器的时候，怎么做负载均衡，会话保持怎么做，某个容器迁了之后 IP 跟着变怎么办，还有对应的健康检查怎么配，如果想用域名来做访问入口要怎么处理……这些其实就是 Kubernetes 的 Service 引入所要解决的问题。**

Kubernetes Service 与 Endpoints

![img](https://pic1.zhimg.com/80/v2-9a0e8b04f8b848e1418fcf13e17b36fc_hd.jpg)

这张图表现了 Service 与其它几个对象的对应关系。首先是 Service，它保存的是服务的访问入口信息（如 IP、端口），可以简单理解为 Kubernetes 内置的一个 LoadBalancer，它的作用就是给多个 Pod 提供负载均衡。

图中是一个 Replication Controller 部署出来 2 个 pod 所对应的 Service。我们知道 RC 和 pod 的关系是通过 label-selector 来关联的，service 也是一样，通过 Selector 来匹配它所要做负载均衡的 Pod。实际上这中间还有一个对象，叫做 Endpoint，为什么要有这个对象呢？因为在实际应用中，一个 pod 被创建，并不代表它马上就能对外提供服务，而这个 pod 如果将被删除，或处于其他不良状态，我们都希望客户端的请求不被分发到这个无法提供服务的 pod 上。Endpoint 的引入，就是用来映射那些能对外提供服务的 pod。每个 Endpoints 对象的 IP 对应一个 Kubernetes 的内部域名，可以通过这个域名直接访问到具体的 pod。

![img](https://pic1.zhimg.com/80/v2-149bd0cee90ef938d5cc26e6f7cd738c_hd.jpg)

再看 Service 和 Endpoint 的定义。这里注意 Service 有一个 ClusterIP 的属性字段，可以简单理解为是虚 IP。Service 的域名解析通常得到的就是这个 ClusterIP。另外值得注意的是 Service 支持端口映射，即 Service 暴露的端口不必和容器端口一致。

Service 内部逻辑

![img](https://pic3.zhimg.com/80/v2-cb5dad4af0b020a16fe3deaa493ff582_hd.jpg)

刚才介绍了 Service、Pods 跟 Endpoint 三者的关系，再来看 Service 的内部逻辑。这里主要看下 Endpoint Controller，它会 watch Service 对象、还有 pod 的变化情况，维护对应的 Endpoint 信息。然后在每一个节点上，KubeProxy 根据 Service 和 Endpoint 来维护本地的路由规则。

实际上，每当一个 Endpoint 发生变化（即 Service 以及它关联的 Pod 状态发生变化），Kubeproxy 都会在每个节点上做对应的规则刷新，所以这个其实更像是一个靠近客户端的负载均衡——一个 Pod 访问其他服务的 Pod 时，请求在出节点之前，就已经通过本地的路由规则选好了它的目的 Pod。

**Iptables 实现负载均衡**

![img](https://pic3.zhimg.com/80/v2-99f6fa303dd5acd30e087f9efd34a8fa_hd.jpg)

好，我们来看一下 Iptables 模式是怎么实现的。

Iptables 主要分两部分，一个是它的命令行工具，在用户态；然后它也有内核模块，但本质上还是通过 Netfilter 这个内核模块来封装实现的，Iptables 的特点是支持的操作比较多。

![img](https://pic2.zhimg.com/80/v2-67a3cfa434c3918f85b61d60b1235cd5_hd.jpg)

这是 IPtables 处理网络包的一个流程图，可以看到，每个包进来都会按顺序经过几个点。首先是 PREROUTING，它会判断接收到的这个请求包，是访问本地进程还是其他机器的，如果是访问其他机器的，就要走 FORWARD 这个 chain，然后再会做一次 Routing desicion，确定它要 FORWARD 到哪里，最后经 POSTROUTING 出去。如果是访问本地，就会进来到 INPUT 这条线，找到对应要访问哪个本地请求，然后就在本地处理了。处理完之后，其实会生成一个新的数据包，这个时候又会走 OUTPUT，然后经 POSTROUTING 出去。

Iptables 实现流量转发与负载均衡

我们知道，Iptables 做防火墙是专业的，那么它是如何做流量转发、负载均衡甚至会话保持的呢？如下图所示：

![img](https://pic4.zhimg.com/80/v2-f7d38d8297ac5215f97a6fa1de2b7683_hd.jpg)

Iptables 在 Kubernetes 的应用举例

![img](https://pic4.zhimg.com/80/v2-4ed43d6b9b6ae41c0f80e4915282280f_hd.jpg)

那么，在 Kubernetes 里面是怎么用 Iptables 来实现负载均衡呢？来看一个实际的例子。在 Kubernetes 中，从VIP到RIP，中间经过的Iptables链路包括：PREROUTING/OUTPUT（取决于流量是从本机还是外机过来的）-> KUBE-SERVICES（所有 Kubernetes 自定义链的入口）->KUBE-SVC-XXX（后面那串 hash 值由 Service 的虚 IP 生成）->KUBE-SEP->XXX（后面那串 hash 值由后端 Pod 实际 IP 生成）。

**当前 Iptables 实现存在的问题**

Iptables 做负载均衡的问题

![img](https://pic2.zhimg.com/80/v2-8466f4feec190b0d646a5f585073a431_hd.jpg)

**那么 Iptables 做负载均衡主要有什么缺陷呢？**起初我们只是分析了原理，后来在大规模场景下实测，发现问题其实非常明显。

- **首先是时延，匹配时延和规则更新时延**。我们从刚刚的例子就能看出，每个 Kubernetes Service 的虚 IP 都会在 kube-services 下对应一条链。Iptables 的规则匹配是线性的，匹配的时间复杂度是 O(N)。规则更新是非增量式的，哪怕增加/删除一条规则，也是整体修改 Netfilter 规则表。
- **其次是可扩展性。**我们知道当系统中的 Iptables 数量很大时，更新会非常慢。同时因为全量提交的过程中做了保护，所以会出现 kernel lock，这时只能等待。
- **最后是可用性。**服务扩容/缩容时，Iptables 规则的刷新会导致连接断开，服务不可用。

Iptables 规则匹配时延

![img](https://pic4.zhimg.com/80/v2-74533b435b66ef6ffdfbb6f1f761962f_hd.jpg)

上图说明了 Service 访问时延随着规则数的增加而增长。但其实也还能接受，因为时延最高也就 8000us（8ms），这说明真正的性能瓶颈并不在这里。

Iptables 规则更新时延

![img](https://pic1.zhimg.com/80/v2-2d2fc08e7bc0fc26af08ef47a5ba2154_hd.jpg)

那么 Iptables 的规则更新，究竟慢在哪里呢

首先，Iptables 的规则更新是全量更新，即使 --no--flush 也不行（--no--flush 只保证 iptables-restore 时不删除旧的规则链）。

再者，kube-proxy 会周期性的刷新 Iptables 状态：先 iptables-save 拷贝系统 Iptables 状态，然后再更新部分规则，最后再通过 iptables-restore 写入到内核。当规则数到达一定程度时，这个过程就会变得非常缓慢。

出现如此高时延的原因有很多，在不同的内核版本下也有一定的差异。另外，时延还和系统当前内存使用量密切相关。因为 Iptables 会整体更新 Netfilter 的规则表，而一下子分配较大的内核内存（>128MB）就会出现较大的时延。

Iptables 周期性刷新导致 TPS 抖动

![img](https://pic2.zhimg.com/80/v2-2e2d7e375d8d892681a5fd9c0395e291_hd.jpg)

上图就说明了在高并发的 loadrunner 压力测试下，kube-proxy 周期性刷新 Iptables 导致后端服务连接断开，TPS 的周期性波动。

**K8S Scalability**

所以这个就给 Kubernetes 的数据面的性能带来一个非常大的限制，我们知道社区管理面的规模，其实在去年就已经支持到了 5000 节点，而数据面由于缺乏一个权威的定义，没有给出规格。

**我们在多个场景下评估发现 Service 个数其实很容易达到成千上万，所以优化还是很有必要的。当时先到的优化方案主要有两个**：

- 用树形结构来组织 Iptables 的规则，让匹配和规则更新过程变成树的操作，从而优化两个时延。
- 使用 IPVS，后面会讲它的好处。

使用树形结构组织 Iptables 规则的一个例子如下所示:

![img](https://pic3.zhimg.com/80/v2-81f8d30c6c4da1c9c4b3fd6b9cacada2_hd.jpg)

在这个例子中，树根是 16 位地址，根的两个子节点是 24 位地址，虚 IP 作为叶子节点，根据不同的网段，分别挂在不同的树节点下。这样，规则匹配的时延就从 O(N) 降低到 O(N 的 M 次方根)，M 即树的高度。但这么做带来的代价是 Iptables 规则变得更加复杂。

**IPVS 实现 Service 负载均衡**

什么是 IPVS

- 传输层 Load Balancer，LVS 负载均衡器的实现；
- 同样基于 Netfilter，但使用的是 hash 表；
- 支持 TCP, UDP，SCTP 协议，IPV4，IPV6；
- 支持多种负载均衡策略，如 rr, wrr, lc, wlc, sh,dh, lblc…
- 支持会话保持， persistent connection 调度算法。

IPVS 的三种转发模式

**IPVS 有三种转发模式，分别是：DR，隧道和 NAT。**

● DR 模式工作在 L2，使用的 MAC 地址，速度最快。请求报文经过 IPVS director，转发给后端服务器，响应报文直接回给客户端。缺点是不支持端口映射，于是这种模式就很可惜地 PASS 掉了。

● 隧道模式，使用 IP 包封装 IP 包。后端服务器接收到隧道包后，首先会拆掉封装的 IP 地址头，然后响应报文也会直接回给客户端。IP 模式同样不支持端口映射，于是这种模式也被 PASS 掉了。

● NAT 模式支持端口映射，与前面两种模式不同的是，NAT 模式要求回程报文经过 IPVS 的 director。内核原生版本 IPVS 只做 DNAT，不做 SNAT。

使用 IPVS 实现流量转发

使用 IPVS 做流量转发只需经过以下几个简单的步骤。

- **绑定 VIP**

由于 IPVS 的 DNAT 钩子挂在 INPUT 链上，因此必须要让内核识别 VIP 是本机的 IP。绑定 VIP 至少有三种方式：

1.创建一块 dummy 网卡，然后绑定，如下所示。

\# ip link add dev dummy0 type dummy # ip addr add 192.168.2.2/32 dev dummy0

2.直接在本地路由表中加上 VIP 这个 IP 地址。

\# ip route add to local 192.168.2.2/32 dev eth0proto kernel

3.在本地网卡上增加一个网卡别名。

\# ifconfig eth0:1 192.168.2.2netmask255.255.255.255 up

- **为这个虚 IP 创建一个 IPVS 的 virtual server**

\# ipvsadm -A -t 192.168.60.200:80 -s rr -p 600

这上面的例子中，IPVS virtual server 的虚 IP 是 192.168.60.200:80，会话保持时间 600s。

- **为这个 IPVS service 创建相应的 real server**

\# ipvsadm -a -t 192.168.60.200:80 -r 172.17.1.2:80–m

\# ipvsadm -a -t 192.168.60.200:80 -r 172.17.2.3:80–m

这上面的例子中，为 192.168.60.200:80 这个 IPVS 的 virtual server 创建了两个 real server：172.17.1.2:80 和 172.17.2.3:80。

**Iptables vs. IPVS**

Iptables vs. IPVS 规则增加时延

![img](https://pic3.zhimg.com/80/v2-7d4a896a99b7bc940d4d3f01213f6112_hd.jpg)

通过观察上图很容易发现：

- 增加 Iptables 规则的时延，随着规则数的增加呈“指数”级上升；
- 当集群中的 Service 达到 2 万个时，新增规则的时延从 50us 变成了 5 小时；
- 而增加 IPVS 规则的时延始终保持在 100us 以内，几乎不受规则基数影响。这中间的微小差异甚至可以认为是系统误差。
- Iptables vs. IPVS 网络带宽

![img](https://pic2.zhimg.com/80/v2-4fa595891c09dc3602f123d3bd129711_hd.jpg)

这是我们用 iperf 实测得到两种模式下的网络带宽。可以看到 Iptables 模式下第一个 Service 和最后一个 Service 的带宽有差异。最后一个 Service 带宽明显小于第一个，而且随着 Service 基数的上升，差异越来越明显。

而 IPVS 模式下，整体带宽表现高于 Iptables。当集群中的 Service 数量达到 2.5 万时，Iptables 模式下的带宽已基本为零，而 IPVS 模式的服务依然能够保持在先前一半左右的水平，提供正常访问。

Iptables vs. IPVS CPU/内存消耗

![img](https://pic2.zhimg.com/80/v2-8b6bb9aa6433e63962960ecdeba5a10d_hd.jpg)

很明显，IPVS 在 CPU/内存两个维度的指标都要远远低于 Iptables。

**特性社区状态**

![img](https://pic3.zhimg.com/80/v2-a644e3cd656960a3bdd464b16efad56a_hd.jpg)

**这个特性从 1.8 版本引入 Alpha，到 1.9 版本发布 Beta，修复了大部分的问题，目前已经比较稳定，强烈推荐大家使用。**另外这个特性目前主要是我们华为云 K8S 开源团队在维护，大家在使用中如果发现问题，欢迎反映到社区，或者我们这边。谢谢大家！

![img](https://pic4.zhimg.com/80/v2-d714082d85e09cf9b068187ba8490233_hd.jpg)

**王泽锋/华为云 Kubernetes 开源负责人**

多年电信领域系统软件开发和性能调优经验，对深度报文解析、协议识别颇有研究。华为云 PaaS 服务团队核心成员，专注于 PaaS 产品和容器开源社区，目前负责华为云 K8S 开源团队在社区贡献的整体工作。