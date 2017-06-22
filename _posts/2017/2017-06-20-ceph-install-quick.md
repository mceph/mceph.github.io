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
<br /><br />

**1.2.1 INSTALL NTP**

我们建议在Ceph所有的节点上（特别是Ceph Monitor节点）安装NTP来阻止由于时钟偏移所产生的问题。针对Debian/Ubuntu上执行如下命令：
<pre><code>
# sudo apt-get install ntp
</code></pre>

确保使能了NTP service，并且每一个Ceph节点都使用同样的NTP时间服务器。可以查看NTP对应的网站了解详细信息：http://www.ntp.org/

安装完成后通过如下命令查看NTP是否正常工作：

![ceph-install-ntpsrv.png](https://mceph.github.io/assets/images/2017/ceph-inst/ntp-service.png)
<br /><br />

**1.2.2 INSTALL SSH SERVER**

针对所有的Ceph节点执行如下的步骤：

1）	安装一个SSH server到每一个Ceph节点上
<pre><code>
# sudo apt-get install openssh-server
</code></pre>

2） 确保SSH运行在所有Ceph节点上
<pre><code>
# service --status-all | grep ssh

 [ + ] ssh
</code></pre>
<br /><br />

**1.2.3 CREATE A CEPH DEPLOY USER**

ceph-deploy工具必须登录到Ceph节点上，并且必须具有passwordless sudo特权，因为需要登录到相应的节点上无密码的安装软件及配置文件。

最近版本的ceph-deploy支持--username选项 ，因此你可以指定任何用户使其具有password-less sudo权限（也包括root用户，但是并不推荐这样做）。为了使用``ceph-deploy --username {username}``，你所指定的用户必须有通过SSH password-less的方式访问Ceph节点的权利，这样ceph-deploy就不会提示你输入密码。

我们推荐为ceph-deploy创建一个特定的用户来访问ceph集群节点。请不要使用“ceph”作为用户名。一个统一的用户名可以方便的对集群进行管理，但你应该避免用一些通常见到的用户名，这样可以防止黑客进行暴力的破解（例如：root、admin、{productname}）。如下描述如何创建passwordless sudo用户的例子中请使用你自己定义的用户名来替换{username}。

``Note: "ceph"用户名保留内部Ceph daemon使用``

如下我们统一使用：

``用户名：test1001``

``密码:  123456``


1）	在每一个Ceph节点上创建一个新的用户
<pre><code>
# ssh user@ceph-server
# sudo useradd -d /home/{username} -m {username}
# sudo passwd {username}
</code></pre>
我们可以直接通过secureCRT登录到每一个节点上创建：
<pre><code>
# sudo useradd -d /home/test1001 -m test1001
# sudo passwd test1001
</code></pre>

2） 对于你为每一个Ceph添加的用户，确保其具有sudo权限
<pre><code>
# echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
# sudo chmod 0440 /etc/sudoers.d/{username}
</code></pre>


针对我们的test1001用户：
<pre><code>
# echo "test1001 ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/test1001
# sudo chmod 0440 /etc/sudoers.d/test1001
</code></pre>

<br />

**1.2.4 ENABLE PASSWORD-LESS SSH**

因为ceph-deploy并不会尝试输入密码，因此你必须在admin node(这里为192.168.190.128虚拟机)上产生SSH key，并将公钥分发到每一个Ceph节点上。

1）	产生SSH key，但是不要使用sudo或者root用户，并保持passphase为空
<pre><code>
# ssh-keygen

Generating public/private key pair.
Enter file in which to save the key (/ceph-admin/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /ceph-admin/.ssh/id_rsa.
Your public key has been saved in /ceph-admin/.ssh/id_rsa.pub.
</code></pre>

这里我们都统一切换为我们上面创建的test1001用户，然后再执行ssh-keygen:

![ceph-install-sshkeygen.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-sshkeygen.png)

2）	将key拷贝到每一个Ceph节点，请用你上面所创建的用户名替换``{username}``
<pre><code>
ssh-copy-id {username}@node1
ssh-copy-id {username}@node2
ssh-copy-id {username}@node3
</code></pre>

这里我们执行如下拷贝：
<pre><code>
ssh-copy-id test1001@192.168.190.128
ssh-copy-id test1001@192.168.190.129
ssh-copy-id test1001@192.168.190.130
ssh-copy-id test1001@192.168.190.131
</code></pre>

如下图所示：


![ceph-install-sshidcpy.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-sshidcpy.png)

我们尝试执行如下命令，看是否能够无密码登录到相应节点上：
<pre><code>
# ssh test1001@192.168.190.128
# ssh test1001@192.168.190.129
# ssh test1001@192.168.190.130
# ssh test1001@192.168.190.131
</code></pre>


3）	通过hostname方式访问对应的ceph节点

后续很多地方我们会通过hostname，而不是直接的IP地址的方式来访问ceph节点，因此这里我们修改所有节点的/etc/hosts文件来实现该功能。
<pre><code>
# 192.168.190.128	ceph-admin
# 192.168.190.129	ceph-node1-mon
# 192.168.190.130	ceph-node2-osd
# 192.168.190.131	ceph-node3-osd
</code></pre>

4）推荐修改ceph-deploy admin节点（这里为192.168.190.128）的~/.ssh/config文件，使ceph-deploy不需要在登录ceph集群节点的时候每一次都添加 ``--username {username}``选项，对于ssh、scp等也可使用这一功能

这里切换到test1001用户，修改/home/test1001/.ssh/config文件：

![ceph-install-sshcfg.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-sshcfg.png)


添加完成之后，我们用ssh测试一下，不添加用户名也可以直接登录到对应的node节点：

![ceph-install-sshlogin.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-sshlogin.png)

``注意：这里第一次使用ssh ceph-node1-mon的时候会要求你进行确认是否连接，后续则可以直接连接了。``
<br /> <br />


**1.2.5 ENABLE NETWORKING ON BOOTUP**

Ubuntu16.04 暂时不需要做任何改变
<br /><br />

**1.2.6 ENSURE CONNECTIVITY**

确保我们可以通过ping {hostname}的方式连通集群的各个节点，hostname地址解析在Ceph环境中很多地方会用到。注意，我们不应该将hostname解析成为一个127.0.0.1这样的本地回环地址。

这里我们通过上面的步骤修改/etc/hosts，已经完成了这一功能，在此只是再确认一下此功能是否有问题。

<br /><br />

**1.2.7 OPEN REQUIRED PORTS**

Ceph monitors默认采用6789端口。Ceph OSDs默认采用6800~7300区间的端口进行通信。Ceph OSDs可以通过许多网络连接来与clients、monitors、其他OSDs进行通信。详细的网络配置可参考：http://docs.ceph.com/docs/master/rados/configuration/network-config-ref

在有一些环境中，默认的防火墙配置相当严格。你也许需要调整防火墙的设置以使客户端能够与Ceph daemons进行通信。

这里针对Ubuntu，我们暂时先禁止防火墙：
<pre><code>
sudo ufw disable
</code></pre>
<br /><br />



## 2. STORAGE CLUSTER QUICK START

假如你暂时还未完成PREFLIGHT流程，请先参考上节完成。本章将会介绍通过在admin node上使用ceph-deploy来快速部署一套Ceph Storage Cluster。我们会创建拥有3个Ceph节点的集群来演示Ceph的一些功能
![ceph-install-toplogic1.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-toplogic-1.png)

为了方便，我们再一次给出如上部署图。每一个节点对应的配置及IP地址请参看上节。

作为一个例子，我们首先会创建一个拥有1个Ceph Monitor，2个 Ceph OSD Daemons的集群。一旦该集群达到active + clean状态，可以通过再增加1个Ceph OSD Daemon，1个Metadata Server，2个Ceph Monitors来对集群进行扩展。最好在admin node上创建一个来存放ceph-deploy在部署集群时所产生的配置文件及keys文件。

<pre><code>
# mkdir ceph-cluster
# cd ceph-cluster
</code></pre>
