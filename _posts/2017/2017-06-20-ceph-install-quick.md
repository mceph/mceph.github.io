--- 
layout: post
title: 快速搭建ceph集群
category: ceph installation
tags: [原创,ceph quick installation]
---

ceph存储集群环境的搭建较为复杂，即使参照官方文档，也容易在实际搭建过程中遇到各种各样的问题。这里我们详细的介绍两种安装方法：
* [ceph-deploy快速搭建ceph集群](https://mceph.github.io/ceph%20installation/2017/06/20/ceph-install-quick.html)
* [手动方式搭建ceph集群](https://mceph.github.io/ceph%20installation/2017/06/21/ceph-install-manual.html)

本文主要介绍在Ubuntu16.04上通过ceph-deploy工具来快速部署一套ceph集群。

## 1. PREFLIGHT

这里我们先给出我们将要创建的**Ceph Storage Cluster**的拓扑结构图。
![ceph-install-toplogic1.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-toplogic-1.png)

如上图所示，我们在admin节点上采用ceph-deploy来构建一个拥有3-node的Ceph存储集群。

在我们的构建过程中，我们采用4台Ubuntu16.04虚拟机：

|        ``主机IP``          |         ``部署组件``             |     ``主机名``      |
|:--------------------------:|:-------------------------------:|:------------------:|
| ``192.168.190.128``        | ``admin-node(ceph-deploy)``     |   ``ceph-admin``   |
| ``192.168.190.129``        | ``node1(mon.node1)``            |  ``ceph-node1-mon``|
| ``192.168.190.130``        | ``node2(osd.0)``                |  ``ceph-node2-osd``|
| ``192.168.190.131``        | ``node3(osd.1)``                |  ``ceph-node3-osd``|

<br />
修改上述每一个主机上的/etc/hosts文件，添加如下:
<pre>
192.168.190.128         ceph-admin-3
192.168.190.129         ceph-node1-mon
192.168.190.130         ceph-node2-osd
192.168.190.131         ceph-node3-osd
</pre>
<br />
修改每一个节点的主机名，将上面每一台虚拟机的主机名分别更改为ceph-admin,ceph-node1-mon,ceph-node2-osd,ceph-node3-osd。例如修改192.168.190.128主机的名字为ceph-admin：
<pre>
# hostname ceph-admin
</pre>
同时修改/etc/hostname文件，以使对主机名的更高永久生效。

### 1.1 CEPH DEPLOY SETUP
添加Ceph仓库到``ceph-deploy`` admin node，然后开始安装ceph-deploy。
针对DEBIAN/UBUNTU执行如下步骤：

1） 添加release key
<pre><code>
# wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
</code></pre> 

2） 添加ceph包到仓库中，请用具体的Ceph稳定版本号替换``{ceph-stable-release}``(例如：hammer,jewel等)
<pre><code>
# echo deb https://download.ceph.com/debian-{ceph-stable-release}/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
</code></pre>

这里我们选用jewel版本：
<pre>
# echo deb https://download.ceph.com/debian-jewel/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
</pre>
``NOTE: Ceph的一些发布版本可以在这里找到(http://docs.ceph.com/docs/master/releases/)``

3） 更新源仓库并安装ceph-deploy
<pre><code>
# sudo apt-get update && sudo apt-get install ceph-deploy
</code></pre>


### 1.2 CEPH NODE SETUP
admin node(管理节点）必须要能够通过SSH password-less的访问Ceph节点。当ceph-deploy登录到ceph节点上的时候，该特殊的用户必须具有passwordless sudo特权。

**INSTALL NTP**

我们建议在Ceph所有的节点上（特别是Ceph Monitor节点）安装NTP来阻止由于时钟偏移所产生的问题。针对Debian/Ubuntu上执行如下命令：
<pre><code>
# sudo apt-get install ntp
</code></pre>

确保使能了NTP service，并且每一个Ceph节点都使用同样的NTP时间服务器。可以查看NTP对应的网站了解详细信息：http://www.ntp.org/

安装完成后通过如下命令查看NTP是否正常工作：

![ceph-install-ntpsrv.png](https://mceph.github.io/assets/images/2017/ceph-inst/ntp-service.png)

**INSTALL SSH SERVER**
针对所有的Ceph节点执行如下的步骤：

1）	安装一个SSH server到每一个Ceph节点上
<pre><code>
# sudo apt-get install openssh-server
</pre><code>

2） 确保SSH运行在所有Ceph节点上
<pre><code>
# service --status-all | grep ssh

 [ + ] ssh
</pre><code>