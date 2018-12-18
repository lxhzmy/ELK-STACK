# kubernetes基本概念和术语

我们先了解一下Kubernetes集群的两种管理角色：`Master`和`Node`

## Master

Kubernetes里的Master指的是集群控制节点，每个Kubernetes集群里需要有一个Master节点来负责整个集群的管理和控制，基本上Kubernetes的所有控制命令都发给它，它来负责具体的执行过程，我们后面执行的所有命令基本都是在Master节点上运行的。Master节点通常会占据一个独立的服务器（高可用部署建议用3台服务器），其主要原因是它太重要了，是整个集群的“首脑”，如果宕机或者不可用，那么对集群内容器应用的管理都将失效。

## Node

除了Master，Kubernetes集群中的其他机器被称为Node节点，在较早的版本中也被称为Minion。与Master一样，Node节点可以是一台物理主机，也可以是一台虚拟机。Node节点才是Kubernetes集群中的工作负载节点，每个Node都会被Master分配一些工作负载（Docker容器），当某个Node宕机时，其上的工作负载会被Master自动转移到其他节点上去。

## Pod

 我们看到每个Pod都有一个特殊的被成为“根容器”的Pause容器。Pause容器对应的镜像属于Kubernetes平台的一部分，除了Pause容器，每个Pod还包含一个或多个紧密相关的用户业务容器

 Kubernetes为每个Pod都分配了唯一的IP地址，称之为Pod IP，一个Pod里的多个容器共享Pod IP地址。Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟而层网络技术来实现，例如Flannel、Open vSwitch等，因此我们需要牢记一点：在Kubernetes里，一个Pod里的容器与另外主机上的Pod容器能够直接通信。

  Pod其实有两种类型：普通的Pod及静态Pod（Static Pod），后者比较特殊，它并不存放在Kubernetes的etcd存储里，而是存放在某个具体的Node上的一个具体文件中，并且只在此Node上启动运行。而普通的Pod一旦被创建，就会被放入到etcd中存储，随后会被Kubernetes Master调度到某个具体的Node上并进行绑定（Binding），随后该Pod被对应的Node上的kubelet进程实例化成一组相关的Docker容器并且启动起来。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问题并且重新启动这个Pod（重启Pod里的所有容器），如果Pod所在的Node宕机，则会将这个Node上的所有Pod重新调度到其他节点上。Pod、容器与Node的关系图如图所示。![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFv0IS-AbqyhAACfzxoTgIY518.png)

  

## 资源限额

每个Pod都可以对其能使用的服务器上的计算资源设置限额，当前可以设置限额的计算资源有CPU与Memory两种，其中CPU的资源单位为CPU （Core）的数量，是一个绝对值而非相对值。

一个CPU的配额对于绝大多数容器来说是相当大的一个资源配额了，所以，在Kubernetes里，通常以千分之一的CPU配额为最小单位，用m来表示。通常一个容器的CPU配额被定义为100～300m，即占用0.1~0.3个CPU。由于CPU配额是一个绝对值，所以无论在拥有一个Core的机器上，还是在拥有48个Core的机器上，100m这个配额所代表的CPU的使用量都是一样的。与CPU配额类似，Memory配额也是一个绝对值，它的单位是内存字节数。

在Kubernetes里，一个计算资源进行配额限定需要设定以下两个参数。

- `Request`：该资源的最小申请量，系统必须满足要求。
- `Limits`：该资源最大允许使用量，不能被突破，当容器试图使用超过这个量的资源时，可能会被Kubernetes Kill并重启。

通常我们会把Request设置为一个比较小的数值，符合容器平时的工作负载情况下的资源需求，而把Limit设置为负载均衡情况下资源占用的最大量。比如下面这些定义，表明MySQL容器申请最少0.25个CPU及64MiB内存，在运行过程中MySQL容器所能使用的资源配额为0.5个CPU及128MiB内存：

```
spec:
   containers:
   - name: db
     image: mysql
     resources:
       request:
        memory: "64Mi"
        cpu: "250m"
       limits:
        memory: "128Mi"
        cpu: "500m"
```

