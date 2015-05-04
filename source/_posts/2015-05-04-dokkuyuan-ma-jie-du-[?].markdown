---
layout: post
title: "dokku源码解读二"
date: 2015-05-04 17:03:52 +0800
comments: true
categories: 
---

###写在前面
	dokku，号称只用了100行左右代码就实现的简单paas平台，号称是你能见到的最小的paas平台，确实，很NB。整个dokku的实现大部分采用了bash脚本命令，只有少数的go语言文件，我们接下来就通过几篇blog，来看看，dokku到底是怎么实现的这个pass平台。

###dokku进入点
我们都知道git和ssh密不可分，从前面的知识我们也知道，当git push的时候，会首先执行ssh git@git.com git-receive-pack mygit.git这个操作，我们也说过了，这个会是dokku的入口。
我们停止服务器端的ssh服务，重新使用debug模式开启。
`sudo service ssh stop`
`sudo /usr/sbin/sshd -d`
然后在客户端重新部署一个服务，可以看到这样的提示。
![sshd](/Users/monkey/Documents/dokku/codereading/photo/sshd.png)
我们去看看，/home/dokku/.sshcommand做了什么。
`cat /home/dokku/.sshcommand`	
输出
![catsshcommand](/Users/monkey/Documents/dokku/codereading/photo/catsshcommand.png)
这也就说明，不管你原来的命令是什么（$SSH_ORIGINAL_COMMAND），dokku都在这个命令前面增加了一个/usr/local/bin/dokku，也就是修改了客户端ssh进去的进入点，从而实现了进入pass平台部署任务的功能。
那么这个.sshcommand是怎么来的呢？我们去查看我们的安装文档，我这里是使用的vagrant安装的，我们可以去查看Vagrantfile文件。
![vagrantfile](/Users/monkey/Documents/dokku/codereading/photo/vagrantfile.png)
在这里面，我们可以看到，这里只要是用的makefile安装的，我们接着查看Makefile。
![sshcommand](/Users/monkey/Documents/dokku/codereading/photo/sshcommand.png)
这里可以看到，下载了一个脚本文件，然后执行了
`sshcommand create dokku /usr/local/bin/dokku`
命令，我们可以从makefile中找到SSHCOMMAND_URL的网址
`SSHCOMMAND_URL ?= https://raw.github.com/progrium/sshcommand/master/sshcommand`
进入到这个网址，我们可以看到
![sshcommand](/Users/monkey/Documents/dokku/codereading/photo/sshcommandcreate.png)
也就是说，在安装dokku paas平台，这个地步的时候，加入了了这个进入点。

###dokku push流程（git插件文件夹解读）

我们现在来浏览一遍，dokku完整的push一个应用都做了什么。
首先，我们先把dokku源码的目录整理一下。
![structure](/Users/monkey/Documents/dokku/codereading/photo/structure.png)
其次，我们已经说过了，dokku修改了进入点，当执行git-receive-pack的时候，并不是直接直接执行，而是修改成了/usr/local/bin/dokku git-receive-pack，那么这样一来，就是调用dokku的命令了。我们从目录结构里面找到dokku文件，在这里面的case里面，并没有找到git-receive-pack这个case，但是，我们找到了一个通配的东西
![dokku-git*](/Users/monkey/Documents/dokku/codereading/photo/dokku-git*.png)
从这里面可以看到，如果找不到对应的case，就去$PLUGIN_PATH/*/command里面去找，直到找到对应的东西。$PLUGIN_PATH路径下的东西就是我们源码中plugins路径下面的东西，我们可以直接在源码里面找。plugins里面的东西的命名还是很科学的，基本每个文件下的都是相关的hook。我们可以在plugins/git/command里面找到相关的东西

![git-git*](/Users/monkey/Documents/dokku/codereading/photo/git-git*.png)
这里当遇到是git-receive-pack命令的时候，就先初始化一个空的git仓库，然后写了一个脚本，给这个脚本附上可执行权限，并执行。脚本里执行的是
`dokku git-hook $APP`
同样的，我们在相同的文件里面，找到git-hook
![git-hook](/Users/monkey/Documents/dokku/codereading/photo/git-hook.png)
这里主要是判断用户push是否push到了master分支上，如果没有，提示如何push，如果正确push了，就执行,说明这是一个新push的应用。
`pluginhook receive-app $APP $newrev`
既然是pluginhook方式的调用，我们肯定要在$PLUGIN_PATH里面查找这个文件了，这个文件也在git文件夹里面
![receive-app](/Users/monkey/Documents/dokku/codereading/photo/receive-app.png)
可以看到，这里执行了
`dokku git-build $APP $REV`
![git-build](/Users/monkey/Documents/dokku/codereading/photo/git-build.png)
这里有两个case，git-build判断是否这个app正在部署，或者被加锁了，否则就加锁，开始部署。git-build-locked就要开始真正的部署了。同样的文件夹找到git_build_app_repo()，
![git-build-app](/Users/monkey/Documents/dokku/codereading/photo/git-build-app.png)
首先，检查一下app name，看看是否为空，或者是否不存在这个app目录，verify_app_name()这个函数在plugins/common/functions这里面，然后建造一个用来run这个应用的临时目录，并把应用拷贝到这个临时目录，最后，通过查看应用中有没有Dockerfile来判断，接下来执行的命令。但是不管有没有dockerfile，都会执行dokku receive命令。
终于，我们在dokku源码文件中，找到了receive的case分支。
