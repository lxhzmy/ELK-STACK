## Pod

Pod是一组紧密关联的容器集合，它们共享PID、IPC、Network和UTS namespace，是Kubernetes调度的基本单位。Pod的设计理念是支持多个容器在一个Pod中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

缺点: 不支持高并发, 高可用, 当Pod当机后无法自动恢复.

1.创建Pod

```
# vi pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
  - image: httpd
    name: httpd
    imagePullPolicy: Always
```

kubectl create -f pod.yaml

2.查看Pod

```
kubectl get pods
NAME    READY     STATUS    RESTARTS   AGE
demo    1/1       Running      0       8d

kubectl describe pods
...
```

3.删除Pod

```
kubectl delete pod demo
```

## ReplicaSet

Replicaset在继承Pod的所有特性的同时, 它可以利用预先创建好的模板定义副本数量并自动控制, 通过改变Pod副本数量实现Pod的扩容和缩容

缺点: 无法修改template模板, 也就无法发布新的镜像版本

1.创建Replicaset

```
# vi replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: demo-rc
  labels:
    app: demo-rc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-rc
  template:
    metadata:
      labels:
        app: demo-rc
    spec:
      containers:
      - name: httpd
        image: httpd
        imagePullPolicy: Always

# kubectl create -f replicaset.yaml
```

2.查看replicaset

```
# kubectl get replicaset
NAME      READY     STATUS    RESTARTS   AGE
demo-rc    1/1       Running      0       8d
# kubectl describe replicaset
...
```

3.删除replicaset

```
# kubectl delete replicaset demo-rc
```

##  Deployment

Deployment在继承Pod和Replicaset的所有特性的同时, 它可以实现对template模板进行实时滚动更新并具备我们线上的Application life circle的特性。

1.创建Deployment

```
# vi deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
  labels:
    app: httpd-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd-demo
  template:
    metadata:
      labels:
        app: httpd-demo
    spec:
      containers:
      - name: httpd
        image: httpd
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v1"
# kubectl create -f deployment.yaml
```

2.查看Deployment

```
# kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
httpd-deployment   2         2         2            2           8d
# kubectl get pods -o wide
NAME                               READY     STATUS    RESTARTS   AGE       IP            NODE
httpd-deployment-956697567-8mqch   1/1       Running   0          8d        10.244.0.36   kube-master
httpd-deployment-956697567-wcbs6   1/1       Running   0          8d        10.244.0.37   kube-master
# kubectl describe deployment
...
```

3.更新deployment

通过此命令可以呼出vi编辑器对模板进行编辑.

```
# kubectl edit -f deployment.yaml
```

通过此命令使当前编辑结果生效.

```
# kubectl apply -f deployment.yaml
```

再次查看可以看到老版本的deployment已经下架, 新版本的已经生效.

```
# kubectl get deployment
NAME                          DESIRED   CURRENT   READY     AGE
httpd-deployment-6b98d94474   0         0         0         1m
httpd-deployment-956697567    2         2         2         7m
```

4.扩容与缩容

可以修改replicas的赋值对deployment进行扩容与缩容

```
#  kubectl scale deployment/httpd-deployment --replicas=1
```

5.删除deployment

```
# kubectl delete deployment httpd-deployment
```

## Pod定义详解

yaml格式的Pod定义文件的完整内容：

```
apiVersion: v1
kind: Pod
metadata:
  name: string
  namespace: string
  labels:
    - name: string
  annotations:
    - name: string
spec:
  containers:
  - name: string
    image: string
    imagePullPolicy: [Always | Never | IfNotPresent]
    command: [string]
    args: [string]
    workingDir: string
    volumeMounts:
    - name: string
      mountPath: string
      readOnly: boolean
    ports:
    - name: string
      containerPort: int
      hostPort: ing
      protocol: string
    env:
    - name: string
      value: string
    resources:
      limits:
        cpu: string
        memory: string
      requests:
        cpu: string
        memory: string
    livenessProbe:
      exec:
        command: [string]
      httpGet:
        path: string
        port: number
        host: string
        scheme: string
        httpHeaders:
        - name: string
          value: string
      tcpSocket:
        port: number
      initialDelaySeconds: 0
      timeoutSeconds: 0
      periodSeconds: 0
      successThreshold: 0
      failureThreshold: 0
      securityContext:
        privileged: false
    restartPolicy: [Always | Never | OnFailure]
    nodeSelector: object
    imagePullSecrets:
    - name: string
    hostNetwork: false
    volumes:
    - name: string
      emptyDir: {}
      hostPath:
        path: string
      secret:
        secretName: string
        items:
        - key: string
        path: string
      configMap:
        name: string
        items:
        - key: string
          path: string
```

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8SdOAZFdjAAEExapfTYw83.jpeg)

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8Se-APwRPAAFgT91o4rA78.jpeg)

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8Sg6AVMWTAAERiKNFNqg12.jpeg)

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8SjiALdhtAAE2uvHy2LQ50.jpeg)

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8SkeAGi7bAAEkXJ0hI9M46.jpeg)

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8SlOAWWScAAGNUCRmDmM67.jpeg)