## Label

Label是Kubernetes系统中另外一个核心概念。一个Label是一个key=value的键值对，其中key与vaue由用户自己指定。Label可以附加到各种资源对象上，例如Node、Pod、Service、RC等，一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上去，Label通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。

我们可以通过指定的资源对象捆绑一个或多个不同的Label来实现多维度的资源分组管理功能，以便于灵活、方便地进行资源分配、调度、配置、部署等管理工作。例如：部署不同版本的应用到不同的环境中；或者监控和分析应用（日志记录、监控、告警）等。一些常用等label示例如下。

- 版本标签："release" : "stable" , "release" : "canary"...
- 环境标签："environment" : "dev" , "environment" : "production"
- 架构标签："tier" : "frontend" , "tier" : "backend" , "tier" : "middleware"
- 分区标签："partition" : "customerA" , "partition" : "customerB"...
- 质量管控标签："track" : "daily" , "track" : "weekly"

Label相当于我们熟悉的“标签”，給某个资源对象定义一个Label，就相当于給它打了一个标签，随后可以通过Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象，Kubernetes通过这种方式实现了类似SQL的简单又通用的对象查询机制。

Label Selector可以被类比为SQL语句中的where查询条件，例如，name=redis-slave这个label Selector作用于Pod时，可以被类比为select * from pod where pod's name = 'redis-slave'这样的语句。当前有两种Label Selector的表达式：基于等式的（Equality-based）和基于集合的（Set-based），前者采用“等式类”的表达式匹配标签，下面是一些具体的例子。

- name=redis-slave：匹配所有具有标签name=redis-slave的资源对象。
- env != production：匹配所有不具有标签env=production的资源对象，比如env=test就满足此条件的标签之一。
- name in (redis-master)：匹配所有具有标签name=redis-master或者name=redis-slave的资源对象。
- name not in (php-frontend)：匹配所有不具有标签name=php-frontend的资源对象。

可以通过多个Label Selector表达式的组合实现复杂的条件，多个表达式之间用“,”进行分隔即可，几个条件之间是“AND”的关系，即同时满足多个条件，比如下面的例子：

```
name=redis-slave,env!=production
name notin (php-fronted),env!=production
```

以myweb Pod为例，Label定义在其metadata中：

```
apiVersion: v1
kind: Pod
metadata:
    name: myweb
    labels:
        app: myweb
```

管理对象RC和Service在spec中定义Selector与Pod进行关联：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 1
  selector:
    app: myweb
  template:
  ...略...

apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  selector:
    app: myweb
  ports:
  - port: 8080
```

新出现的管理对象如Deploment、ReplicaSet、DaemonSet和Job则可以在Selector中使用基于集合的筛选条件定义，例如：

```
selector:
  matchLabels:
    app: myweb
  matchExpressions:
    - {key: tier, operator: In, values: [frontend]}
    - {key: environment, operator: NorIn, values: [dev]}
