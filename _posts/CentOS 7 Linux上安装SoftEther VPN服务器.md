---
title: SoftEther VPN
date: 2020-04-06 14:14:36
tags: VPN,代理
---



# CentOS 7 Linux上安装SoftEther VPN服务器

虚拟专用网（VPN），是一种常用于连接中、大型企业或团体与团体间的私人网络的通讯方法。虚拟私人网络的讯息透过公用的网络架构来传送内联网的网络讯息。它利用已加密的通道协议来达到保密、发送端认证、消息准确性等私人消息安全效果。这种技术可以用不安全的网络来发送可靠、安全的消息。

[SoftEther VPN](https://www.softether.org/)是由日本筑波大学的[登 大遊](http://dnobori.cs.tsukuba.ac.jp/en/)在硕士论文中提出的开源、跨平台、多重协议的虚拟专用网方案，是专门为穿过防火墙而设计的。我们可以用它在自己的云主机上搭建一个简单的VPN来使用。本文将介绍如何在CentOS Linux系统上搭建SoftEther VPN服务。并在Windows客户端上连接始用！

## 服务器端  

首先要在服务器上下载并解压安装文件，注意是32位还是64位系统（可通过uname -a命令查看），我这里下载的是64位：

```bash
wget http://www.softether-download.com/files/softether/v4.25-9656-rtm-2018.01.15-tree/Linux/SoftEther_VPN_Server/64bit_-_Intel_x64_or_AMD64/softether-vpnserver-v4.25-9656-rtm-2018.01.15-linux-x64-64bit.tar.gz
```

将下载好的软件包解压到/usr/local/目录下：

~~~bash
tar zxvf softether-vpnserver-v4.25-9656-rtm-2018.01.15-linux-x64-64bit.tar.gz -C /usr/local/
~~~

~~~bash
cd /usr/local/vpnserver/
~~~

**进入解压好的目录开始安装：**

在编译安装之前先要安装一下系统常用软件：

```bash
yum -y install make cmake gcc gcc-c++ gcc-g77 flex bison file libtool libtool-libs autoconf kernel-devel patch wget crontabs libjpeg libjpeg-devel libpng libpng-devel libpng10 libpng10-devel gd gd-devel libxml2 libxml2-devel zlib zlib-devel glib2 glib2-devel unzip tar bzip2 bzip2-devel libevent libevent-devel ncurses ncurses-devel curl curl-devel libcurl libcurl-devel e2fsprogs e2fsprogs-devel krb5 krb5-devel libidn libidn-devel openssl openssl-devel vim-minimal gettext gettext-devel ncurses-devel gmp-devel pspell-devel unzip libcap diffutils ca-certificates net-tools libc-client-devel psmisc libXpm-devel git-core c-ares-devel libicu-devel libxslt libxslt-devel xz expat-devel
```

~~~bash
make ## 执行
~~~



> SoftEther VPN Server (Ver 4.25, Build 9656, Intel x64 / AMD64) for Linux Install Utility
> Copyright (c) SoftEther Project at University of Tsukuba, Japan. All Rights Reserved.
> \-------------------------------------------------------------------
>
> Do you want to read the License Agreement for this software ?
>
> \1. Yes
>  \2. No
>
> Please choose one of above number:
> 1

以上是提示是否阅读license许可，那是必须读了，所以输入：1，回车。

> NOTES
>
> SoftEther provides source codes of some GPL/LGPL/other libraries listed above on its web server. Anyone can download, use and re-distribute them under individual licenses which are contained on each archive file, available from the following URL:
> http://uploader.softether.co.jp/src/
> Did you read and understand the License Agreement ?
>
> (If you couldn't read above text, Please read 'ReadMeFirst_License.txt'
>  file with any text editor.)
>
> \1. Yes
>  \2. No
>
> Please choose one of above number:
> 1

继续输入：1，回车。

> Did you agree the License Agreement ?
>
> \1. Agree
> \2. Do Not Agree
>
> Please choose one of above number:

是否同意，输入：1，同意，回车。之后会执行编译安装过程，结束之后可以始用`echo $?`，如果返回0，说明安装成功。

```bash
./vpnserver start    #开启服务
./vpnserver stop     #关闭服务
```

## 设置远程管理密码 

启动成功后需要设置远程管理密码以便远程管理VPN服务器。运行`./vpncmd`进入VPN的命令行：

```bash
vpncmd command - SoftEther VPN Command Line Management Utility
SoftEther VPN Command Line Management Utility (vpncmd command)
Version 4.25 Build 9656   (English)
Compiled 2018/01/15 10:17:04 by yagi at pc33
Copyright (c) SoftEther VPN Project. All Rights Reserved.

By using vpncmd program, the following can be achieved.

1. Management of VPN Server or VPN Bridge
2. Management of VPN Client
3. Use of VPN Tools (certificate creation and Network Traffic Speed Test Tool)

Select 1, 2 or 3:
1

#选择1，然后出现：

Specify the host name or IP address of the computer that the destination VPN Server or VPN Bridge is operating on.
By specifying according to the format 'host name:port number', you can also specify the port number.
(When the port number is unspecified, 443 is used.)
If nothing is input and the Enter key is pressed, the connection will be made to the port number 8888 of localhost (this computer).
Hostname of IP Address of Destination: localhost:5555

#这里需要选择地址和端口，我的云主机用了SSL占用了443端口，所以默认的443端口不能用，所以改用了5555端口，所以在这里输入localhost:5555，然后出现：

If connecting to the server by Virtual Hub Admin Mode, please input the Virtual Hub name.
If connecting by server admin mode, please press Enter without inputting anything.
Specify Virtual Hub Name:[这里就是指定一个虚拟HUB名字，用默认的直接回车就行。]
Connection has been established with VPN Server "localhost" (port 5555).

You have administrator privileges for the entire VPN Server.

VPN Server>
```

这时需要输入`ServerPasswordSet`命令设置远程管理密码，确认密码后就可以通过Windows版的SoftEther VPN Server Manager远程管理了。

```bash
VPN Server>ServerPasswordSet
ServerPasswordSet command - Set VPN Server Administrator Password
Please enter the password. To cancel press the Ctrl+D key.

Password: ****************
Confirm input: ****************

The command completed successfully.
```

最后需要在服务器防火墙上开启相关的端口，如果是云主机，还需要在控制台安全组放行对应的端口。这里配置的是iptables：

```bash
iptables -A INPUT -p udp -m multiport --dport 4500,1701,1194,40000,500 -j ACCEPT
```

如果不是iptables的话，开放防火墙端口

~~~bash
firewall-cmd --zone=public --add-port=5555/tcp --permanent
~~~

~~~bash
firewall-cmd --reload  ##重启
~~~



## VPN Server Manager 远程管理配置（windows）

下载地址 :https://www.softether-download.com/cn.aspx?product=softether

   

下载界面

![image-20200406155139488](https://wanglei.club/image-20200406155139488.png)



下载完成点击安装



![img](..\images\20180520135425.png)

下一步：

![img](https://wanglei.club/20180520135442.png)

这里选择第三个（管理工具），下一步安装结束，打开软件：

![img](https://wanglei.club/20180520141603.png)

点击新设置

主机名填服务器的地址或域名，端口选择之前设置的5555，密码设置之前设置的管理密码：

![img](https://wanglei.club/20180520141845.png)

![img](https://wanglei.club/20180520142012.png)

选择远程访问VPN Server

![img](https://wanglei.club/20180520142040.png)



启动L2TP，设置IPsec 预共享密钥

![img](https://wanglei.club/20180520142302.png)



这个没什么用，禁用确当即可。

![img](https://wanglei.club/20180520142443.png)



![img](https://wanglei.club/20180520142804.png)



这里新建用户，用于连接VPN服务器。如图配置好，点确定。

![img](https://wanglei.club/20180520143006.png)



这里点击管理虚拟HUB。

![img](https://wanglei.club/20180520143050.png)



![img](https://wanglei.club/20180520143119.png)



![img](https://wanglei.club/20180520143207.png)





## WINDOWS客户端连接

下载client  地址:https://www.softether-download.com/cn.aspx?product=softether

![image-20200406153244932](https://wanglei.club/image-20200406153244932.png)



下载完成后打开

![技术分享图片](https://wanglei.club/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=.png)



填入服务端 的配置

![技术分享图片](https://wanglei.club/1231234.png)



双击 连接

![技术分享图片](https://wanglei.club/313114.png)





## MACOS 客户端 连接

MACOS 下载使用【SoftEtherGUI】   版本已经不更新了

下载地址： http://softethergui.lastgrid.com/download/ 

> 注意： MACOS 10.15版本 会自动退出多次，安装后需要重启电脑 



配置服务端连接

![image-20200406153748771](https://wanglei.club/image-20200406153748771.png)



填写完成后点击Apply



选择配置好的地址 并且打开下面的开关



![image-20200406153911910](https://wanglei.club/image-20200406153911910.png)