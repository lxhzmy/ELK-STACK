# [IPVS负载均衡](https://www.cnblogs.com/hongdada/p/9758939.html)



#### 概念：

**ipvs (IP Virtual Server)** 实现了传输层负载均衡，也就是我们常说的4层`LAN`交换，作为 Linux 内核的一部分。`ipvs`运行在主机上，在真实服务器集群前充当负载均衡器。`ipvs`可以将基于`TCP`和`UDP`的服务请求转发到真实服务器上，并使真实服务器的服务在单个 IP 地址上显示为虚拟服务。

### ipvs vs. iptables

我们知道`kube-proxy`支持 iptables 和 ipvs 两种模式， 在`kubernetes` v1.8 中引入了 ipvs 模式，在 v1.9 中处于 beta 阶段，在 v1.11 中已经正式可用了。iptables 模式在 v1.1 中就添加支持了，从 v1.2 版本开始 iptables 就是 kube-proxy 默认的操作模式，ipvs 和 iptables 都是基于`netfilter`的，那么 ipvs 模式和 iptables 模式之间有哪些差异呢？

- ipvs 为大型集群提供了更好的可扩展性和性能
- ipvs 支持比 iptables 更复杂的复制均衡算法（最小负载、最少连接、加权等等）
- ipvs 支持服务器健康检查和连接重试等功能

### ipvs 依赖 iptables

ipvs 会使用 iptables 进行包过滤、SNAT、masquared(伪装)。具体来说，ipvs 将使用`ipset`来存储需要`DROP`或`masquared`的流量的源或目标地址，以确保 iptables 规则的数量是恒定的，这样我们就不需要关心我们有多少服务了

### LVS调度算法：

**1. 轮叫调度 rr**
这种算法是最简单的，就是按依次循环的方式将请求调度到不同的服务器上，该算法最大的特点就是简单。轮询算法假设所有的服务器处理请求的能力都是一样的，调度器会将所有的请求平均分配给每个真实服务器，不管后端 RS 配置和处理能力，非常均衡地分发下去。

**2. 加权轮叫 wrr**
这种算法比 rr 的算法多了一个权重的概念，可以给 RS 设置权重，权重越高，那么分发的请求数越多，权重的取值范围 0 – 100。主要是对rr算法的一种优化和补充， LVS 会考虑每台服务器的性能，并给每台服务器添加要给权值，如果服务器A的权值为1，服务器B的权值为2，则调度到服务器B的请求会是服务器A的2倍。权值越高的服务器，处理的请求越多。

**3. 最少链接 lc**
这个算法会根据后端 RS 的连接数来决定把请求分发给谁，比如 RS1 连接数比 RS2 连接数少，那么请求就优先发给 RS1

**4. 加权最少链接 wlc**
这个算法比 lc 多了一个权重的概念。

**5. 基于局部性的最少连接调度算法 lblc**
这个算法是请求数据包的目标 IP 地址的一种调度算法，该算法先根据请求的目标 IP 地址寻找最近的该目标 IP 地址所有使用的服务器，如果这台服务器依然可用，并且有能力处理该请求，调度器会尽量选择相同的服务器，否则会继续选择其它可行的服务器

**6. 复杂的基于局部性最少的连接算法 lblcr**
记录的不是要给目标 IP 与一台服务器之间的连接记录，它会维护一个目标 IP 到一组服务器之间的映射关系，防止单点服务器负载过高。

**7. 目标地址散列调度算法 dh**
该算法是根据目标 IP 地址通过散列函数将目标 IP 与服务器建立映射关系，出现服务器不可用或负载过高的情况下，发往该目标 IP 的请求会固定发给该服务器。

**8. 源地址散列调度算法 sh**
与目标地址散列调度算法类似，但它是根据源地址散列算法进行静态分配固定的服务器资源。

### LVS三种模式对比:

![img](https://www.centos.bz/wp-content/uploads/2017/09/3.png)

### ipvsadm参数：

```
添加虚拟服务器
    语法:ipvsadm -A [-t|u|f]  [vip_addr:port]  [-s:指定算法]
    -A:添加
    -t:TCP协议
    -u:UDP协议
    -f:防火墙标记
    -D:删除虚拟服务器记录
    -E:修改虚拟服务器记录
    -C:清空所有记录
    -L:查看
添加后端RealServer
    语法:ipvsadm -a [-t|u|f] [vip_addr:port] [-r ip_addr] [-g|i|m] [-w 指定权重]
    -a:添加
    -t:TCP协议
    -u:UDP协议
    -f:防火墙标记
    -r:指定后端realserver的IP
    -g:DR模式
    -i:TUN模式
    -m:NAT模式
    -w:指定权重
    -d:删除realserver记录
    -e:修改realserver记录
    -l:查看
通用:
    ipvsadm -ln:查看规则
    service ipvsadm save:保存规则
```

#### 负载均衡器端：

```
安装LVS
    [root@lb01 ~]#yum -y install ipvsadm 
    [root@lb01 ~]#ipvsadm  
添加绑定VIP
    [root@lb01 ~]#ip addr add 192.168.0.89/24 dev eth0 label eth0:1
配置LVS-DR模式
    [root@lb01 ~]#ipvsadm -A -t 192.168.0.89:80 -s rr //创建一个DR，并指定调度算法采用rr。
    [root@lb01 ~]#ipvsadm -a -t 192.168.0.89:80 -r 192.168.0.93 -g  //添加RS
    [root@lb01 ~]#ipvsadm -a -t 192.168.0.89:80 -r 192.168.0.94 -g  //添加RS
```

**Real-Server端**

```
配置测试后端realserver
    配置httpd省略
    [root@realserver-1 ~]#curl 192.168.0.93 #测试realserver-1网站是否正常    
    192.168.0.93
    [root@realserver-2 ~]#curl 192.168.0.94 #测试realserver-2网站是否正常
    192.168.0.94
绑定VIP到lo网卡
    [root@realserver-1 ~]#ip addr add 192.168.0.89/32 dev lo label lo:1  #由于DR模式需要realserver也有VIP
抑制ARP
    [root@realserver-1 ~]#echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce  
    [root@realserver-1 ~]#echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
    [root@realserver-1 ~]#echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
    [root@realserver-1 ~]#echo 1 >/proc/sys/net/ipv4/conf/lo/arp_ignore  
```

**客户端测试**

```
[root@test ~]#curl 192.168.0.89
192.168.0.93
[root@test ~]#curl 192.168.0.89
192.168.0.94
```

#### 参考：

https://www.cnblogs.com/hongdada/p/9758939.html

<https://blog.csdn.net/qq_15437667/article/details/50644443>

<https://www.centos.bz/2017/09/lvs-intro-and-lvs-keepalived/>

<http://blog.maxkit.com.tw/2016/05/lvs-lvs-natlvs-tunlvs-dr.html>

<https://jishu.io/kubernetes/ipvs-loadbalancer-for-kubernetes/>

<https://blog.qikqiak.com/post/how-to-use-ipvs-in-kubernetes/>

<https://www.opsdev.cn/post/IPVSinKube-proxy.html>

<https://segmentfault.com/a/1190000016333317>

<https://www.cnblogs.com/liwei0526vip/p/6370103.html>