```

matchLabels用于定义一组Label，与直接写在Selector中作用相同：matchExpression用于定义一组基于集合的筛选条件，可用的条件运算符包括：In、NotIn、Exists和DoesNotExist。

如果同时设置了matchLabels和matchExpression，则两组条件为“AND”关系，即所有条件需要满足才能完成Selector的筛选。

Label Selector在Kubernetes中多重要使用场景有以下几处。

- kube-controller进程通过资源对象RC上定义都Label Selector来筛选要监控的Pod副本的数量，从而实现Pod副本的数量始终符合预期设定的全自动控制流程。
- kube-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立起每个Service到对应Pod的请求转发路由表，从而实现Service的智能负载均衡机制。
- 通过对某些Node定义特定的Label，并且在Pod定义文件中使用NodeSelector这种标签调度策略，kube-scheduler进程可以实现Pod“定向调度”的特性。

在前面的留言板例子中，我们只使用了一个`name=XXX`的`Label Selector`。让我们看一个更复杂的例子。假设为Pod定义了Label: release、env和role，不同的Pod定义了不同的Label值，如图1.7所示，如果我们设置了“role=frontend”的Label Selector，则会选取到Node 1和Node 2上到Pod。

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFv0Is2AfboAAACHQ83n5vE958.png)

而设置“release=beta”的Label Selector，则会选取到Node 2和Node 3上的Pod，如图所示。

总结：使用Label可以給对象创建多组标签，Label和Label Selector共同构成了Kubernetes系统中最核心的应用模型，使得被管理对象能够被精细地分组管理，同时实现了整个集群的高可用性。

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFv0IvCAGtgRAACVc5xjKNI178.png)

## Replica Set

- 在大多数情况下，我们通过定义一个RC实现Pod的创建过程及副本数量的自动控制。

- RC里包括完整的Pod定义模版。

- RC通过Label Selector机制实现对Pod副本的自动控制。

- 通过改变RC里的Pod副本数量，可以实现Pod的扩容或缩容功能。

- 通过改变RC里的Pod模版中的镜像版本，可以实现Pod的**滚动升级**功能。

## 滚动升级

当我们的应用升级时，通常会通过Build一个新的Docker镜像，并用新的镜像版本来替代旧的版本的方式达到目的，在系统升级的过程中，我们希望是平滑的方式，比如当前系统中10个对应的旧版本的Pod，最佳的方式是旧版本的Pod每次停止一个，同时创建一个新版本的Pod，在整个升级过程中，此消彼长，而运行中的Pod数量始终是10个，几分钟以后，当所有的Pod都已经是最新版本时，升级过程完成。通过RC的机制，Kubernetes很容易就实现了这种高级实用的特性，被称为“滚动升级”（Rolling Update），具体的操作方法详见第四章。

## Deployment

Deployment的典型使用场景有以下几个。

- 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建过程。
- 检查Deployment的状态来看部署动作是否完成（Pod副本的数量是否达到预期的值）。
- 更新Deployment以创建新的Pod（比如镜像升级）。
- 如果当前Deployment不稳定，则回滚到一个早先的Deployment版本。
- 暂停Deployment以便于一次性修改多个PodTemplateSpec的配置项，之后再恢复Deployment，进行新的发布。
- 扩展Deployment以应对高负载。
- 查看Deployment的状态，以此作为发布是否成功的指标。
- 清理不再需要的旧版本ReplicaSets。

 Deployment的定义与Replica Set的定义很类似，除了API声明与Kind类型等有所区别：

```
apiVersion: extensions/v1beta1      apiVersion: v1
kind: Deployment                    kind: ReplicaSet
metadata:                           metadata:
  name: nginx-deployment              name: nginx-repset
```

 下面我们通过运行一些例子来一起直观地感受这个新概念。首先创建一个名为tomcat-deployment.yaml的Deployment描述文件，内容如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      containers:
      - name: tomcat-demo
        image: tomcat
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
 运行下述命令创建Deploymeny:

# kubectl create -f tomcat-deployment.yaml
deployment "tomcat-deploy" created
 运行下述命令查看Deployment的信息：

# kubectl get deployments
NAME            DESIRED     CURRENT     UP-TO-DATE      AVAILABLE   AGE
tomcat-deploy   1           1           1               1           4m
```

对上述输出中涉及的数量解释如下：

- DESIRED：Pod副本数量的期望值，即Deployment里定义的Replica。
- CURRENT：当前Replica的值，实际上是Deployment所创建的Replica Set里的Replica值，这个值不断增加，直到达到DESIRED为止，表明整个部署过程完成。
- UP-TO-DATE：最新版本的Pod副本数量，用于指示在滚动升级的过程中，有多少个Pod副本已经成功升级。
- AVAILABLE：当前集群中可用的Pod副本数量，即集群中当前存活的Pod数量。

##  自动扩容

Horizontal Pod Autoscaling（HPA）

## 为什么选择相对比率

