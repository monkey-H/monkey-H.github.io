---
layout: post
title: "dokku源码解读四"
date: 2015-05-05 19:50:07 +0800
comments: true
categories: 
---

###写在前面
dokku，号称只用了100行左右代码就实现的简单paas平台，号称是你能见到的最小的paas平台，确实，很NB。整个dokku的实现大部分采用了bash脚本命令，只有少数的go语言文件，我们接下来就通过几篇blog，来看看，dokku到底是怎么实现的这个pass平台。

<!--more-->

###dokku服务器端操作
在dokku搭建的dokku平台上，有如下一些基本命令。
![](http://i1066.photobucket.com/albums/u407/5681713/octopress/help_zpsawheqycp.png)
在前面我们已经介绍过了，如果在dokku源码文件里找不到对应的case命令，就会去plugins/\**/command里面去找。对这些命令而言，在这些目录下都是可以找到的，而且代码的可读性很强，我们这里就不再一个一个介绍，找几个比较有意思的，常用的命令介绍一下。

+ apps:create <app>
![](http://i1066.photobucket.com/albums/u407/5681713/octopress/create_zpsuwx8dj6q.png)
第一个判断语句是在用户没有指定创造的应用的名字的时候的提示语句。
第二个判断语句是在已经存在指定名字的应用的时候的提示语句。
创造的过程就是在$DOKKU_ROOT路径下建造了一个$APP的文件夹，就创建完成了，是不是和我们想象中的不一样？也就是说，dokku的apps:create并不是创建了一个应用，而只是创建了一个文件夹而已，如果你想创建应用容器，还是需要用docker run来做。
+ apps:destroy <app>
![](http://i1066.photobucket.com/albums/u407/5681713/octopress/destroy_zpsqccw8iun.png)
和create不同的是，删除就是真正的删除了，除了我最后圈出来的那两个停止容器，和删除容器的操作之外，前面还有很多准备工作，比如，用箭头指出来的，和[deis](deis.com)很像的一个防止用户误删的判断操作，和支持强制删除等操作。值得注意的是，在最后，dokku还删除了对应的image，用来由于部署的应用过多，从而空间不够的情况，当然了，这种情况可能会导致上传之前已经部署过的一个应用，需要重新制作镜像的情况，可能会花费的时间多一些，但是，瑕不掩瑜。
+ backup:export [file]
![](http://i1066.photobucket.com/albums/u407/5681713/octopress/backup_zps9ctf2nbl.png)
这里我们要注意到一件事情，就是这里实际是调用了`pluginhook backup-export 1 $BACKUP_DIR`，要注意的是，我们从整个源码的目录结构里面可以看到，有很多backup-export文件，pluginhook的一个机制是，当有很多相同名字的插件的时候，他会把所有的插件都执行一遍，这样才是dokku的需求，可以把很多需要备份的东西，在不同的地方写上，而不会产生冲突，这也是让dokku发展pluginhook的一个很大的需求点，要不然，dokku很有可能会直接采用hook机制实现这些东西。
+ ls
![](http://i1066.photobucket.com/albums/u407/5681713/octopress/ls_zpszlg1xjln.png)
可以看到，dokku查找所有应用容器的方式相当简单，因为每次创建一个容器，他都会在$DOKKU_ROOT目录下创建一个对应的文件夹，所以，只要读取这个文件夹就可以了，没有用到docker的那一套，简单多了。
+ run
![](http://i1066.photobucket.com/albums/u407/5681713/octopress/run_zpsi0vbhyap.png)
这个run我们可以看到，应该是在create之后做的事情，这个run的前提是$DOKKU_ROOT目录下，已经存在了需要run的应用的目录，包括里面应有的文件。