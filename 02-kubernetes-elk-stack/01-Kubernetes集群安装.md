# 这里将梳理K8S的国内搭建教程

## K8S版本： V1.12.0

## 简介：

Kubernetes 

弹性扩展能力强

大规模集群可快速上线

运维管理大规模集群简单

可以提供 容器服务、代码构建、镜像仓库、服务编排、集群管理、系统管理等多元化、全方位的服务。

## 架构说明：

```
192.168.0.101 master

192.168.0.102 slave1

192.168.0.103 slave1
```

### 开启最大map

```
sysctl -w vm.max_map_count=262144
```

### 禁用SELinux

```
setenforce 0
```

编辑文件/etc/selinux/config，将SELINUX修改为disabled，如下：

```
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux

#SELINUX=disabled
```

### 关闭系统Swap

```
swapoff -a
```

修改/etc/fstab文件，注释掉SWAP的自动挂载，使用free -m确认swap已经关闭。

```
sed -i 's/.*swap.*/#&/' /etc/fstab
```

```
[root@localhost /]# free -m
total        used        free      shared  buff/cache   available
Mem:            962         154         446           6         361         612
Swap:             0           0           0
```

### 安装docker

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum makecache fast
yum list docker-ce --showduplicates | sort -r

yum -y install docker-ce-18.06.0.ce-3.el7
systemctl enable docker.service
systemctl restart docker
```

我这里安装的是docker-ce 18.06

## 使用kubeadm部署Kubernetes:

### 安装kubeadm和kubelet

下面在各节点安装kubeadm和kubelet：

```
# 配置源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装
yum makecache fast
yum -y install kubelet-1.12.0 kubeadm-1.12.0 kubectl-1.12.0 ipvsadm
```

**配置：**

```
# 配置转发相关参数，否则可能会出错
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF

# 使配置生效
 sysctl --system

# 如果net.bridge.bridge-nf-call-iptables报错，加载br_netfilter模块
 modprobe br_netfilter
 sysctl -p /etc/sysctl.d/k8s.conf

# 加载ipvs相关内核模块
# 如果重新开机，需要重新加载（可以写在 /etc/rc.local 中开机自动加载）
 modprobe ip_vs
 modprobe ip_vs_rr
 modprobe ip_vs_wrr
 modprobe ip_vs_sh
 modprobe nf_conntrack_ipv4
# 查看是否加载成功
 lsmod | grep ip_vs
```

**配置启动kubelet（所有节点）**

```
# 配置kubelet使用国内pause镜像
# 配置kubelet的cgroups
# 获取docker的cgroups
DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f3)
echo $DOCKER_CGROUPS
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF

# 启动
 systemctl daemon-reload
 systemctl enable kubelet && systemctl restart kubelet
```

查看错误：

```
journalctl -xefu kubelet 命令查看systemd日志才发现，真正的错误
```

### 配置master节点

**使用kubeadm-master.config配置文件**，在/etc/kubernetes/文件夹下面操作：

vi /etc/kubernetes/kubeadm-master.config

```
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.12.0
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers #这里是访问国内的镜像加速，因为google无法访问。
api:
  advertiseAddress: 192.168.0.101 #这里换成你的master ip

controllerManagerExtraArgs:
  node-monitor-grace-period: 10s
  pod-eviction-timeout: 10s

networking:
  podSubnet: 10.244.0.0/16  #子网的网段
  
kubeProxy:
  config:
    mode: ipvs
    # mode: iptables

```

安装过程中遇到异常：

```
[preflight] Some fatal errors occurred:
[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
```

直接删除/var/lib/etcd文件夹

**如果初始化过程出现问题，使用如下命令重置：**

kubeadm reset

rm -rf /var/lib/cni/ $HOME/.kube/config

### 初始化master节点：

kubeadm init --config  kubeadm-master.config

```
[root@master ~]# kubeadm init --config  /etc/kubernetes/kubeadm-master.config
[init] using Kubernetes version: v1.12.0
[preflight] running pre-flight checks
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [master localhost] and IPs [127.0.0.1 ::1]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [master localhost] and IPs [192.168.0.101 127.0.0.1 ::1]
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.0.101]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[certificates] Generated sa key and public key.
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests" 
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 26.505612 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.12" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node master as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node master as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "master" as an annotation
[bootstraptoken] using token: j1o3of.u5dzg9mdltb2g0ir
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.0.101:6443 --token j1o3of.u5dzg9mdltb2g0ir --discovery-token-ca-cert-hash sha256:582342d41b93f2b144b18f2cdab285705c6ddc8d04d6c449df53d9d2de391d7b

