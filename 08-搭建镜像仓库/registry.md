# docker：用registry快速搭建私有镜像仓库

[![img](https://s2.51cto.com/oss/201712/07/760d496cb53865af6268debd8f2c0fe8.jpg?x-oss-process=image/resize,m_fixed,h_120,w_120)](http://blog.51cto.com/ganbing)

甘兵

关注

4人评论



21643人阅读

2018-03-02 18:20:38



## 1、背景

> 在 Docker 中，当我们执行 docker pull xxx 的时候，可能会比较好奇，docker 会去哪儿查找并下载镜像呢？

 它实际上是从 registry.hub.docker.com 这个地址去查找，这就是Docker公司为我们提供的公共仓库，上面的镜像，大家都可以看到，也可以使用。所以，我们也可以带上仓库地址去拉取镜像，如：docker pull registry.hub.docker.com/library/alpine，不过要注意，这种方式下载的镜像的默认名称就会长一些。
 如果要在公司中使用 Docker，我们基本不可能把商业项目上传到公共仓库中，那如果要多个机器共享，又能怎么办呢？

正因为这种需要，所以私有仓库也就有用武之地了。

 所谓**私有仓库**，也就是在本地（局域网）搭建的一个类似公共仓库的东西，搭建好之后，我们可以将镜像提交到私有仓库中。这样我们既能使用 Docker 来运行我们的项目镜像，也避免了商业项目暴露出去的风险。

 下面我们用官方提供的registry镜像来搭建私有镜像仓库，当然还有其它很多方法。

## 2、环境

准备两台安装好docker的服务器：
服务端机器 （**主机名为registry**）：docker私有仓库服务器，运行registry容器；
测试端机器 （**主机名为node**）：普通的docker服务器，在这台服务器上下载一个测试镜像busybox，然后上传到registry服务器进行测试；

## 3、部署(服务端操作)

**3.1 下载镜像registry**

```
[root@registry ~]# docker pull registry
Using default tag: latest
latest: Pulling from library/registry
81033e7c1d6a: Pull complete 
b235084c2315: Pull complete 
c692f3a6894b: Pull complete 
ba2177f3a70e: Pull complete 
a8d793620947: Pull complete 
Digest: sha256:672d519d7fd7bbc7a448d17956ebeefe225d5eb27509d8dc5ce67ecb4a0bce54
Status: Downloaded newer image for registry:latest
```

**3.2 查看镜下是否pull下来了**
![docker：用registry快速搭建私有镜像仓库](http://i2.51cto.com/images/blog/201803/02/5ca85f3fed4b4445dab9ba46700620b3.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

**3.3 运行registry容器**

```
[root@registry ~]# docker run -itd -v /data/registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry:latest 
06a972de6218b1f1c3bf9b53eb9068dc66d147d14e18a89ab51db13e339d3dc9
```

参数说明
-itd：在容器中打开一个伪终端进行交互操作，并在后台运行；
-v：把宿主机的/data/registry目录绑定 到 容器/var/lib/registry目录(这个目录是registry容器中存放镜像文件的目录)，来实现数据的持久化；
-p：映射端口；访问宿主机的5000端口就访问到registry容器的服务了；
--restart=always：这是重启的策略，假如这个容器异常退出会自动重启容器；
--name registry：创建容器命名为registry，你可以随便命名；
registry:latest：这个是刚才pull下来的镜像；

**3.4 测试镜像仓库中所有的镜像**

```
[root@registry ~]# curl http://127.0.0.1:5000/v2/_catalog
{"repositories":[]}
```

现在是空的，因为才刚运行，里面没有任何镜像内容。

## 4、测试镜像仓库（测试端操作）

**4.1 修改下镜像源并重启docker服务**

```
[root@node ~]# vim /etc/docker/daemon.json
{
  "registry-mirrors": [ "https://registry.docker-cn.com"]
}

[root@node ~]# systemctl restart docker
```

**4.1 下载busybox镜像**

```
[root@node ~]# docker pull busybox
[root@node ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              f6e427c148a7        36 hours ago        1.15MB
```

**4.2 为镜像打标签**

```
[root@node ~]# docker tag busybox:latest  172.18.18.90:5000/busybox:v1
```

> 格式说明：Usage: docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

busybox:lastest 这是源镜像，也是刚才pull下来的镜像文件；
172.18.18.90:500/busybox:v1：这是目标镜像，也是registry私有镜像服务器的IP地址和端口；

查看一下打好的tag：
![docker：用registry快速搭建私有镜像仓库](http://i2.51cto.com/images/blog/201803/02/ce7e31defff5bc7c68c66edda0ee0760.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

**4.3 上传到镜像服务器**

```
[root@node ~]# docker push 172.18.18.90:5000/busybox:v1 
The push refers to repository [172.18.18.90:5000/busybox]
Get https://172.18.18.90:5000/v2/: http: server gave HTTP response to HTTPS client
```

注意了，这是报错了，需要https的方法才能上传，我们可以修改下daemon.json来解决：

```
[root@node ~]# vim /etc/docker/daemon.json 
{
  "registry-mirrors": [ "https://registry.docker-cn.com"],
  "insecure-registries": [ "172.18.18.90:5000"]
}
```

添加私有镜像服务器的地址，注意书写格式为json，有严格的书写要求，然后重启docker服务：

```
[root@node ~]# systemctl  restart docker
```

在次上传可以看到没问题 了：

```
[root@node ~]# docker push 172.18.18.90:5000/busybox:v1 
The push refers to repository [172.18.18.90:5000/busybox]
c5183829c43c: Pushed 
v1: digest: sha256:c7b0a24019b0e6eda714ec0fa137ad42bc44a754d9cea17d14fba3a80ccc1ee4 size: 527
```

**4.4 测试下载镜像**
上传测试没问题了，我们接下来测试一下从registry服务器上下载刚才上传的busybox镜像，先删除node主机上的镜像：

```
[root@node ~]# docker rmi -f $(docker images -aq)
Untagged: 172.18.18.90:5000/busybox:v1
Untagged: 172.18.18.90:5000/busybox@sha256:c7b0a24019b0e6eda714ec0fa137ad42bc44a754d9cea17d14fba3a80ccc1ee4
Untagged: busybox:latest
Untagged: busybox@sha256:2107a35b58593c58ec5f4e8f2c4a70d195321078aebfadfbfb223a2ff4a4ed21
Deleted: sha256:f6e427c148a766d2d6c117d67359a0aa7d133b5bc05830a7ff6e8b64ff6b1d1d
Deleted: sha256:c5183829c43c4698634093dc38f9bee26d1b931dedeba71dbee984f42fe1270d

查看一下node主机上的镜像全部删除了：
[root@node ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

然后，从registry服务器上下载busybox镜像：

```
[root@node ~]# docker pull 172.18.18.90:5000/busybox:v1
v1: Pulling from busybox
d070b8ef96fc: Pull complete 
Digest: sha256:c7b0a24019b0e6eda714ec0fa137ad42bc44a754d9cea17d14fba3a80ccc1ee4
Status: Downloaded newer image for 172.18.18.90:5000/busybox:v1
[root@node ~]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
172.18.18.90:5000/busybox   v1                  f6e427c148a7        36 hours ago        1.15MB
```

**列出所有镜像：**

```
[root@node ~]# curl  http://172.18.18.90:5000/v2/_catalog
{"repositories":["busybox"]}
```

**列出busybox镜像有哪些tag：**

```
[root@node ~]# curl  http://172.18.18.90:5000/v2/busybox/tags/list
{"name":"busybox","tags":["v1"]}
```