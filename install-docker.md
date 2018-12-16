# Centos7上安装docker

[![img](https://s3.51cto.com//wyfs02/M01/5A/23/wKioL1T31F-iR_MmAAA3CbP0FkY778_middle.jpg)](http://blog.51cto.com/qiangsh)

qianghong000

0人评论



1383人阅读

2018-08-31 11:52:48



Docker从1.13版本之后采用时间线的方式作为版本号，分为社区版CE和企业版EE。

Docker CE 即社区免费版，Docker EE 即企业版，强调安全，但需付费使用。企业版会提供额外的收费服务，比如经过官方测试认证过的基础设施、容器、插件等。

社区版按照stable和edge两种方式发布，每个季度更新stable版本，如17.06，17.09；每个月份更新edge版本，如17.09，17.10。

# 前提条件

> 目前，CentOS 仅发行版本中的内核支持 Docker。
>
> Docker 运行在 CentOS 7 上，要求系统为64位、系统内核版本为 3.10 以上。
>
> Docker 运行在 CentOS-6.5 或更高的版本的 CentOS 上，要求系统为64位、系统内核版本为 2.6.32-431 或者更高版本。

# 安装docker

1、Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker 。
通过 uname -r 命令查看你当前的内核版本

```
[root@localhost ~]# uname -r
3.10.0-862.11.6.el7.x86_64
```

2、使用 root 权限登录 Centos。确保 yum 包更新到最新。

```
[root@localhost ~]# sudo yum update
```

3、卸载旧版本(如果安装过旧版本的话)

```
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

4、安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的
`[root@localhost ~]# sudo yum install -y yum-utils device-mapper-persistent-data lvm2`

5、设置yum源
`sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo`

6、更新 yum 缓存：
`sudo yum makecache fast`

7、可以查看所有仓库中所有docker版本，并选择特定版本安装

```
[root@bogon ~]# yum list docker-ce --showduplicates | sort -r
已加载插件：fastestmirror
可安装的软件包
 * updates: mirrors.tuna.tsinghua.edu.cn
Loading mirror speeds from cached hostfile
 * extras: mirrors.tuna.tsinghua.edu.cn
docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.0.ce-1.el7.centos             docker-ce-stable
 * base: mirrors.tuna.tsinghua.edu.cn
```

8、安装docker

```
#由于repo中默认只开启stable仓库，故这里安装的是最新稳定版18.06.1
[root@bogon ~]# sudo yum install docker-ce

# 指定版本安装
[root@bogon ~]# yum install https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
[root@bogon ~]# sudo yum install docker-ce-17.03.2.ce
```

9、启动并加入开机启动

```
[root@bogon ~]# sudo systemctl start docker
[root@bogon ~]# sudo systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

10、验证安装是否成功(有client和service两部分表示docker安装启动都成功了)

```
[root@bogon ~]# docker version
```

# 镜像加速

鉴于国内网络问题，后续拉取 Docker 镜像十分缓慢，我们可以需要配置加速器来解决。
在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）

```
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

注意，一定要保证该文件符合 json 规范，否则 Docker 将不能启动。
之后重新启动服务。

# 删除 Docker CE

如果卸载 Docker CE，执行以下命令：

```
[root@bogon ~]# sudo yum remove docker-ce
[root@bogon ~]# sudo rm -rf /var/lib/docker
```

# 更换 docker 安装源

由于 docker 官方服务器位于国外，下载速度可能异常缓慢，建议将官方源下载地址替换为清华大学镜像源后进行安装。

```
#替换docker官方源下载地址
sudo sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo

#清除缓存
sudo yum clean all

#重新建立缓存
sudo yum makecache

#安装docker-ce
sudo yum install docker-ce
```

 