为了简便，选用了相对比率（90%的CPU资源）而不是`0.6`个`CPU core`来描述扩容、缩容条件。如果选择使用绝对度量，用户需要保证目标（限额）要比请求使用的低，否则，过载的Pod未必能够消耗那么多，从而自动扩容永远不会被触发：假设设置CPU为1个核，那么这个pod只能使用1个核，可能Pod在过载的情况下也不能完全利用这个核，所以扩容不会发生。在修改申请资源时，还有同时调整扩容的条件，比如将1个core变为1.2core，那么扩容条件应该同步改为1.2core，真是太麻烦了，与自动扩容的目标相悖。

下面是HPA定义的一个具体例子：

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: java-apache
  namespace: default
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    kind: Deployment
    name: java-apache
  targetCPUUtilizationPercentage: 90
```

根据上面的定义，我们可以知道，这个HPA控制的目标对象为一个名叫`java-apache`的Deployment里的Pod副本，当这些Pod副本的`CPUUtilizationPercentage`的值超过90%时会触发自动动态扩容行为，扩容或缩容时必须满足的一个约束条件是Pod的副本数要介于1与10之间。

```
kubectl autoscale deployment java-apache --cpu-percent=90 --min=1 --max=10
```

## StatefulSet

在Kubernetes系统中，Pod的管理对象RC、Deployment、DaemonSet和Job都是面向无状态的服务。但现实中有很多服务是有状态的，特别是一些复杂的中间件集群，例如MySQL集群、MongoDB集群、Kafka集群、Zookeeper集群等，这些应用集群有以下一些共同点。

- 每个节点都有固定的身份ID，通过这个ID，集群中的成员可以相互发现并且通信。
- 集群的规模是比较固定的，集群规模不能随意变动。
- 集群里的每个节点都是有状态的，通常会持久化数据到永久存储中。
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损。

 Deployment/RC的一个特殊变种，它有如下一些特性。

- StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设StatefulSet的名字叫kafka，那么第一个Pod叫kafak-0，第二个Pod叫kafak-1，以此类推。
- StatefulSet控制的Pod副本的启停顺序是受控的，操作第n个Pod时，前n-1个Pod已经时运行且准备好的状态。
- StatefulSet里的Pod采用稳定的持久化存储卷，通过PV/PVC来实现，删除Pod时默认不会删除与StatefulSet相关的存储卷（为了保证数据的安全）。

## NodePort 

为了更好深入地理解和掌握Kubernetes，我们需要弄明白Kubernetes里的“三种IP”这个关键问题，这三种分别如下。

- Node IP：Node节点的IP地址。
- Pod IP：Pod的IP地址。
- Cluster IP：Service的IP地址。

首先，Node IP是Kubernetes集群中每个节点的物理网卡的IP地址，这是一个真实存在的物理网络，所有属于这个网络的服务器之间都能通过这个网络直接通信，不管它们中是否有部分节点不属于这个Kubernetes集群。这也表明了Kubernetes集群之外的节点访问Kubernetes集群之内的某个节点或者TCP/IP服务时，必须要通过Node IP进行通信。

 最后，我们说说Service的Cluster IP，它也是一个虚拟的IP，但更像是一个“伪造”的IP网络，原因有以下几点。

- Cluster IP仅仅作用于Kubernetes Service这个对象，并由Kubernetes管理和分配IP地址（来源于Cluster IP地址池）。
- Cluster IP无法被Ping，因为没有一个“实体网络对象”来响应。
- Cluster IP只能结合Service Port组成一个具体的通信端口，单独的Cluster IP不具备TCP/IP通信的基础，并且它们属于Kubernetes集群这样一个封闭的空间，集群之外的节点如果要访问这个通信端口，则需要做一些额外的工作。
- 在Kubernetes集群之内，Node IP网、Pod IP网与Clsuter IP之间的通信，采用的是Kubernetes自己设计的一种编程方式的特殊的路由规则，与我们所熟知的IP路由有很大的不同。

根据上面的分析和总结，我们基本明白了：Service的Cluster IP属于Kubernetes集群内部的地址，无法在集群外部直接使用这个地址。那么矛盾来了：实际上我们开发的业务系统中肯定多少由一部分服务是要提供給Kubernetes集群外部的应用或者用户来使用的，典型的例子就是Web端的服务模块，比如上面的tomcat-service，那么用户怎么访问它？

采用NodePort是解决上述问题的最直接、最常用的做法。具体做法如下，以tomcat-service为例，我们在Service的定义里做如下扩展即可（黑体字部分）：

```
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  type: NodePort
  ports:
   - port: 8080
     nodePort: 31002
  selector:
    tier: frontend