## Pod的基本用法

Pod可用由1个或多个容器组合而成，例如名为`frontend`的`Pod`只由一个容器组成：

```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    name: frontend
spec:
  containers:
  - name: frontend
    image: kubeguide/guestbook-php-frontend
    env:
    - name: GET_HOSTS_FROM
      value: env
    ports:
    - containerPort: 80
```

这个Pod启动后，将启动1个Docker容器另一种场景是，当两个容器为紧耦合关系，应将两个容器打包为一个Pod，如下：

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu8eXGAIqJ3AABMykv1T7Q81.jpeg)

配置文件`frontend-localredis-pod.yaml`如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: redis-php
  labels:
    name: redis-php
spec:
  containers:
  - name: frontend
    image: kubeguide/guestbook-frontend:localredis
    ports:
    - containerPort: 80
  - name: redis
    image: kubeguide/redis-master
    ports:
    - containerPort: 6379
```

运行命令创建Pod：

```
kubectl create -f frontend-localredis-pod.yaml
kubectl get pods
NAME |Ready |status |Restarts |Age
redis-php |2/2 |Running |0 |10m
```

可以看到Ready信息为2/2，表示Pod中有两个容器在运行

查看这个Pod的详细信息，可以看到两个容器的定义及创建过程(Event事件信息)

```
#kubectl describe pod redis-php
```

##  静态Pod

静态Pod是由kubectl进行管理的仅存于特定Node上的Pod。他们不能通过API Server惊醒管理，无法与ReplicationController、Deployment或者DaemonSet进行关联，并且kubelet也无法对他们进行健康检查。静态Pod总是由kubectl进行创建，并且总是在kubelet所在的Node上运行。

创建Pod有两种方式：`配置文件`或`HTTP方式`，这里只说常用的配置文件方式

配置文件方式

在目录`/etc/kubelet.d`中放入`static-web.yaml`文件，内容：

```
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    name: static-web
  spce:
    containers:
    - name: static-web
      image: nginx
      ports:
      - name: web
        containerPort: 80
```

等一会，查看本机中已经启动的容器：

```
docker ps
```

就可以看到一个Nginx容器已经被Kubelet成功创建了出来。

到Master节点查看Pod列表，可以看到这个`static pod`：

```
kubectl get pods
```

由于静态Pod无法通过API Server直接管理，所以在Master节点尝试删除这个Pod，将会使其标为Pending状态，且不会被删除。

```
kubectl delete pod static-web-node1
```

删除该Pod的操作只能是到其所在Node上，将其自定义文件static-web.yaml从/etc/kubelet.d目录下删除

```
rm /etc/kubelet.d/static-web.yaml
docker ps
```

容器已经删除了

##  Pod容器共享Volume

在同一个Pod中多个容器能够共享Pod级别的存储卷Volume，如图：

![screenshot](http://img.orchome.com:8888/group1/M00/00/03/dr5oXFu9WO6AS8hfAABPkKvGy9040.jpeg)

在下面的例子中，Pod内包含两个容器：`tomcat`和`busybox`，在Pod级别设置Volume“app-logs”，用于tomcat向其中写日志文件，busybox读取日志文件

配置文件pod-volume-applogs.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: tomcat
    image: tomcat
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: app-logs
      mountPath: /usr/local/tomcat/logs
  - name: busybox
    image: busybox
    command: ["sh","-c","tail -f /logs/catalina*.log"]
    volumeMounts:
    - name: app-logs
      mountPath: /logs
  volumes:
  - name: app-logs
    emptyDir: {}
```

这里设置的Volume名为app-logs，类型为emptyDir，挂载到tomcat容器内`/usr/local/tomcat/logs`目录同时挂载到`logreader容器内的/logs目录`。

通过kubectl logs命令查看logreader容器的输出内容：

```
kubectl logs Volume-pod -c busybox
```

这个文件即为tomcat生成的日志文件/usr/local/tomcat/logs/catalina..log的内容，登录tomcat容器进行查看：

```
kubectl exec -it volume-pod -c tomcat -- ls /usr/local/tomcat/logs
kubectl exec -it volume-pod -c tomcat -- tail /usr/local/tomcat/logs/ctalima.2017-07-30.log
```

##  共享存储

​                     请参考:    http://orchome.com/1284



参考文献: http://orchome.com/1204#/collapse-1057









