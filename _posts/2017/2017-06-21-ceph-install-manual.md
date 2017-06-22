--- 
layout: post
title: 手动搭建ceph集群
category: ceph installation
tags: [原创,ceph manual installation]
---



## 概览

如下我们是在无外网环境的centos7.1上手动安装ceph集群。
<pre>
cat /etc/redhat-release

Centos Linux release 7.1.1503 (Core)
</pre>

我们这里采用了4台虚拟机：

|        ``主机IP``          |         ``部署组件``             |     ``主机名``      |
|:--------------------------:|:-------------------------------:|:------------------:|
| ``172.20.30.196``          | ``admin-node``                  |   ``ceph001-admin``|
| ``10.133,134,211``         | ``node1``                       |  ``ceph001-node1`` |
| ``10.133.134.212``         | ``node2``                       |  ``ceph001-node2`` |
| ``10.133.134.213``         | ``node3``                       |  ``ceph001-node3``|

1)	在上述所有节点上修改/etc/hosts中添加如下内容：
<pre>
# For Ceph Cluster
172.20.30.196	ceph001-admin
10.133.134.211	ceph001-node1
10.133.134.212	ceph001-node2
10.133.134.213	ceph001-node3
</pre>


2) 修改主机名

在上述所有节点上分别修改/etc/sysconfig/network文件，指定其主机名分别为``ceph001-admin、ceph001-node1、ceph001-node2、ceph001-node3``。例如：

![manual-inst-hostname.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-hostname.png)

上述修改需要在系统下次重启时才生效。

此外，我们需要分别在每一个节点上执行hostname命令来立即更改主机名称。例如：

![manual-inst-hostname2.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-hostname2.png)

3）	通过主机名测试各节点之间是否联通

例如：

![manual-inst-ping.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-ping.png)


4）	检查当前CentOS内核版本是否支持rbd，并装载rbd模块
<pre>
modinfo rbd
modprobe rbd    #装载rbd模块
lsmod | grep rbd  #查看模块是否已经装载
</pre>


5）	安装ntp，并配置ceph集群节点之间的时间同步

在各节点执行如下命令：
<pre>
rpm –qa | grep ntp     #查看当前是否已经安装ntp
ps –ef | grep ntp      # 查看ntp服务器是否启动
ntpstat                 #查看当前的同步状态
</pre>

在/etc/ntp.conf配置文件中配置时间同步服务器地址

``参看：http://www.centoscn.com/CentosServer/test/2016/0129/6709.html``

6）关闭iptables

这里我们直接关闭iptables
<pre>
systemctl stop firewalld.service  ##停止iptables服务
systemctl disable firewalld.service  ##关闭iptables开机运行
</pre>


7)	TTY

在CentOS及RHEL上，当你尝试执行ceph-deploy时，你也许会收到一个错误。假如requiretty在你的Ceph节点上默认被设置了的话，可以执行``sudo visudo``然后定位到Defaults requiretty的设置部分，将其改变为Defaults:ceph !requiretty或者直接将其注释掉

**``NOTE:假如直接修改/etc/sudoers，确保使用sudo visudo，而不要用其他的文本编辑器``**

8） SELINUX

在CentOS及RHEL上，SELINUX默认会设置Enforcing。为了顺利的进行安装，我们建议设置SELINUX为Permissive或者完全禁止它。设置SELINUX值为Permissive，执行如下的命令：
<pre>
sudo setenforce 0
</pre>

为了永久的配置SELinux（假如SELinux会影响到Ceph集群的安装、运行的话），请修改配置文件/etc/selinux/config.
<pre>
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config         ##关闭SELinux开机启动
<pre>

