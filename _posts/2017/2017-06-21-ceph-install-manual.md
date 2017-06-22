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
</pre>

<br /><br />









## 1. Get Software


### 1.1 ADD KEYS

添加key到你的系统受信任keys列表来避免一些安全上的警告信息，对于major releases(例如hammer，jewel)请使用release.asc。

我们先从https://download.ceph.com/keys/release.asc 下载对应的release.asc文件，上传到集群的每一个节点上，执行如下命令：
<pre>
sudo rpm --import './release.asc'
</pre>



### 1.2 DOWNLOAD PACKAGES

假如你是需要在无网络访问的防火墙后安装ceph集群，在安装之前你必须要获得相应的安装包。注意，你的连接外网的下载ceph安装包的机器环境与``你实际安装ceph集群的环境最好一致，否则可能出现安装包版本不一致的情况而出现错误。``。

**``RPM PACKAGES``**

先新建三个文件夹dependencies、ceph、ceph-deploy分别存放下面下载的安装包。

1)	Ceph需要一些额外的第三方库。添加EPEL repository，执行如下命令：
<pre>
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
</pre>

Ceph需要如下一些依赖包：

* snappy
* leveldb
* gdisk
* python-argparse
* gperftools-libs

现在一台可以连接外网的主机上下载这些依赖包，存放在dependencies文件夹：
<pre>
sudo yumdownloader snappy
sudo yumdownloader leveldb
sudo yumdownloader gdisk
sudo yumdownloader python-argparse
sudo yumdownloader gperftools-libs
</pre>


2)	安装yum-plugin-priorities
<pre>
sudo yum install yum-plugin-priorities
</pre>
>
修改/etc/yum/pluginconf.d/priorities.conf文件：
<pre>
[main]
enabled = 1
</pre>


3)	通过如下的命令下载适当版本的ceph安装包
<pre>
su -c 'rpm -Uvh https://download.ceph.com/rpm-{release-name}/{distro}/noarch/ceph-{version}.{distro}.noarch.rpm'
</pre>

也可以直接到官方对应的网站去下载：

``https://download.ceph.com/``


这里我们在CentOS7.1上配置如下：
<pre>
su -c 'rpm -Uvh https://download.ceph.com/rpm-jewel/el7/noarch/ceph-release-1-0.el7.noarch.rpm'
</pre>

修改/etc/yum.repos.d/ceph.repo文件，添加priority项：

![manual-inst-priority.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-priority.png)

下载ceph安装包：
<pre>
yum clean packages            #先清除本地可能缓存的一些包
yum clean
sudo yum install --downloadonly --downloaddir=/ceph-cluster/packages/ceph ceph ceph-radosgw
</pre>

**``NOTE：这里我们目前就下载ceph,ceph-radosgw两个包，其依赖的一些包会自动下载下来。如果在实际安装中，仍缺少一些依赖包，我们可以通过yum search ${package-name} 查找到该包，然后再下载下来.``**



4）下载ceph-deploy安装包
<pre>
sudo yum install --downloadonly --downloaddir=/ceph-cluster/packages/ceph-deploy ceph-deploy
</pre>

4)	将上述的依赖包分别打包压缩称dependencies.tar.gz、ceph.tar.gz、ceph-deploy.tar.gz，并上传到集群的各个节点上来进行安装

<br /><br />












## 2. INSTALL SOFTWARE

在我们下载好前面这些安装包之后，分别上传到集群各节点进行安装。



### 2.1 INSTALL CEPH DEPLOY

这里我们虽然是通过手动方式来安装ceph集群，但是我们还是可以安装ceph-deploy来让其协助我们完成手动安装。

在ceph001-admin节点上安装ceph-deploy包：
<pre>
sudo rpm –ivh ceph-deploy-1.5.37-0.noarch.rpm
</pre>

执行ceph-deploy看是否安装成功：

![manual-inst-deploy.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-deploy.png)
<br />