```

其中，nodePort:31002这个属性表明我们手动指定tomcat-service的NodePort为31002，否则Kubernetes会自动分配一个可用的端口。接下来，我们在浏览器里访问`http://<nodePort IP>:31002`，就可以看到Tomcat的欢迎界面了，如图1.14所示。

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFv0LHiAK4xJAANr1wikilU976.png)
通过NodePort访问Service

NodePort的实现方式是在Kubernetes集群里的每个Node上为需要外部访问的Service开启一个对应的TCP监听端口，外部系统只要用任意一个Node的IP地址+具体的NodePort端口号即可访问此服务，在任意Node上运行netstat命令，我们就可以看到有NodePort端口被监听：

```
# netstat -tlp|grep 31002
tcp6       0      0 [::]:31002              [::]:*                  LISTEN      19043/kube-proxy
```

但NodePort还没有完全解决外部访问Service的所有问题，比如负载均衡问题，假如我们的集群中有10个Node，则此时最好有一个负载均衡器，外部的请求只需要访问此负载均衡器的IP地址，由负载均衡负责转发流量到后面某个Node的NodePort上。如图1.15所示。

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFv0LJyAEg4DAACO7Y4xANo549.png)
NodePort与Load balancer

图中的Load balancer组件独立于Kubernetes集群之外，通常是一个硬件的负载均衡器，或者是以软件方式实现的，例如HAProxy或者Nginx。对于每个Service，我们通常需要配置一个对应的Load balancer实例来转发流量到后端的Node上，这的确增加了工作量及出错的概率。于是Kubernetes提供了自动化的解决方案，如果我们的集群运行在谷歌的GCE公有云上，那么只要我们把Service的type=NodePort改为type=LoadBalancer，此时Kubernetes会自动创建一个对应的Load balancer实例并返回它的IP地址供外部客户端使用。此时Kubernetes会自动创建一个对应的Load balancer实例并返回它的IP地址供外部客户端使用。其他公有云提供商只要实现了支持此特性的驱动，则也可以达到上述目的。此外，裸机上的类似机制（Bare Metal Service Load Balancers）也正在被开发

##  Volume(存储卷)

Volume是Pod中能够被多个容器访问的共享目录。Kubernetes的Volume概念、用途和目的与Docker的Volume比较类似，但两者不能等价。首先，Kubernetes中的Volume定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下；其次，Kubernetes中的Volume中的数据也不会丢失。最后，Kubernetes支持多种类型的Volume，例如Gluster、Ceph等先进的分布式文件系统。

Volume的使用也比较简单，在大多数情况下，我们先在Pod上声明一个Volume，然后在容器里引用该Volume并Mount到容器里的某个目录上。举例来说，我们要給之前的Tomcat Pod增加一个名字为datavol的Volume，并且Mount到容器的/mydata-data目录上，则只要对Pod的定义文件做如下修正即可（注意黑体字部分）：

```
template:
  metadata:
    labels:
      app: app-demo
      tier: frontend
  spec:
    volumes:
    - name: datavol
      emptyDir: {}
    containers:
    - name: tomcat-demo
      image: tomcat
      volumeMounts:
       - mountPath: /mydata-data
         name: datavol
      imagePullPolicy: IfNotPersent
```

