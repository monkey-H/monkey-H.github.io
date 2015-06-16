---
layout: post
title: "coreos集群离线安装"
date: 2015-05-06 09:54:34 +0800
comments: true
categories: 
---

###写在前面
最近要搭建一个coreos集群，但是实验环境中没有外网，这样就不能使用官网的安装方法，整合了一些大牛的安装经验，加上我自己的安装实践，这里做个记录，希望大家可以少踩一些坑。

<!--more-->

####刻录安装盘
官网给的coreos的镜像是一个iso文件，我们需要把他刻成安装盘，中文的大白菜，比较好的UltroISO都可以，我一般都是用后者，不会用大白菜，但是据很多装系统的人描述，大白菜好像挺简单的。
我是在服务器上安装的，他自带一个iDRAC虚拟控制台，可以直接从iso安装，就省略了刻盘的过程。

####启动服务器
如果用的iso安装盘，直接从U盘启动，就可以进入到一个系统，如果从iDRAC虚拟控制台，从虚拟CD DVD ISO启动，就可以进入到一个预安装系统，这个系统是用来帮助你安装系统的，这个时候，系统并没有安装到你的服务器上。

####配置临时ip地址
由于我们没有外网，所以我们需要在本地局域网开一个tomcat服务器，从这个tomcat服务器里面下载我们需要的东西，那么为了可以获得这个tomcat服务器的东西，我们就需要给这台机器临时分配一个内网ip。
首先，我们先看看我们都有几个网卡。使用`ifconfig | less`命令。
![ifconfig](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/ifconfig_zps44mqiukv.png)
从上图中我们可以看到，我这台服务器上有两个网卡，我们就使用eno1来分配ip。
进入到/etc/systemd/network目录，分配内网ip。
```
cd /etc/systemd/network
sudo vi static.network
```
编辑static.network如下所示。

![staticnetwork](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/staticnetwork_zps6zii1usf.png)

注意，Name后面的eno1应该是上一步ifconfig出来的网卡的名称，Address后面的ip地址应该是你要分配的你的子网内的ip地址，网关也是对应的网关，DNS也是你自己的DNS解析器，当然，如果没有，可以不写。
接下来，使得网络生效。
```
sudo systemctl restart systemd-networkd
```
可以通过ping你的tomcat服务器来看看是否设置成功。
![tomcat](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/ping_zpshqq6duxh.png)

