---
layout: post
title: 定位某个文件被ceph存在哪里
category: monitor
tags: [原创,ceph,crush算法]
---
# 定位文件被ceph真正存在哪里

我们知道,将一个文件存到ceph里之后，ceph会将该文件条带化为若干个小的object，然后使用cursh算法，将每一个object复制若干份（根据pool size 来定）分别存储到不同的osd上。
本文会介绍，如何通过命令找到一个文件真正被存在哪里了。

## 集群的配置情况
### 集群osd树

    [root@sz-1 ~]# ceph osd tree
    ID  WEIGHT  TYPE NAME                                UP/DOWN REWEIGHT PRIMARY-AFFINITY 
    -20 1.34999 failure-domain sata-00-ssd                                                 
    -19 1.34999     replica-domain replica-0-ssd                                           
    -14 0.45000         rack rack-01-ssd                                                   
    -13 0.45000             host sz-1-ssd                                                  
      0 0.45000                 osd.0                         up  1.00000          1.00000 
    -16 0.45000         rack rack-02-ssd                                                   
    -15 0.45000             host sz-2-ssd                                                  
      3 0.45000                 osd.3                         up  1.00000          1.00000 
    -18 0.45000         rack rack-03-ssd                                                   
    -17 0.45000             host sz-3-ssd                                                  
      6 0.45000                 osd.6                         up  1.00000          1.00000 
    -10 1.34999 failure-domain sata-00                                                     
     -9 1.34999     replica-domain replica-0                                               
     -8 0.45000         host-domain host-group-0-rack-01                                   
     -2 0.45000             host sz-1                                                      
      1 0.14999                 osd.1                         up  1.00000          1.00000 
      2 0.14999                 osd.2                         up  1.00000          1.00000 
    -11 0.45000         host-domain host-group-0-rack-02                                   
     -4 0.45000             host sz-2                                                      
      4 0.14999                 osd.4                         up  1.00000          1.00000 
      5 0.14999                 osd.5                         up  1.00000          1.00000 
    -12 0.45000         host-domain host-group-0-rack-03                                   
     -6 0.45000             host sz-3                                                      
      7 0.14999                 osd.7                         up  1.00000          1.00000 
      8 0.14999                 osd.8                         up  1.00000          1.00000 
     -1 1.34999 root default                                                               
     -3 0.45000     rack rack-01                                                           
     -2 0.45000         host sz-1                                                          
      1 0.14999             osd.1                             up  1.00000          1.00000 
      2 0.14999             osd.2                             up  1.00000          1.00000 
     -5 0.45000     rack rack-02                                                           
     -4 0.45000         host sz-2                                                          
      4 0.14999             osd.4                             up  1.00000          1.00000 
      5 0.14999             osd.5                             up  1.00000          1.00000 
     -7 0.45000     rack rack-03                                                           
     -6 0.45000         host sz-3                                                          
      7 0.14999             osd.7                             up  1.00000          1.00000 
      8 0.14999             osd.8                             up  1.00000          1.00000 

本测试集群共3台hosts，每台host上平均有3个osds。

### pool size
    [root@sz-1 ~]# ceph osd dump | grep pool
    pool 18 '.rgw' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 194 flags hashpspool stripe_width 0
    pool 19 '.rgw.root' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 16 pgp_num 16 last_change 195 flags hashpspool stripe_width 0
    pool 20 '.rgw.control' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 16 pgp_num 16 last_change 196 flags hashpspool stripe_width 0
    pool 21 '.rgw.gc' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 197 flags hashpspool stripe_width 0
    pool 22 '.rgw.buckets' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 128 pgp_num 128 last_change 305 flags hashpspool stripe_width 0
    pool 23 '.rgw.buckets.index' replicated size 2 min_size 2 crush_ruleset 6 object_hash rjenkins pg_num 64 pgp_num 64 last_change 206 flags hashpspool stripe_width 0
    pool 24 '.log' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 198 flags hashpspool stripe_width 0
    pool 25 '.intent-log' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 199 flags hashpspool stripe_width 0
    pool 26 '.usage' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 200 flags hashpspool stripe_width 0
    pool 27 '.users' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 201 flags hashpspool stripe_width 0
    pool 28 '.users.email' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 202 flags hashpspool stripe_width 0
    pool 29 '.users.swift' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 203 flags hashpspool stripe_width 0
    pool 30 '.users.uid' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 204 flags hashpspool stripe_width 0
    pool 31 '.rgw.buckets.extra' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 32 pgp_num 32 last_change 207 flags hashpspool stripe_width 0
    pool 32 '.rgw.buckets.ssd' replicated size 2 min_size 2 crush_ruleset 6 object_hash rjenkins pg_num 64 pgp_num 64 last_change 209 flags hashpspool stripe_width 0