[root@master ~]# 
```

上面记录了完成的初始化输出的内容，根据输出的内容基本上可以看出手动初始化安装一个Kubernetes集群所需要的关键步骤。

其中有以下关键内容：

- `[kubelet]` 生成kubelet的配置文件”/var/lib/kubelet/config.yaml”
- `[certificates]`生成相关的各种证书
- `[kubeconfig]`生成相关的kubeconfig文件
- `[bootstraptoken]`生成token记录下来，后边使用`kubeadm join`往集群中添加节点时会用到

下面的命令是配置常规用户如何使用kubectl访问集群：

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

最后给出了将节点加入集群的命令:

```
  kubeadm join 192.168.0.101:6443 --token j1o3of.u5dzg9mdltb2g0ir --discovery-token-ca-cert-hash sha256:582342d41b93f2b144b18f2cdab285705c6ddc8d04d6c449df53d9d2de391d7b
```

### 配置使用kubectl

**如下操作在master节点操作**

```
$ rm -rf $HOME/.kube
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 查看node节点
$ kubectl get nodes
NAME    STATUS     ROLES    AGE     VERSION
node1   NotReady   master   6m19s   v1.12.0
```

### 配置使用网络插件

**如下操作在master节点操作**

```
# 下载配置
cd ~ && mkdir flannel && cd flannel
wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
```

**修改配置文件kube-flannel.yml:

```
# 修改kube-flannel.yml中配置
# 此处的ip配置要与上面kubeadm的pod-network一致
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }

# 默认的镜像是quay.io/coreos/flannel:v0.10.0-amd64，如果你能pull下来就不用修改镜像地址，否则，修改yml中镜像地址为阿里镜像源
image: registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64

# 如果Node有多个网卡的话，参考flannel issues 39701，
# https://github.com/kubernetes/kubernetes/issues/39701
# 目前需要在kube-flannel.yml中使用--iface参数指定集群主机内网网卡的名称，
# 否则可能会出现dns无法解析。容器无法通信的情况，需要将kube-flannel.yml下载到本地，
# flanneld启动参数加上--iface=<iface-name>
    containers:
      - name: kube-flannel
        image: registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=ens33
        - --iface=eth0
⚠️⚠️⚠️--iface=ens33 的值，是你当前的网卡,或者可以指定多网卡

# 1.12版本的kubeadm额外给node1节点设置了一个污点(Taint)：node.kubernetes.io/not-ready:NoSchedule，
# 很容易理解，即如果节点还没有ready之前，是不接受调度的。可是如果Kubernetes的网络插件还没有部署的话，节点是不会进入ready状态的。
# 因此我们修改以下kube-flannel.yaml的内容，加入对node.kubernetes.io/not-ready:NoSchedule这个污点的容忍：
    tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node.kubernetes.io/not-ready
        operator: Exists
        effect: NoSchedule
```

启动：

```
# 启动
  kubectl apply -f ~/flannel/kube-flannel.yml

# 查看
kubectl get pods --namespace kube-system
# kubectl get service
 kubectl get svc --namespace kube-system

# 只有网络插件也安装配置完成之后，才能会显示为ready状态
# 设置master允许部署应用pod，参与工作负载，现在可以部署其他系统组件
# 如 dashboard, heapster, efk等
$ kubectl taint nodes --all node-role.kubernetes.io/master-
# 或者 kubectl taint nodes node1 node-role.kubernetes.io/master-   
  node/node1 untainted

# master不运行pod
# kubectl taint nodes node1 node-role.kubernetes.io/master=:NoSchedule
```

### 配置node节点加入集群

**如下操作在所有node节点操作**

```
# 此命令为初始化master成功后返回的结果
  kubeadm join 192.168.0.101:6443 --token j1o3of.u5dzg9mdltb2g0ir --discovery-token-ca-cert-hash sha256:582342d41b93f2b144b18f2cdab285705c6ddc8d04d6c449df53d9d2de391d7b
