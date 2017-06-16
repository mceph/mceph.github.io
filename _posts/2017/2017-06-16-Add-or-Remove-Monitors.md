---
layout: post
title: ceph monitors管理
category: monitor
tags: [原创,ceph,monitor,监控器]
---

# 添加或移除monitors
本文参照[官网操作手册](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/)

## 预知条件
### 集群概况
    [root@sz-3 ceph]# ceph osd tree
    ID  WEIGHT  TYPE NAME                                UP/DOWN REWEIGHT PRIMARY-AFFINITY 
    -10 0.89999 failure-domain sata-00                                                     
     -9 0.89999     replica-domain replica-0                                               
     -8 0.29999         host-domain host-group-0-rack-01                                   
      0 0.09999             osd.0                             up  1.00000          1.00000 
      1 0.09999             osd.1                             up  1.00000          1.00000 
      2 0.09999             osd.2                             up  1.00000          1.00000 
    -11 0.29999         host-domain host-group-0-rack-02                                   
      3 0.09999             osd.3                             up  1.00000          1.00000 
      4 0.09999             osd.4                             up  1.00000          1.00000 
      5 0.09999             osd.5                             up  1.00000          1.00000 
    -12 0.29999         host-domain host-group-0-rack-03                                   
      6 0.09999             osd.6                             up  1.00000          1.00000 
      7 0.09999             osd.7                             up  1.00000          1.00000 
      8 0.09999             osd.8                             up  1.00000          1.00000 
     -1 0.89996 root default                                                               
     -3 0.29999     rack rack-01                                                           
     -2 0.29999         host sz-1                                                          
      0 0.09999             osd.0                             up  1.00000          1.00000 
      1 0.09999             osd.1                             up  1.00000          1.00000 
      2 0.09999             osd.2                             up  1.00000          1.00000 
     -5 0.29999     rack rack-02                                                           
     -4 0.29999         host sz-2                                                          
      3 0.09999             osd.3                             up  1.00000          1.00000 
      4 0.09999             osd.4                             up  1.00000          1.00000 
      5 0.09999             osd.5                             up  1.00000          1.00000 
     -7 0.29999     rack rack-03                                                           
     -6 0.29999         host sz-3                                                          
      6 0.09999             osd.6                             up  1.00000          1.00000 
      7 0.09999             osd.7                             up  1.00000          1.00000 
      8 0.09999             osd.8                             up  1.00000          1.00000 

由上可见：本集群由 3 个 hosts 组成，每个 host 上有 3 个 osd，共有 9 个 osds。

    [root@sz-3 ceph]# ceph -s
        cluster 1fbf02e7-7117-499b-b3cb-389c908242ae
         health HEALTH_OK
         monmap e3: 3 mons at {sz-1=10.133.134.241:6789/0,sz-2=10.133.134.242:6789/0,sz-3=10.133.134.243:6789/0}
                election epoch 10, quorum 0,1,2 sz-1,sz-2,sz-3
         osdmap e57: 9 osds: 9 up, 9 in
          pgmap v136: 408 pgs, 14 pools, 0 bytes data, 0 objects
                313 MB used, 899 GB / 899 GB avail
                     408 active+clean

    [root@sz-3 ceph]# ceph mon dump
    dumped monmap epoch 3
    epoch 3
    fsid 1fbf02e7-7117-499b-b3cb-389c908242ae
    last_changed 2017-06-10 12:45:48.871531
    created 2017-06-10 11:38:03.196203
    0: 10.133.134.241:6789/0 mon.sz-1
    1: 10.133.134.242:6789/0 mon.sz-2
    2: 10.133.134.243:6789/0 mon.sz-3

由上可见：本集群共有 3 个 monitor。且该集群处于 `HEALTH_OK` 状态


## 添加mon（手动）

添加monitor之前，要保证网络端口的连通性，即每个monitor都要能够：
* 与其它 monitors 互联
* 被所有的 osd 连接

1, 为新monitor创建默认数据目录，{mon-id}一般是 `hostname`，但也可以是其它的任意字符串，只要保证不和其它的monitors的id相同就行


    sudo mkdir -p /var/lib/ceph/mon/ceph-{mon-id}
    

2, 创建一个临时目录，以保存在后续步骤中产生的文件。这个目录不能是上一步创建的monitor数据目录，且在完成所有操作后能被删除。


    mkdir -p {tmp}
    