本集群为测试集群，pool_size 值为 3。
### 查看ceph存储系统中条带化对象尺寸：

    [root@sz-1 ~]# ceph --show-config | grep rgw_obj_stripe_size
    rgw_obj_stripe_size = 4194304
    
本集群条带化尺寸为 4MB。


## 上传文件
我们通过[美的云 OSS Web 管理控制台](http://mconsole.midea.net) 上传一个大小为 15,771,063 字节，名为 `cmake-3.6.2-win64-x64.msi` 的文件。
![](../../assets/images/upload-file-by-oss.jpg)

*当然，你也可以通过其它工具上传，比如 S3 Browser 等*

## 找出刚才上传的文件被ceph存到什么地方去了

1, 查看bucket `wucq-test` 中的对象列表，确认文件是否已存到ceph集群里

    [root@sz-1 ~]# radosgw-admin bucket list --bucket=wucq-test
    [
        {
            "name": "cmake-3.6.2-win64-x64.msi",
            "instance": "",
            "namespace": "",
            "owner": "965DE31419464D8C92C20907668C9CE0",
            "owner_display_name": "app_965DE31419464D8C92C20907668C9CE0",
            "size": 15771063,
            "mtime": "2017-06-16 06:05:23.000000Z",
            "etag": "ef8f09c0e85fad29a1a62b339b992471-11",
            "content_type": "binary\/octet-stream",
            "tag": "default.19821.5258",
            "flags": 0
        }
    
    ]


2, 查看该文件在rados集群中的分布
在ceph的对象存储中，对象数据是保存在池 .rgw.buckets 中的，我们来查看一下刚上传的文件在ceph存储集群中的对象名。

    [root@sz-1 ~]# rados -p .rgw.buckets ls | grep  cmake-3.6.2-win64-x64.msi
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.10
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.8
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.4
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.7
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.6
    default.21039.18_cmake-3.6.2-win64-x64.msi
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.9
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.1
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.11
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.5
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.3
    default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.2

一共列出了12 个相关对象，其中有一个是元数据，其它的是刚刚上传的文件 `cmake-3.6.2-win64-x64.msi` 被条带化后产生的子object，子object的名称后面有它的序号。也就是说，`cmake-3.6.2-win64-x64.msi` 被条带化为了 11 个子object。结合本集群的条带化尺寸为 4MB，文件大小为 15,771,063 字节，
`15771063 / 4194304 = 3.76` ，也就是需要 4 个子object。为什么和实际的 11 个子object不符呢？
因为我们的Web控制台在上传的时候，使用了分片上传，每个分片的大小是1.5MB，`15771063 / (1.5 * 1024 * 1024) = 10.0269718170166015625`，也就是说需要11个子object，正好与实际相符。

3, 接下来，逐个找出上面 11 个子object被 ceph 存在哪里，然后将它们的 size 相加， 是否和原始文件的大小相等。
下面以第一个子object `default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.1` 为例，讲解如何定位 object 的位置。

    [root@sz-1 ~]# ceph osd map .rgw.buckets default.21039.18__multipart_fm727chb174_build_setup.exe.2~9OQ6y6sBS3qGXujULlYDx5sAQy9h71h.1
    osdmap e337 pool '.rgw.buckets' (22) object 'default.21039.18__multipart_fm727chb174_build_setup.exe.2~9OQ6y6sBS3qGXujULlYDx5sAQy9h71h.1' -> pg 22.a492dfd8 (22.58) -> up ([4,7], p4) acting ([4,7], p4)

可见，子object `default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.1` 分布在归置组 `22.58` 上面，在 `osd.4, osd.7` 上面各有一份副本以保证数据的安全性。

4，查看子object 的 size
由文章开头的 osd tree 可以看到，`osd.4` 在 `host sz-2` 上面，我们 SSH 登录到 `host sz-2`，并进入 `osd.4` 的数据目录
 
     ssh sz-2
     cd /var/lib/ceph/osd/ceph-4/current/
     
 然后，找出归置组 `22.58` 的数据目录
 
    [root@sz-2 current]# ll | grep 22.58   
    drwxr-xr-x 3 root root 12288 Jun  3 13:53 22.58_head
    drwxr-xr-x 2 root root     6 Jun  3 13:49 22.58_TEMP
    [root@sz-2 current]# cd 22.58_head

找出子object在 `osd.4`的文件系统上的真实文件

    [root@sz-2 DIR_D]# ll | grep fm727
    -rw-r--r-- 1 root root 1572864 Jun 16 14:46 default.21039.18\u\umultipart\ucmake\u3.6.2\uwin64\ux64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.1__head_A492DFD8__16
    
可见，子object `default.21039.18__multipart_cmake-3.6.2-win64-x64.msi.2~kcu7j-Dfaenre3_ZPzygH0iLfjla1qR.1` 的 size 为 1572864 字节。