```

 

查看pods:

```
[root@node1 flannel]# kubectl get pods -n kube-system
kubectl get pods --all-namespaces
NAME                            READY   STATUS             RESTARTS   AGE
coredns-6c66ffc55b-l76bq        1/1     Running            0          16m
coredns-6c66ffc55b-zlsvh        1/1     Running            0          16m
etcd-node1                      1/1     Running            0          16m
kube-apiserver-node1            1/1     Running            0          16m
kube-controller-manager-node1   1/1     Running            0          15m
kube-flannel-ds-sr6tq           0/1     CrashLoopBackOff   6          7m12s
kube-flannel-ds-ttzhv           1/1     Running            0          9m24s
kube-proxy-nfbg2                1/1     Running            0          7m12s
kube-proxy-r4g7b                1/1     Running            0          16m
kube-scheduler-node1            1/1     Running            0          16m
```

查看异常pod信息：kubectl  describe pods kube-flannel-ds-sr6tq -n  kube-system

```
[root@node1 flannel]# kubectl  describe pods kube-flannel-ds-sr6tq -n  kube-system
Name:               kube-flannel-ds-sr6tq
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
。。。。。
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Pulling    12m                  kubelet, node2     pulling image "registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64"
  Normal   Pulled     11m                  kubelet, node2     Successfully pulled image "registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64"
  Normal   Created    11m                  kubelet, node2     Created container
  Normal   Started    11m                  kubelet, node2     Started container
  Normal   Created    11m (x4 over 11m)    kubelet, node2     Created container
  Normal   Started    11m (x4 over 11m)    kubelet, node2     Started container
  Normal   Pulled     10m (x5 over 11m)    kubelet, node2     Container image "registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64" already present on machine
  Normal   Scheduled  7m15s                default-scheduler  Successfully assigned kube-system/kube-flannel-ds-sr6tq to node2
  Warning  BackOff    7m6s (x23 over 11m)  kubelet, node2     Back-off restarting failed container
```

遇到这种情况直接 删除异常pod:

```
[root@node1 flannel]# kubectl delete pod kube-flannel-ds-sr6tq -n kube-system
pod "kube-flannel-ds-sr6tq" deleted
[root@node1 flannel]# kubectl get pods -n kube-system
NAME                            READY   STATUS    RESTARTS   AGE
coredns-6c66ffc55b-l76bq        1/1     Running   0          17m
coredns-6c66ffc55b-zlsvh        1/1     Running   0          17m
etcd-node1                      1/1     Running   0          16m
kube-apiserver-node1            1/1     Running   0          16m
kube-controller-manager-node1   1/1     Running   0          16m
kube-flannel-ds-7lfrh           1/1     Running   1          6s
kube-flannel-ds-ttzhv           1/1     Running   0          10m
kube-proxy-nfbg2                1/1     Running   0          7m55s
kube-proxy-r4g7b                1/1     Running   0          17m
kube-scheduler-node1            1/1     Running   0          16m
```

查看节点：

```
[root@node1 flannel]# kubectl get nodes -n kube-system
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   17m     v1.12.1
node2   Ready    <none>   8m14s   v1.12.1
```

## 参考：

https://www.cnblogs.com/hongdada/p/9761336.html

<https://www.cnblogs.com/liangDream/p/7358847.html>

<https://my.oschina.net/binges/blog/1615955?p=2&temp=1521445654544>

**https://blog.frognew.com/2018/10/kubeadm-install-kubernetes-1.12.html**

**https://www.jianshu.com/p/31bee0cecaf2**

<https://www.zybuluo.com/ncepuwanghui/note/953929>

<https://www.kubernetes.org.cn/4256.html>

<https://note.youdao.com/share/?id=31d9d5db79cc3ae27e72c029b09ac4ab&type=note#/>

**https://juejin.im/post/5b45d4185188251ac062f27c**

<https://www.jianshu.com/p/02dc13d2f651>

<https://blog.csdn.net/qq_34857250/article/details/82562514>

<https://www.cnblogs.com/ssss429170331/p/7685044.html>

<https://imroc.io/posts/kubernetes/install-kubernetes-1.9-on-centos7-with-kubeadm/>



 

 



 

