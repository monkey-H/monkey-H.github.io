---
layout: post
title: "使用vagrant搭建coreos集群安装flannel服务并使用mpi测试"
date: 2015-05-06 15:20:00 +0800
comments: true
categories: 
---

###写在前面
容器技术已经成为了当前技术的潮流，学习容器技术已经刻不容缓。除了原始的warden容器，docker容器之外，coreos也在发展自己的容器技术，rocket，就连之前不看好容器技术的microsoft也开始创建自己的容器技术，nano server。我们这里尝试搭建coreos集群，并安装了flannel服务，用来实现mpi程序的测试。

<!--more-->

####需要的软件支持
我们采用的是使用vagrant + virtualbox的方式来搭建coreos集群，所以，这两个软件的正确安装是必不可少的。

####coreos集群安装
在之前的blog [coreos离线安装](http://monkey-h.github.io/blog/coreoschi-xian-an-zhuang/) 里面，我们已经在服务器上安装了coreos集群，但是并不是所有的同学都有这么多资源，可以有那么多的服务器可以使用。通过搭建虚拟机的方式又很慢，所以，我们这里采用vagrant + virtualbox的方式来搭建集群。
使用vagrant搭建coreos集群官网，官网上有coreos集群通过vagrant搭建的具体步骤，基本没有什么坑，我们这里简单说明一下。
首先，在宿主机目录下，下载coreos-vagrant的git代码，并进入代码目录。
```
git clone https://github.com/coreos/coreos-vagrant.git
cd coreos-vagrant
```
接着，我们就可以修改其中的某些参数，来搭建我们的集群了。
首先，把示例代码拷贝一下，我们这里直接重命名了。
```
mv user-data.sample user-data
mv config.ruby.sample config.ruby
```
接着，修改两个文件的参数。打开config.ruby，把里面的
```
#$num_instance=1
```
改成
```
$num_instance=3
```
因为我们搭建的集群如果只有一台机器，是没有意义的。
还有把
```
#$update_channel='alpha'
```
改成
```
$update_channel='stable'
```
我们还是使用比较稳定的版本较好，免得出现乱起八糟的问题。
打开user-data，把里面的
```
#discovery: https://discovery.etcd.io/<token>
```
改成
```
discovery: https://discovery.etcd.io/<token>
```
至于是怎么来的，你可以访问这个网址，这个网址可以自动生成一个唯一的，用来达到建立集群的目的。至于这个东西有什么用，可以去查etcd的文档，简单来说，就是每个启动的机器都会去找这个token，找到了，就连接到这个集群中，从而实现集群的建立。
打开Vagrantfile，修改
```
config.vm.box_version ">= 308.0.1"
```
为
```
#config.vm.box_version ">= 308.0.1"
```
就是把他删掉，因为后面可能会报这个错误，这个又不是必须的，我们就直接删掉。
好了，该修改的都修改了，这个时候，就可以大胆的启动了。
```
vagrant up
```

####flannel搭建
vagrant搭建完成后，我们就可以进入搭建好的coreos主机了。
```
vagrant ssh core-01 -- -A
```
分别进入三个主机，三个主机的名字分别为core-01，core-02，core-03。现在假如我们进入了core-01。我们可以在里面看到集群中所有的机器。
```
fleetctl list-machines
```
![list](http://i1066.photobucket.com/albums/u407/5681713/coreos/list_zpshwqj6n0d.png)
可以看到，集群中一共有三台机器，ip也都是知道的。接着，让我们来安装flannel服务。在每台机器里，都这么做。
```
cd /
git clone https://github.com/coreos/flannel.git
mv /flannel /opt
cd opt
sudo docker run -v /opt:/opt/flannel -ti google/golang /bin/bash -c "cd /opt/flannel && ./build"
```
然后，采用etcd的键值对服务，设置全局子网域。
```
etcdctl rm /coreos.com/network --recursive
etcdctl mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'
```
然后启动flannel服务。注意，我们在启动集群的时候，eth0网卡是通过nat方式和host主机连在一起的，但是，eth1是和别的host主机在同一个局域网的网卡，所以我们在启动flannel服务的时候，要制定eth1网卡。
```
sudo /opt/bin/flanneld -iface=eth1 &
```
flannel启动。在core-02和core-03上运行相同的过程。但是我们的docker并没有使用指定的子网，所以，我们还要重新运行docker，让他使用我们指定的子网。
```
source /run/flannel/subnet.env
sudo rm /var/run/docker.pid
sudo ifconfig docker0 ${FLANNEL_SUBNET}
sudo docker -d --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```
大功告成，检验一下，`ifconfig`
![](http://i1066.photobucket.com/albums/u407/5681713/coreos/flannel_zpspjmjfrk6.png)
这里可以看到，我们的flannel0已经建成。同时
![](http://i1066.photobucket.com/albums/u407/5681713/coreos/docker_zpsqy85ijfh.png)
docker0也已经是我们指定的ip了。

####mpi应用测试
我在dockerhub上上传了一个关于mpi的镜像，我们这里可以下载下来运行一下。
在core-01上，运行
```
sudo docker run -ti --name master hmonkey/openmpi14.04:v3 /bin/bash
```
在core-02上运行
```
sudo docker run -ti --name slave1 hmonkey/openmpi14.04:v3 /bin/bash
```
在core-03上运行
```
sudo docker run -ti --name slave2 hmonkey/openmpi14.04:v3 /bin/bash
```
然后在core-01上，
```
cd /mpi`
touch hostfile
vi hostfile //在里面添加slave1和slave2的ip地址。
mpirun -n 3 -hostfile hostfile mpi_hello
```
大功告成，结果如下
![](http://i1066.photobucket.com/albums/u407/5681713/coreos/mpi_zps1vidqse2.png)
要注意的是，我的mpi镜像忘记默认启动ssh服务了。如果运行mpi应用的时候包connect refused，把ssh服务打开就可以了