####配置tomcat服务器和上传安装脚本，配置文件
配置tomcat服务器的教材有很多，我们这里就不再描述了，可以随便google一个，然后配置好。我们这里要讲的就是coreos的安装脚本和配置文件。
[https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install](https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install)这个是官网给的安装脚本，首先我们要把他下载下来，然后，把他放在刚刚配置的tomcat路径下，tomcat/webapps/ROOT路径下。然后修改其中的一些内容。
![install](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/coreos-install_zpshx5fw1ft.png)
这里的ip要配置成你本机的ip，并且是可以被服务器连接到的ip。理解起来也很简单，就是把去网上要下载的东西，都从本机下载。那么很明显，我们也要把他要下载的东西，也预先下载下来，放在ROOT目录下。有两个东西，一个是镜像，也就是要安装的东西，一个是签名，是用来检测下载的镜像有没有破损之类的。（我们这里下载的是607版本的，现在应该有更新了）
镜像地址：[http://stable.release.core-os.net/amd64-usr/607.0.0/coreos_production_image.bin.bz2](http://stable.release.core-os.net/amd64-usr/607.0.0/coreos_production_image.bin.bz2)
签名地址：[http://stable.release.core-os.net/amd64-usr/607.0.0/coreos_production_image.bin.bz2.sig](http://stable.release.core-os.net/amd64-usr/607.0.0/coreos_production_image.bin.bz2.sig)
除了这个东西，还有一个配置文件，叫做cloud-config.yaml。这个文件是安装coreos系统必不可少的文件。我们这里默认要装的东西不多，所以cloud-config.yaml文件就简单了很多，如下。
```
#cloud-config
  
hostname: coreos1
  
coreos:    
  etcd:      
    peers: 114.212.189.136:7001
    addr: 114.212.189.136:4001  
    peer-addr: 114.212.189.136:7001  
  units:  
    - name: etcd.service  
      command: start  
    - name: fleet.service  
      command: start  
    - name: static.network  
      content: |  
        [Match]  
        Name=eno1
  
        [Network]  
        Address=114.212.189.136/24  
        Gateway=114.212.189.1
		DNS=202.119.32.6
users:    
  - name: core  
    ssh-authorized-keys:   
      - ssh-rsa #你的rsa key
  - groups:  
      - sudo  
      - docker 
```

```
#cloud-config
  
hostname: coreos2
  
coreos:    
  etcd:      
    peers: 114.212.189.136:7001
    addr: 114.212.189.143:4001  
    peer-addr: 114.212.189.143:7001  
  units:  
    - name: etcd.service  
      command: start  
    - name: fleet.service  
      command: start  
    - name: static.network  
      content: |  
        [Match]  
        Name=eno1
  
        [Network]  
        Address=114.212.189.143/24  
        Gateway=114.212.189.1
        DNS=202.119.32.6
users:    
  - name: core  
    ssh-authorized-keys:   
      - ssh-rsa #你的rsa key
  - groups:  
      - sudo  
      - docker 
```
这是两个服务器的cloud-config.yaml文件的比较，对比一下就很明显了。要修改的地方如下。
+ hostname 看你心情怎么命名，但是每个最好不一样。
+ peers 这个是etcd服务建立集群用来发现新机器的，除了第一台服务器安装的时候，是自己的ip，其余的都要使用集群中已经存在的某一个主机的ip。
+ addr 本机的ip和默认用来配对的4001端口
+ peer-addr 本机的ip和默认的用来配对的7001端口
+ Address 本机ip，前面的addr就是根据这个设置的，两个要一致。
+ Gateway 网关。你的局域网的网关
+ DNS 域名服务器，如果没有，可以不设置
+ ssh-rsa rsa密钥，启动服务器后，不能使用密码登陆，一般都是使用rsa密钥对来登陆，所以，这里要配置。至于rsa怎么产生，你可以在一个ubuntu系统上ssh-keygen命令参数，然后把~/.ssh/id_rsa.pub拷贝过来就是了。然后，你就可以从这个ubuntu系统登陆服务器，如果想从别的系统登陆，只要把那个系统的id_rsa.pub拷到服务器的authorized_keys里面就可以了。
+ 要注意，推荐你直接复制我拷上去的命令，因为cloud-config.yaml文件很特殊，他的语法很奇怪，比如DNS前面不是tab，而是8个空格等，如果这些你用的不对，就很容易出错。

####下载安装脚本和配置文件，并安装
准备就绪，可以安装了。在服务器上下载需要的安装脚本和配置文件。并给安装脚本赋予权限。
```
cd ~
wget 172.28.11.65:8080/coreos-install
chmod +x coreos-install
wget 172.28.11.65:8080/cloud-config143.yaml
```
注意这里的cloud-config143.yaml就是我们上一步提到的，不同的机器对应的不同的cloud-config文件。
安装
```
sudo ./coreos-install -d /dev/sda -C stable -c cloud-config143.yaml
```
![sucess](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/success_zps7auwgjzn.png)
出现这个界面就说明成功了，然后重启就可以了。
```
sudo reboot
```
重启好了之后，你可以从你刚刚那个ubuntu系统ssh进去了。
```
ssh core@114.212.189.143
```
![login](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/login_zpssxrdse5c.png)
同样的，使用基本相同的步骤，不同的cloud-config.yaml文件，配置其余的机器，配置完成后，在其中的一台机器上，查看集群中有哪些机器。
```
fleetctl list-machines
```
![](http://i1066.photobucket.com/albums/u407/5681713/coreos-install/listmachines_zpskqtipnkp.png)
