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

# PREFLIGHT

这里我们先给出我们将要创建的**Ceph Storage Cluster**的拓扑结构图。
![ceph-install-toplogic1.png](https://mceph.github.io/assets/images/2017/ceph-inst/ceph-inst-toplogic-1.png)

如上图所示，我们在admin节点上采用ceph-deploy来构建一个拥有3-node的Ceph存储集群。

在我们的构建过程中，我们采用4台Ubuntu16.04虚拟机：

*---------------------------------------------------------------------------------*
*192.168.190.128  --------  admin-node(ceph-deploy)     ceph-admin*
*192.168.190.129  --------  node1(mon.node1)            ceph-node1-mon*
*192.168.190.130  --------  node2(osd.0)                ceph-node2-osd*
*192.168.190.131  --------  node3(osd.1)                ceph-node3-osd*
