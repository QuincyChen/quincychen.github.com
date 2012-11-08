---
layout: post
title: "使用Vagrant veewee建立base box"
---

[vagrant](http://vagrantup.com)是自动建立虚拟环境的利器，他基本的工作原理包括：
* 基于VirtualBox，使用其提供的API，将虚拟机的安装和基础配置自动化，并用可读脚本的形式保存下来；
* 基于Puppet或者chef，将软件安装和配置自动化，并且可记录。

由于Vagrant工作中利用的是可读的脚本，从而使得环境建立可以用Git等跟踪和追溯。同时由于其便于分发和保持团队工作的一致性，简直就是配置管理员的福音。

[veewee](https://github.com/jedi4ever/veewee/blob/master/doc/vagrant.md)是vagrant的插件，提供了常用的一系列basebox的配置模板，可以基于这些模板快速地建立basebox，从而简化工作。

以下是一些参考链接：
* [Vagrant Tricks and Troubleshooting](http://devops.me/2011/10/10/vagrant-tricks-and-troubleshooting/)
* [Windows7系统中使用vagrant构建Linux虚拟化开发环境](http://www.360ito.com/article/199.html)
* [用veewee创建vagrant的虚拟机](http://www.continuousdelivery.info/index.php/2011/11/04/veewee-vargrant/)
* [Customising a Vagrant box with Veewee](http://chrisyallop.com/2012/04/customising-a-vagrant-box-with-veewee/)

问题:

* 首先我碰到的第一个问题是，如何将现有的虚拟机文件直接包装成box，我想这是最起码的。vagrant package命令好像是，但我没有成功，有知道的请告诉我下。

* 建立box过程很长，机器可能因为网络环境等各种原因在post install阶段中断，那么从头装不是好办法。
   建立与虚拟机的ssh连接，假设你没有改变缺省端口号(7222)和用户名密码(vagrant/vagrant)
   ssh -p 7222 vagrant@127.0.0.1 
   连接上后，你可以随便使用机器了，此时看到目录下有postinstall.sh。这个是安装虚拟机后要执行的脚本，你可以直接执行：
   sudo ./postintall.sh
   重复后期安装的过程，当然，你可以注释掉完成的部分，只执行需要的部分

建立basebox基本步骤:

<pre class="prettyprint lang-cpp">

	vagrant basebox templates#列举可以获取的模板，若没有，你可以选择一个最接近的来进行编辑
    mkdir envir_dir; cd envir_dir;#建立环境目录并进入
    vagrant basebox define 'ubuntu1204' 'ubuntu-12.04.1-server-i386'#创建基本的环境配置定义
    find . #可以看到得到了definition目录和其下的三个文件,修改文件以便适合自己的要求
    #可以修改ISO文件为本地，前提是你已经下载了。要把下载的ISO文件放到与definition同级别的iso目录下
    vagrant basebox build ubuntu1204#开始建立虚拟机( long time .......zZZ^^^^)
    vagrant basebox export 'ubuntu1204'#导出虚拟机

    #使用vagrant 开始下面的其他工作
    vagrant box add 'ubuntu1204' ubuntu1204.box 
</pre>   
