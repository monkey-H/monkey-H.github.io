---
layout: post
title: "在coreos系统下，同时添加flannel和私有仓库的参数"
date: 2015-05-06 17:07:40 +0800
comments: true
categories: 
---

###写在前面
前面我们已经搭建好了一个coreos集群 [看这里](http://monkey-h.github.io/blog/coreoschi-xian-an-zhuang/)，正如这篇blog里面说的一样，我们没有外网，所以我们自己搭建了一个私有仓库，方法可以去官网找，很简答的两行命令，这里就不再叙述。既然搭建了私服，我们就希望我们的这个集群使用这个集群。其次，我们搭建了一个集群，我们也希望屏蔽这是一个集群的概念，让用户用起来就像在使用一个主机的服务器一样，所以我们需要[flanel](github.com/coreos/flannel)服务。这就涉及到给docker daemon添加多个参数，我们这里记录一下过程。同时，这也是如果在coreos集群中如何运行service的介绍。
<!--more-->

####添加私服参数
我们的服务器是没有外网的，所以，从docker hub上下载一个镜像都是问题，所以我们搭建了一个私有的docker registry。关于docker registry的东西，其实也是很有意思的，如果以后有机会，我们还会写一篇文章来介绍docker registry。我们现在的方式是，自己建立一个私服，然后从可以连接外网的地方下载一些image，然后上传到私服，这样，就可以直接从我们的私服上下载了，而且速度相当快。因为是局域网内，而且没有经过国防网。
系统默认的docker.service在
```
/usr/lib64/systemd/system/docker.service
```
里面，但是这个路径下的文件是可读不可写的，所以我们要把他移动到一个可读可写的地方。不只是docker.service，只要是unit files都要在这个路径
```
/etc/systemd/system
```
所以，第一步，拷贝一份docker.service。并编辑。
```
sudo cp /usr/lib64/systemd/system/docker.service /etc/systemd/system/
cd /etc/systemd/system
sudo vim docker.service
```
把其中的这句话
```
ExecStart=/usr/lib/coreos/dockerd --daemon --host=fd:// $DOCKER_OPTS -- $DOCKER_OPT_BIP $DOCKER_OPT_MTU $DOCKER_OPT_IPMASQ
```
改成
```
ExecStart=/usr/lib/coreos/dockerd --daemon --host=fd:// $DOCKER_OPTS --insecure-registry docker.iwanna.xyz:5000 $DOCKER_OPT_BIP $DOCKER_OPT_MTU $DOCKER_OPT_IPMASQ
```
加的那个参数，就是告诉本机，这个地址的私有仓库可以不走https协议，要不然在pull image的时候会报错。
![insecure](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/insecure_zpsjexwxehm.png)
接下来，我们要让这个service生效，然后启动。
```
sudo systemctl stop docker.service
sudo systemctl enable /etc/systemd/system/docker.service
sudo systemctl daemon-reload
sudo systemctl start docker.service
```
接下来就可以pull私有仓库里可以编译flannel的image了。在docker hub里面，叫做google/golang是一个支持go语言的image，我在上传到私有仓库的时候，为了名字好记，重新命名了，实际上就是这个image。
![pull](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/pull_zpsihbxjfxl.png)


####安装flannel
现在我们已经从私服上下载下来了编译flannel的image，现在我们需要flannel的源码，由于我们这里不能使用外网，所以，还是通过tomcat的方式下载，如果可以使用外网，可以直接使用git clone来下载
```
git clone https://github.com/coreos/flannel.git
```
我们这里是从tomcat服务器上下载的。
![wget](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/wget_zpswmf12hii.png)
接下来，解压，移动到/opt目录，为什么这么做，是为了防止以后误删。
```
tar -zxf flannel.tar.gz
sudo mkdir /opt
sudo mv flannel/* /opt
cd /opt
```
好了，现在可以编译了。
```
docker run -v /opt:/opt/flannel -ti docker.iwanna.xyz:5000/hmonkey/flannel /bin/bash -c "cd /opt/flannel && ./build"
```
![building](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/building_zps5vlcndnc.png)
接下来，我们就要创建flannel服务了。
进入到我们的unit files目录。
```
cd /etc/systemd/system
sudo touch flannel.service
sudo vim flannel.service
```
在里面写入这样的代码。
```
[Unit]
Description=flannel
Requires=etcd.service
After=etcd.service

[Service]
ExecStartPre=-/usr/bin/etcdctl mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'
ExecStart=/opt/bin/flanneld

[Install]
WantedBy=multi-user.target
```
然后，启动这个服务。
```
sudo systemctl enable flannel.service
sudo systemctl start flannel.service
```
可以查看状态，如果是这样，就说明成功了。
![status](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/status_zpsk02mrnhs.png)
接下来，停掉docker.service，删除网桥docker0，让docker0使用flannel的网桥，然后修改docker.service，重启docker。
```
source /run/flannel/subnet.env
sudo systemctl stop docker.service
sudo ifconfig docker0 ${FLANNEL_SUBNET}
sudo vim docker.service
sudo systemctl daemon-reload
sudo systemctl start docker.service
```
编辑docker.service的时候，使得最后，docker.service看起来是这个样子的。
```
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=docker.socket early-docker.target network.target
Requires=docker.socket early-docker.target

[Service]
Environment=TMPDIR=/var/tmp
EnvironmentFile=-/run/flannel/subnet.env
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
ExecStart=/usr/lib/coreos/dockerd --insecure-registry docker.iwanna.xyz:5000 --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU} --daemon --host=fd://

[Install]
WantedBy=multi-user.target
```

好了，大功告成，验证一下。
我们在coreos3上随便跑一个container，可以看到，他的ip是我们之前用flannel获得的。
![try](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/try_zpsdxnqxstq.png)
然后我们在coreos1上，ping这个ip，可以看到，完全可以ping通。
![ping](http://i1066.photobucket.com/albums/u407/5681713/flannel%20regitstry/ping_zpsfvnfwq69.png)
同样的，简单学习一下systemd的语法，就可以让我们开启不同的service，如果这些service中有涉及到有可能修改docker daemon的参数的，就可以通过上面这个例子进行修改。