3,  为新 monitor 获取 `keyring`，其中 `{tmp}` 是上一步创建的临时目录，`{key-filename}` 是保存获取到的 `keyring`的文件名


    ceph auth get mon. -o {tmp}/{key-filename}
    
4, 获取 monitor map，其中 `{tmp}` 是上一步创建的临时目录，`{map-filename}` 是保存 `monmap` 的文件名


    ceph mon getmap -o {tmp}/{map-filename}
    
5, 初始化第一步中创建的数据目录。你必须指定 `monmap`文件路径，以便得到集群监控器的法定人数和 `fsid`。同时也必须指定 monitor keyring 文件的路径。


    sudo ceph-mon -i {mon-id} --mkfs --monmap {tmp}/{map-filename} --keyring {tmp}/{key-filename}
    
6, 将新 monitor 添加到集群的监控器列表中（运行时添加），这使得其它节点在初次启动时使用这个monitor。


    ceph mon add {mon-id} <ip>[:<port>]

7, 启动新的 monitor， 它会自动加入集群。monitor 进程需要知道绑定到哪个地址，既可以通过 `--public-addr {ip:port}`（推荐方法）， 也可以通过 `ceph.conf` 中的相应配置来实现（实测不行）。例如：


    ceph-mon -i {mon-id} --public-addr {ip:port}

8, 修改`ceph.conf`，将新的monitor的配置信息添加进去，以便让其它的组件得知新的monitor信息


    [mon.{mon-id}]
    host = {mon-id}
    addr = {ip:port}

## 移除mon
现在准备移除 `mon.sz-3` 这个 monitor。

### 从“健康”状态的集群中移除monitor

1, 停止monitor


    [root@sz-3 ceph]# service ceph -a stop mon.sz-3
    === mon.sz-3 === 
    Stopping Ceph mon.sz-3 on sz-3...kill 11884...done111

        
2, 从集群中移除monitor


    [root@sz-3 ceph]# ceph mon remove sz-3    
    2017-06-14 17:40:07.570951 7f45db885700  0 -- :/1167820720 >> 10.133.134.243:6789/0 pipe(0x7f45e006e550 sd=4 :0 s=1 pgs=0 cs=0 l=1 c=0x7f45e00632f0).fault
    removed mon.sz-3 at 10.133.134.243:6789/0, there are now 2 monitors
    

3, 从 `ceph.conf` 中删除被移除的 monitor 的 `host` 和 IP 地址。主要包括：


    [global]
    mon_initial_members=
    mon_host=
    [mon.{mon-id}]
    host = {mon-id}
    addr = {ip:port}
    


4, 归档备份或删除被移除的 monitor 的数据目录 `/var/lib/ceph/mon/ceph-{mon-id}`
    
5, 查看集群状态


    [root@sz-3 ceph]# ceph -s
        cluster 1fbf02e7-7117-499b-b3cb-389c908242ae
         health HEALTH_OK
         monmap e4: 2 mons at {sz-1=10.133.134.241:6789/0,sz-2=10.133.134.242:6789/0}
                election epoch 18, quorum 0,1 sz-1,sz-2
         osdmap e57: 9 osds: 9 up, 9 in
          pgmap v136: 408 pgs, 14 pools, 0 bytes data, 0 objects
                313 MB used, 899 GB / 899 GB avail
                     408 active+clean




至此，可见 `mon.sz-3` 已经被从集群中移除

### 从“不健康”状态的集群中移除monitor

如果一个集群处于“不健康”状态（比如monitor的数量不足以组成法定人数（quorum）），从这样的集群移除monitor的流程如下：

1, 停止所有 host 上的所有 monitor

    service ceph stop mon || stop ceph-mon-all

2, 识别出一个“健康”的monitor，ssh登录到它所在的 host

3, 提取出 monmap 文件


    ceph-mon -i {mon-id} --extract-monmap {map-path}
    # in most cases, that's
    ceph-mon -i `hostname` --extract-monmap /tmp/monmap
    
4, 移除非健康的或有问题的monitor。例如，如果你有3个monitor：`mon.a`,`mon.b`,`mon.c`,其中 `mon.a` 是健康的，按照下面的步骤操作：


    monmaptool {map-path} --rm {mon-id}
    # for example,
    monmaptool /tmp/monmap --rm b
    monmaptool /tmp/monmap --rm c
    