除了可以让一个Pod里的多个容器共享文件、让容器的数据写到宿主机的磁盘上或者写文件到网络存储中，Kubernetes的Volume还扩展出了一种非常有实用价值的功能，即容器配置文件集中化定义与管理，这是通过ConfigMap这个新的资源对象来实现的，后面我们会详细说明。
 Kubernetes提供了非常丰富的Volume类型，下面逐一进行说明。

### 1.emptyDir

一个emptyDir Volume是在Pod分配到Node时创建的。从它的名称就可以看出，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是Kubernetes自动分配的一个目录，当Pod从Node上移除时，emptyDir中的数据也会被永久删除。empty的一些用途如下。

临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留。
长时间任务的中间过程CheckPoint的临时保存目录。

一个容器需要从另一个容器中获取数据的目录（多容器共享目录）。

目前，用户无法控制emptyDir使用的介质种类。如果kubelet的配置是使用硬盘，那么所有emptyDir都将创建在该硬盘上。Pod在将来可以设置emptyDir是位于硬盘、固态硬盘上还是基于内存的tmpfs上，上面的例子便采用了emptyDir类的Volume。

### 2.hostPath

hostPath为在Pod上挂载宿主机上的文件或目录，它通常可以用于以下几方面。

1. 容器应用程序生成的日志文件需要永久保持时，可以使用宿主机的高速文件系统进行存储。
2. 需要访问宿主机上Docker引擎内部数据结构的容器应用时，可以通过定义hostPath为宿主机/var/lib/docker目录，使容器内部应用可以直接访问Docker的文件系统。

在使用这种类型的Volume时，需要注意以下几点：

1. 在不同的Node上具有相同配置的Pod可能会因为宿主机上的目录和文件不同而导致对Volume上目录和文件的访问结构不一致。
2. 如果使用了资源配额管理，则Kubernetes无法将hostPath在宿主机上使用的资源纳入管理。

在下面对例子中使用宿主机的/data目录定义了一个hostPath类型的Volume：

```
volumes:
- name: "persistent-storage"
  hostPath:
    path: "/data"
```

##  persistent volumes

​	 可参考：http://orchome.com/1278



## Namespace 

Namespace（命名空间）是Kubernetes系统中的另一个非常重要的概念，Namespace在很多情况下用于实现多租户的资源隔离。Nameaspace通过将集群内部的资源对象“分配”到不同的Namespce中，形成逻辑上分组的不同项目、小组或用户组，便于不同的分组在共享使用整个集群的资源的同时还能被分别管理。

Kubernetes集群在启动后，会创建一个名为“default”的Namespace，通过kubectl可以查看到：

```
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    21h
docker        Active    21h
kube-public   Active    21h
kube-system   Active    21h
```

接下来，如果不特别指明Namespace，则用户创建的Pod、RC、Service都被系统创建到这个默认的名为default的Namespace中。
 Namespace的定义很简单。如下所示的yaml定义了名为development的Namespace。

```
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

 一旦创建了Namespace，我们在创建资源对象时就可以指定这个资源对象属于哪个Namespace。比如在下面的例子中，我们定义了一个名为busybox的Pod，放人development这个Namespace里：

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: development
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
```

此时，使用kubectl get命令查看将无法显示：

```
# kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
```

这是因为如果不加参数，则kubectl get 命令将仅显示属于“default”命名空间的资源对象。

可以在kubectl命令中加入--namespace参数来查看某个命名空间中的对象：

```
# kubectl get pods --namespace=development
NAME      READY     STATUS    RESTARTS   AGE
busybox   1/1       Running   0          2m
```

当我们給每个租户创建一个Namespace来实现多租户的资源隔离时，还能结合Kubernetes的资源配额管理，限定不同租户能占用的资源，例如CPU使用量、内存使用量等。关于资源配额管理等问题，

##  Annotation

用Annotation来记录的信息如下。

- build信息、release信息、Docker镜像信息等，例如时间戳、release id号、PR号、镜像hash值、docker registry地址等。
- 日志库、监控库、分析库等资源库的地址信息。
- 程序调试工具信息，例如工具、版本号等。
- 团队等联系信息，例如电话号码、负责人名称、网址等

 