### 2.3 INSTALL CEPH STORAGE CLUSTER
本手册描述了如何手动的安装ceph包。如下的安装过程只适应于那些没有安装ceph-deploy、juju等工具的用户。

**INSTALL WITH RPM**

通过RPM包来安装CEPH，请执行如下的步骤：

1)	安装pre-requisite 包
在所有节点上安装如下包:
* snappy
* leveldb
* gdisk
* python-argparse
* gperftools-libs

![manual-inst-prequire.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-prequire.png)

执行如下命令进行安装：
<pre>
sudo yum localinstall *.rpm
</pre>


2）	在所有节点上安装ceph包


![manual-inst-cephpkg.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-cephpkg.png)

执行如下命令进行安装：
<pre>
sudo yum localinstall *.rpm
</pre>



### 2.4 MANUAL DEPLOYMENT

一个Ceph Cluster至少需要一个monitor，和尽可能多的OSDs来存放对象数据的副本。建立initial monitors是构建Ceph Storage Cluster的首要步骤。Monitor的部署会为整个集群建立重要的准则，比如pools的副本数，每一个OSD的placement groups数，heartbeat间隔，是否需要身份认证等。这些值的大多数都会被设置为默认值，因此在实际产品应用中构建集群的时候我们需要了解到这些。

我们参照如下图来构建集群，node1作为monitor node，node2与node3作为OSD nodes。

![manual-inst-toplogic.png](https://mceph.github.io/assets/images/2017/manual-inst/manual-inst-toplogic.png)

1）	**MONITOR BOOTSTRAPPING**

构建一个monitor需要如下一些步骤：

* ``Unique Identifier：``fsid是集群的唯一标识符，且代表着Ceph Storage Cluster所对应的Ceph Filesystem的文件系统ID。当前，ceph已经支持native interface，block device，object storage gateway interface，因此从这一方面来说fsid已经有些不恰当了。
* ``Cluster Name：``ceph clusters拥有一个cluster名称 ，它是一个不带空格的简单字符串。默认的集群名称为ceph。当你有多个集群的时候，为了清楚的区分当前的集群，修改默认的集群名称就显得很有意义了

例如，当你在一个federated architecture环境中运行多个ceph集群的时候，集群的名字(eg: us-west, us-east)标识着当前是哪一个集群的回话。 NOTE：为了在命令行接口区别cluster name，可以通过集群名字来命名ceph configuration（eg, ceph.conf，us-west.conf， us-east.conf等）。请参看：CLI usage (ceph --cluster {cluster-name})

* ``Monitor Name：``集群中的每一个monitor实例都有一个唯一的名字。通常情况下，ceph-monitor的名字会被设为主机的名字（我们建议一个host上只有一个Monitor，并且不会Ceph OSD Daemons与Ceph monitor混在一起）。你可以通过hostname –s来获得当前主机的名字

* ``Monitor Map:`` 建立initial monitor(s)通常需要建立一个monitor map。Monitor map需要fsid，cluster name(或者使用默认的ceph)，和至少一个主机名与它的IP地址
* ``Monitor Keyring：``monitor相互之间的通信是通过一个secret key来完成的。你必须通过一个monitor secret来产生一个keyring以供建立initial monitor使用
* ``Administrator Keyring：``要使用ceph CLI工具，你必须要有一个client.admin用户，因此你必须要产生一个admin user和keyring，然后你必须添加client.admin 用户到monitor keyring中

上面提到的这么一些需求并不意味着需要创建一个Ceph配置文件。然而，通常情况下，我们建议创建一个ceph配置文件，然后里面配有fsid，mon initial members，和mon host等


你也可以在运行过程中获得和设置monitor settings。然而，一个ceph配置文件只需要包含那些需要覆盖默认值的设置。当你添加设置到Ceph配置文件中的时候，这些设置会覆盖系统的默认值。在一个配置文件中维持这些默认值有助于方面的管理整个集群。
<br />

``整个建立步骤如下：``

1.1）登录initial monitors节点
<pre>
ssh {hostname}
</pre>



