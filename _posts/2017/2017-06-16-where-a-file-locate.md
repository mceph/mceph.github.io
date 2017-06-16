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
    
本集群条带化尺寸为 4M。


## 上传文件
我们通过[美的云 OSS Web 管理控制台](http://mconsole.midea.net) 上传一个大小为 15,771,063 字节，名为 `cmake-3.6.2-win64-x64.msi` 的文件。
![](../../assets/images/upload-file-by-oss.jpg)












