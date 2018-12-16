# [kubeadm 生成的token过期后，集群增加节点](https://www.cnblogs.com/hongdada/p/9854696.html)



通过kubeadm初始化后，都会提供node加入的token:

```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 18.16.202.35:6443 --token zr8n5j.yfkanjio0lfsupc0 --discovery-token-ca-cert-hash sha256:380b775b7f9ea362d45e4400be92adc4f71d86793ba6aae091ddb53c489d218c
```

默认token的有效期为24小时，当过期之后，该token就不可用了。

#### 解决方法如下：

1. 重新生成新的token

   ```
   [root@node1 flannel]# kubeadm  token create
   kiyfhw.xiacqbch8o8fa8qj
   [root@node1 flannel]# kubeadm  token list
   TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS
   gvvqwk.hn56nlsgsv11mik6   <invalid>   2018-10-25T14:16:06+08:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
   kiyfhw.xiacqbch8o8fa8qj   23h         2018-10-27T06:39:24+08:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
   ```

2. 获取ca证书`sha256`编码hash值

   ```
   [root@node1 flannel]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
   5417eb1b68bd4e7a4c82aded83abc55ec91bd601e45734d6aba85de8b1ebb057
   ```

3. 节点加入集群

   ```
     kubeadm join 18.16.202.35:6443 --token kiyfhw.xiacqbch8o8fa8qj --discovery-token-ca-cert-hash sha256:5417eb1b68bd4e7a4c82aded83abc55ec91bd601e45734d6aba85de8b1ebb057
   ```

几秒钟后，您应该注意到`kubectl get nodes`在主服务器上运行时输出中的此节点。

#### 上面太繁琐，一步到位：

```
kubeadm token create --print-join-command
```

### 第二种方法：

```
# 第二种方法
token=$(kubeadm token generate)
kubeadm token create $token --print-join-command --ttl=0
```

##### 参考：

https://www.cnblogs.com/hongdada/p/9854696.html

<https://blog.csdn.net/mailjoin/article/details/79686934>