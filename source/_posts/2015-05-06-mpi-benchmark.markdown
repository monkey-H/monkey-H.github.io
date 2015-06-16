---
layout: post
title: "使用mtt作为mpi的benchmark 测试docker container计算集群性能"
date: 2015-05-06 16:31:43 +0800
comments: true
categories: 
---

###写在前面
docker技术已经十分普遍，利用docker来创建多个虚拟机，实现分布式运算也是一种十分便利的方式，但是docker创建的虚拟机和常规的虚拟机，或者和分布式的集群到底有多少性能的差距，docker能不能通过创建多个container的方式实现常规的分布式运算，我们这里使用openmpi的一个benchmark，mtt来做一个验证。
<!--more-->

###为什么这么做
早些时候老板让做了一个open mpi的image，并在单机环境下，成功使用docker搭建了一个openmpi的集群，可以跑一些hello world的例子。后来，在ubuntu环境下，使用openvswitch搭建了一个多host的集群，在coreos环境下，使用etcd键值对服务和搭建了一个集群，并分别通过tunnel的方式，和flannel的方式，修改docker -d的参数，使得，在不同的host主机上，可以运行在同一个子网内的container，并且相互之间可以通信。参见这篇[blog](http://monkey-h.github.io/blog/shi-yong-vagrantda-jian-coreosji-qun-an-zhuang-flannelfu-wu-bing-shi-yong-mpi-benchmarkce-shi/)。
前几天老板说，你成功搭建了，跑hello world的例子也成功了，但是你这个性能怎么样，多host情况下，你是通过tunnel桥，或者flannel实现的，那么你这样的性能和单host肯定是不一样的，性能差多少呢？你要找个benchmark来检验一下，所以，我就找了个openmpi的mtt来测试一下，这里做个记录。
我们先来说说mtt是个什么东西，mtt是一个检验mpi环境的，不只是openmpi，包括别的版本的mpi也是可以的。第一，mtt会检验你是否正确安装了mpi，甚至mtt可以帮助你安装mpi。第二，mtt会检验你写的代码是否可以通过安装的mpi成功编译。第三，mtt会检验你写的程序是否可以正确在集群环境中运行，并返回集群中云心的结果，包括是否成功啊，花费的时间等。第三个特点，也正是我们需要的功能。
首先要明确的一个观点就是，肯定要搭建一个nfs，或者说，也可以使用别的方法，比如开个docker container作为data container，然后通过这种方式同步也可以，如果单节点当然可以，但是在多节点的时候就不可以了，因为data container不能跨节点共享数据。我们为什么要在不同节点之间共享数据呢？想象一下，我们在运行mtt的时候，他会运行我们的测试程序，那么这个测试程序，理论上只有在我们运行mtt的host才会有，那么怎么让那些slave节点也去运行这个程序呢？mtt并不帮助我们把测试程序传到slave节点上去，需要我们自己拷贝，或者通过别的简单方式，别如我们刚刚说的通过data container的方式，或者nfs。由于我们需要在不同的host节点上创建slave，所以我们这里肯定要搭建nfs服务。否则在运行mtt的时候，会报cannot find execute path file之类的错误。

###搭建nfs
首先，在主host上安装nfs服务。
```
sudo apt-get install nfs-kernel-server
sudo apt-get install nfs-common
```
注意一定要安装nfs-common服务，有的文章指出不需要安装，因为在安装nfs-kernel-server的时候，会自动安装nfs-common和nfs-portmap，我尝试后发现，portmap是自动安装了，但是，并没有安装nfs-common，会报
```
mount wrong nfs type, bad option, bad superblock on XXX
```
的错误。
然后，配置nfs共享目录。假如我们需要共享的目录是/home/monkey/nfs文件夹下的所有文档，那么我们就需要配置/etc/exports文件，在最后一行，加上这么一句。
```
"/home/monkey/nfs" *(rw,sync,no_subtree_check)
```
至于这句话什么意思，前面的路径肯定就是我们要共享的nfs文件，后面的的意思是，所有ip段的机器都可以共享这个路径，如果你要指定某个机器，可以直接把替换成那台机器的ip，比如114.212.87.52，或者你要指定某个子网段的机器，可以这样写，114.212.0.0/24，这种东西查查nfs配置就知道了，我们这里不作为重点。后面的rw是读写权限，ro是read only，sync是同步等，自己查查文档。
配置好后，重启服务，
```
sudo /etc/init.d/portmap resatrt
sudo /etc/init.d/nfs-kernel-server restart
```
就可以了。我们可以输入showmount -e来进行查看。
![mount](http://i1066.photobucket.com/albums/u407/5681713/coreos/mount_zpsoynx36iw.png)
ok，这样nfs就配置完成了。

###docker配置多节点mpi cluster
这一步就比较简单了，因为我们之前已经做好了open mpi的image，在docker hub上也有，如果有同学不想自己做的话，可以自己去下载，名字是hmonkey/openmpi14.04:v3。启动的时候，一定要注意给container超级权限。否则会出现挂载nfs共享文件夹的时候报错的问题。
```
cannot mount on ip readonly
```
类似于这样的错误。
```
sudo docker run -ti --name master --privileged=true hmonkey/openmpi14.04:v3 /bin/bash
sudo docker run -ti --name slave1 --privileged=true hmonkey/openmpi14.04:v3 /bin/bash
sudo docker run -ti --name slave2 --privileged=true hmonkey/openmpi14.04:v3 /bin/bash
```
这样就起来了三个节点，其中我们把其中的一个座位master，另外的两个座位slave。起来之后，要做两件事，第一，我忘记在image里面自动启动ssh服务，所以，起来之后，要首先把ssh服务启动起来，分别运行sudo service ssh start其次，要在master里面把slave的ip加到host列表里面去，这样才可以方便的访问。在master上vim /etc/hosts，把slave1和slave2的地址加上去。
![host](http://i1066.photobucket.com/albums/u407/5681713/coreos/host_zpslj6kzrht.png)
这样，就配置好集群环境了。由于我们这里已经设置好ssh进去的时候不需要输入密码，这一步就可以省略了。关于如何配置不需要密码就可以直接ssh进去，请查看我别的[blog](http://monkey-h.github.io/blog/ssh-image/)。
```
cd /
sudo mkdir nfs
sudo mount ip:/home/monkey/nfs /nfs
```
这样就可以了，这样就把宿主机的nfs文件夹，和当前container的nfs文件夹同步起来了，在三个container上都要这么做。注意这里的ip就是我们宿主机的ip，也就是我们搭建的nfs服务的那个节点的ip。

###下载并运行mtt
在master容器内，
```
cd /
git clone https://github.com/open-mpi/mtt
cd mtt/sample
cat developer.ini trivial.ini | ../client/mtt - hostlist=slave1,slave2 alreadyinstalled_dir=/usr --scratch=/nfs
```
最后一句命令比较难以理解。其中，developer.ini里面是检测你是否正确安装了mpi，trivial.ini是跑测试的程序的。../client/mtt是mtt的运行脚本，hostlist是要检验的集群中的机器，alreadyinstall_dir是你把mpi安装在哪里了，scratch是最重要的，是要写配置的nfs共享目录在container里的位置。
当运行结束时，如果出现这样的画面，就说明，mpi环境运行通过，并且，有数据显示，可以和常规的集群，或者虚拟机进行比较。
![1](http://i1066.photobucket.com/albums/u407/5681713/coreos/1_zpsguiwjprd.png)
![2](http://i1066.photobucket.com/albums/u407/5681713/coreos/2_zps3ucpl1xn.png)
最后提醒一句，mtt的代码已经迁移到github上了，如果你使用
```
svn checkout https://svn.open-mpi.org/svn/mtt/branches/ompi-core-testers
```
这样的命令去下载mtt，那么他会一直让你输入username 和 passwd。