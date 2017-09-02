---
layout: post
title:  "基于pxe/kickstart/tftp/http的linux操作系统深度自定义安装--原理与实践"
date:   2017-09-02 14:08:23 +0800
categories: pxe kickstart
---

# 基于pxe和kickstart的linux操作系统深度自定义安装
			原理与实践

* 网络安装操作系统原理解析
* Linux下initrd.img作用理解和添加自定义二进制程序
* 自动化 

*本文重点是讲定制化安装的原理，PXE/TFTP/Kickstart全套的安装和使用，本文不详细介绍*
 
## 一、缘由：
 
对于平台运维来说，服务器操作系统安装以及安装后新系统的检查、初始化和配置工作，是一项经常要做、很费时间、容易出错、不怎么产生价值但又不得不做的工作。不但占用我们大量的人力，而且存在响应速度慢、操作时间长、结果不一致等问题。
想要解决这些问题，释放人力和提高用户的满意度，需要解决以下几个问题：
1. 自动化。不需要人工干预，自动化安装操作系统。
2. 自助式。用户有相应的权限，即可自己操作按钮来进行操作系统的安装。
3. 完整性。将硬件检查、raid配置、系统安装、初始化等一系列步骤打包在一起，一次完成。
4. 信息反馈。将安装进展或者出现的问题及时反馈给用户。
 
明确了需求之后，就可以开始进行技术筛选和学习了。
 
 
## 二、原理研究：
对于操作系统部署这块，基本上都是基于pxe和kickstart进行的自动化部署。我决定对它们两的原理进行学习。
 
2.1. 什么是PXE
 
PXE(Pre-boot Execution Environment，预启动执行环境)，工作于Client/Server的网络模式，支持Server通过网络从远端服务器下载映像，并由此支持通过网络启动操作系统，在启动过程中，终端要求服务器分配IP地址，再用TFTP（trivial file transfer protocol）协议下载一个启动软件包到本机内存中执行，由这个启动软件包完成终端基本软件设置，从而引导预先安装在服务器中的终端操作系统。
相当于可以从网络远程加载操作系统。现在基本上所有的网卡均支持PXE。
 
 
 
2.2. 什么是Kickstart
 
Kickstart是一种无人值守的安装方式。它的工作原理是在安装过程中记录典型的需要人工干预填写的各种参数，并生成一个名为ks.cfg的文件。如果在安装过程中出现要填写参数的情况，安装程序首先会去查找Kickstart生成的文件，如果找到合适的参数，就采用所找到的参数；如果没有找到合适的参数，便需要安装者手工干预了。所以，如果Kickstart文件涵盖了安装过程中可能出现的所有需要填写的参数，那么安装者完全可以只告诉安装程序从何处取ks.cfg文件，然后就去忙自己的事情。等安装完毕，安装程序会根据ks.cfg中的设置重启系统，并结束安装。.
说白了，就是以前需要手工输入的参数，可以预先写在配置文件中，然后安装过程就不需要人工干预了。
 
 
 
PXE+Kickstart 无人值守安装操作系统完整过程如下：
 

![PXE]({{ site.url }}/assets/pxe.png)
* 以上三个Server只是逻辑上的区分，可以在一台物理Server上实现。

（图一）
 
 
整个过程可以大致分为三个阶段：
* 第一阶段：
通过dhcp Server获取自己的IP地址信息以及TFTP server的地址。
* 第二阶段：
从TFTP server获取pxelinux.0启动文件、pxelinux.cfg配置文件。
* 第三阶段：
读取pxelinux.cfg配置文件，获取vmlinuz、initrd.img和kickstart文件路径，然后运行vmlinuz。
 
 
2.3. 我们来看一个pxelinux.cfg配置文件：
```
default linux
prompt 0
timeout 1
label linux
    kernel /autoOSimages/CentOS6.5-x86_64/vmlinuz
    ipappend 2
append initrd=/autoOSimages/CentOS6.5-x86_64/initrd.img ksdevice=em1 ip=dhcp noipv6 kssendmac text ks=http://192.168.1.2/ks.cfg
``` 

基本上就是提供了vmlinuz/initrd.img以及kickstart文件的来源信息。
 
 
2.4. 我们研究一下kickstart文件（内容过长，就不贴完整配置了），发现里面有一个%pre和%post的配置段，官网的解释是：

_Pre-Installation Script:You can add commands to run on the system immediately after the ks.cfg has been parsed._

_Post-Installation Script :You have the option of adding commands to run on the system once the installation is complete._

意思很清楚，在%pre中的内容，就会在安装系统前运行。%post中的内容，会在安装完系统后运行。正好啊，我的检查硬件配置、配置raid啥的，不就要放在安装系统之前嘛。而我初始化、安装salt等，正好是需要在安装完系统之后啊。
 
 
## 三、自定义安装过程：
vmlinuz就是一个精简的Linux操作系统内核，加载了它，相当于就在内存中运行了一套操作系统。那么，如果有操作系统的话，那不是想干什么都可以了吗？
让我们仔细研究一下。
 
原来Linux操作系统安装ISO文件里面，都放有专门用来进行pxe网络启动用的文件，一般都存放在images/pxeboot目录下：
 
![PXE]({{ site.url }}/assets/pxe-image.png)  

（图二）

vmlinuz文件就是一个Linux操作系统内核，而initrd.img文件是一个初始的内存磁盘，它主要包括了启动过程中可能需要的硬件驱动程序及相关文件。那有没有可能对修改initrd.img文件，增加一些程序，让它干一些我们需要的工作呢？完全可以。
 
 
 
3.1、其实initrd.img是一个压缩包，解压开之后就是一个普通的目录：
 
```
mv initrd.img initrd.lzma
lzma -d initrd.lzma
```
 
解压后目录：

![PXE]({{ site.url }}/assets/pxe-dir.png)

（图三）

 
 
那我们可以把需要的工具放进去，再重新打包，比如添加HP的raid卡配置工具：
![PXE]({{ site.url }}/assets/pxe-hp.png)

（图四）
 
 
还可以添加HP的raid卡驱动内核模块：

![PXE]({{ site.url }}/assets/pxe-hp-module.png)

（图五）
 
 
 
 
3.2、重新打包成initrd.img文件，命令如下：
```
find . -print | cpio -o -H newc >  /tmp/newinit/initrd.img
lzma -z /tmp/newinit/initrd.img
```
 
将修改过的文件替换TFTP Server上的initrd.img即可。
 
 
 
3.3、在kickstart文件的%pre段添加如下内容：

![PXE]({{ site.url }}/assets/pxe-pre.png)

（图六）
这样就会在安装操作系统之前，将硬件的raid配置好。
 
 
同样，在kickstart文件的%post段添加如下内容，可以在操作系统安装完成后，进行初始化和安装salt等工作：

![PXE](/assets/pxe-post.png)
 
（图七）
 
 
 
## 四、后续工作：
有了pxe和kickstart这一套工具的经验，在前端再套上一个按钮，完成权限管理；通过程序，自动生成pxe和kickstart配置文件，一套自动安装操作系统的系统就完整了。
 
 
## 五、总结：
开源工具的特点是原理开放、源代码开放、可以非常灵活地进行裁剪和组合、各种各样的工具都存在，只是使用方法不一样。只要你明确知道自己的需求，在很大程度上都可以通过对现有工具的改造来达到自己的目标。要善于利用这一特点。
 
同时，对于日常使用的工具，不但要知其然，还要知其所以然，只有懂得了原理，才能更灵活地加以利用，而不是受制于工具。
 
 
 
 
