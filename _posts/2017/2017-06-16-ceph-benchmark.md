---
layout: post
title: ceph性能测试
category: monitor
tags: [原创,ceph,性能测试]
---

搭建好ceph集群后的第一件事，往往是做一下性能测试，本文讲的性能测试主要包括：
1. 集群内网络IO
2. ceph节点的磁盘IO
3. ceph集群的读写性能

# ceph性能测试

## 测试磁盘性能
### 找到 osd 的挂载盘

    [root@oss-90 ~]# lsblk -f
    NAME              FSTYPE      LABEL UUID                                   MOUNTPOINT
    sda                                                                        
    ├─sda1                                                                     
    └─sda2            xfs               31906d54-af7f-4505-863e-40d743f36d95   /var/lib/ceph/osd/ceph-0
    sdb                                                                        
    ├─sdb1                                                                     
    └─sdb2            xfs               e686a0ed-b4ed-4265-92fb-68dba1515974   /var/lib/ceph/osd/ceph-1
    sdc                                                                        
    ├─sdc1                                                                     
    └─sdc2            xfs               f53d8f0b-95d2-4170-bd12-721c646ab66a   /var/lib/ceph/osd/ceph-2
    sdd                                                                        
    ├─sdd1                                                                     
    └─sdd2            xfs               df4d3a93-d52d-46af-982a-db92494b16a1   /var/lib/ceph/osd/ceph-3
    sde                                                                        
    ├─sde1            xfs               fd698f9d-74b5-4c7b-8373-95a16eb45f9e   /boot
    ├─sde2            swap              eb10629a-9652-4b80-93ba-8760b89adbc0   [SWAP]
    └─sde3            LVM2_member       JXO8Rs-FIE3-20Nk-5vt4-t3n8-m702-ligOd7 
      ├─midea-lv_root xfs               ac87b4d9-a0c0-4a57-9fc7-345bb321dacc   /
      └─midea-lv_var  xfs               dff642ea-f5f6-41f8-a042-260c549322a5   /var
      
可见，/var/lib/ceph/osd/ceph-0 是其中一个 OSD 的挂载盘。

### 测试磁盘写性能
#### 测试磁盘写吞吐量

    [root@oss-90 ~]# cd /var/lib/ceph/osd/ceph-0
    [root@oss-90 ceph-0]# dd if=/dev/zero of=here bs=1G count=1 oflag=direct
    1+0 records in
    1+0 records out
    1073741824 bytes (1.1 GB) copied, 6.23866 s, 172 MB/s

磁盘吞吐量在 170 MB/s 左右
 
### 测试磁盘写请求数

    [root@oss-90 ceph-0]# dd if=/dev/zero of=512 bs=512 count=10000 oflag=direct
    10000+0 records in
    10000+0 records out
    5120000 bytes (5.1 MB) copied, 85.0074 s, 60.2 kB/s
    
每次写512字节，连续写1万次，共耗时85秒，每次耗时 8.5 ms。

按照上述步骤，对其它所有磁盘都做一下性能测试（本集群使用的磁盘品牌配置都一样）。


## 测试网络带宽
### 测试集群内各节点间的网络IO
*条件：* 测试集群共两个物理节点，集群内网络为 10.3.8.0/24。

下面测试该网络的IO性能：

在其中一个节点执行：

    [root@oss-90 ~]# nc -v -l -n 17480 > /dev/null
    Ncat: Version 6.40 ( http://nmap.org/ncat )
    Ncat: Listening on :::17480
    Ncat: Listening on 0.0.0.0:17480
    
    Ncat: Connection from 10.3.8.92.
    Ncat: Connection from 10.3.8.92:47529.
    
在另一个节点执行（运行一段时间后，``Ctrl-C`` 终止运行）：

    [root@oss-92 ~]# time dd if=/dev/zero | nc -v -n 10.3.8.90 17480  
    Ncat: Version 6.40 ( http://nmap.org/ncat )
    Ncat: Connected to 10.3.8.90:17480.
    
    ^C62730065+0 records in
    62730064+0 records out
    32117792768 bytes (32 GB) copied, 273.049 s, 118 MB/s
    
    
    real    4m33.051s
    user    0m30.261s
    sys     1m56.989s

可见是集群内不同节点间，网络IO 118MB/s。跟实际情况相符，因为本集群是千兆网卡。


### 测试外部到rgw的网络IO
略


## 测试rados集群的性能

1. 创建测试用存储池


    [root@oss-90 ceph-0]#  ceph osd pool create benchmark 4 4
    pool 'benchmark' created
    [root@oss-90 ceph-0]# ceph osd pool set benchmark crush_ruleset 5
    set pool 16 crush_ruleset to 5
    [root@oss-90 ceph-0]# ceph osd pool set benchmark size 2
    set pool 16 size to 2
    [root@oss-90 ceph-0]# ceph osd pool set benchmark min_size 2
    set pool 16 min_size to 2
    [root@oss-90 ceph-0]# ceph osd dump | grep benchmark
    pool 16 'benchmark' replicated size 2 min_size 2 crush_ruleset 5 object_hash rjenkins pg_num 4 pgp_num 4 last_change 107 flags hashpspool stripe_width 0
    
    
2. 查看该池占用的资源


    [root@oss-90 ceph-0]# rados -p benchmark df
    pool name                 KB      objects       clones     degraded      unfound           rd        rd KB           wr        wr KB
    benchmark                  0            0            0            0           0            0            0            0            0
      total used        43315524           44
      total avail    31087251132
      total space    31130566656
      
      
3. 测试写性能


    rados bench -p benchmark 60  write  [ --no-cleanup ]
    
     Total time run:         60.846884
    Total writes made:      1445
    Write size:             4194304
    Bandwidth (MB/sec):     94.993 
    
    Stddev Bandwidth:       40.1555
    Max bandwidth (MB/sec): 180
    Min bandwidth (MB/sec): 0
    Average Latency:        0.672951
    Stddev Latency:         0.482795
    Max latency:            2.77989
    Min latency:            0.0718336
    
    
如果加上可选参数 ``--no-cleanup`` ，那么测试完之后，不会删除该池里面的数据。里面的数据可以继续用于测试集群的读性能。

4. 测试读性能


    rados bench -p benchmark 60 [ seq | rand ]
     Total time run:        37.261000
    Total reads made:     1420
    Read size:            4194304
    Bandwidth (MB/sec):    152.438 
    
    Average Latency:       0.418494
    Max latency:           1.39918
    Min latency:           0.00530366


5. 测试完，删除池里面的内容


    rados -p benchmark cleanup


6. 最后删除池


    ceph osd pool delete benchmark benchmark --yes-i-really-really-mean-it
    
    
