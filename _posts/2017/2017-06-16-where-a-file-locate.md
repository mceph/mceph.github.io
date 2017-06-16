---
layout: post
title: 定位某个文件被ceph存在哪里
category: monitor
tags: [原创,ceph,crush算法]
---

我们知道,将一个文件存到ceph里之后，ceph会将该文件条带化为若干个小的object，然后使用cursh算法，将每一个object复制若干份（根据pool size 来定）分别存储到不同的osd上。
本文会介绍，如何通过命令找到一个文件真正被存在哪里了。