5, 将“健康”的monitors注入到 monmap。例如，将 `mon.a` 注入到 monmap，例如：


    ceph-mon -i {mon-id} --inject-monmap {map-path}
    # for example,
    ceph-mon -i a --inject-monmap /tmp/monmap

6, 启动“健康”的monitor
7, 验证monitor已经组成了法定人数（`ceph -s`）
8, 你可能希望归档被删除的monitor的数据目录 `/var/lib/ceph/mon`以保存到一个安全的地方，或者，如果你确信剩下的monitor是健康的且是冗余足够的，也可以删除数据目录


## 更改 monitor 的 ip 地址
**重要**：不应该改变已经存在的 monitors 的 ip 地址

在 ceph 集群中 monitors 是非常挑剔的组件，它们要正常工作，必须组成法定人数。为了组成法定人数，它们必须能够发现彼此。要发现彼此，ceph有严格的条件。

ceph客户端和其它的ceph组件使用 `ceph.conf` 去发现monitors。然而，monitors 发现彼此使用的是 `monitor map`，而不是 `ceph.conf`。例如，如果你参考[添加monitor](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/#adding-a-monitor-manual)，你会发现当你创建一个新的monitor时，你需要获取当前集群的 monmap，由于它是命令 `ceph-mon -i {mon-id} --mkfs`的一个参数。下一节介绍monitor达到一致性的条件，以及安全地更改monitor的 IP 地址的方法。

### 一致性的条件
monitor总是使用 monmap 的本地拷贝去发现集群中的其它monitor。使用 monmap 而非 ceph.conf 可以避免可能损坏集群的错误（例如在 ceph.conf 中指定monitor的地址或端口时打字错误）。由于monitors使用monmap来实现相互发现，并会与ceph客户端和其它ceph组件共享 monmap，monmap为monitor的一致性提供严格的保证。

更新monmap同样需要严格的一致性。正如monitor的其它更新，改变monmap同样使用称为 [Paxos](http://en.wikipedia.org/wiki/Paxos_(computer_science))的一致性算法。monitor必须对 monmap 的每个更新达成一致，如添加或移除一个monitor，以确保法定人数中的每一个monitor都有相同版本的monmap。对monmap的更新是增量的，所以 monitors 有最新达成一致的版本，和一系列旧版本，并允许拥有旧版本 monmap 的 monitor 能根据集群的当前状态追赶上来。

如果monitors使用 ceph 配置文件而非 monmap 来发现彼此，会引进其它的风险，因为 ceph 配置文件并不会自动自动更新和扩散。monitors可能由于失误而使用了旧版本的 ceph.conf，不能识别monitor，退出法定法定人数，或者出现 Paxos 算法不能精确决定系统状态的情景。因此，修改已存在的 monitors 的IP地址，必须加倍小心。

### 更改 monitor 的 IP 地址（正确的方法）
仅仅在 ceph.conf 中修改 monitor 的 IP 不足以确保集群中的其它 monitor 能接收到该更新。为了改变 monitor 的 IP 地址，你必须增加一个新的、使用你想使用的 IP 地址的 monitor，确保新的 monitor 成功地加入法定人数；然后，移除使用旧的 IP 地址的 monitor。然后，更新 ceph.conf 以确保 ceph 客户端和其它的组件知道新的 monitor 的 IP 地址。

例如，我们假设有3个 monitors ，例如：


    [mon.a]
        host = host01
        addr = 10.0.0.1:6789
    [mon.b]
            host = host02
            addr = 10.0.0.2:6789
    [mon.c]
            host = host03
            addr = 10.0.0.3:6789
            
为了将 `mon.c` 的 `host` 改为 `host04`，IP 地址改为 `10.0.0.4:6789`，需要首先按照[添加monitor](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/#adding-a-monitor-manual)中的步骤添加一个新的monitor `mon.d`，在删除 `mon.c` 之前，确保 `mon.d` 能正常运行，否则将破坏监控器的法定人数。然后参照 [移除monitor](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/#removing-a-monitor-manual) 中描述的步骤将 `mon.c` 移除。如果要更改所有 monitors 的 IP 地址，就需要根据需要重复上述步骤。
