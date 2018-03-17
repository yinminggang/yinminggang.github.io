---
layout:     post
title:      "gdbprof分析ceph的4K-randwrite性能"
subtitle:   "分析ceph进行4k随机写时哪些线程及其函数较为耗资源"
date:       2018-03-17 10:00:00
author:     "YMG"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 云存储
    - ceph
    - gdbprof
---

> gdbprof，是一款基于系统时间（而非cpu事件）的性能分析工具，它通过sampling的方式进行性能统计的。每隔period（默认为0.1秒），给gdb发送sigint信号，然后循环每个thread，根据该thread的call trace，统计函数的调用情况。
它是基于python实现的，中间用到了gdb的python api。<br>
>[gdbprof源码](https://github.com/markhpc/gdbprof)
>本文采用的[gdbprof优化版本](https://github.com/liupan1111/gdbprof)<br>
>[本人forked版本](https://github.com/yinminggang/gdbprof),随时欢迎


## 实验环境

搭建ceph-osd集群（本文用了3个osd.0、osd.1、osd.2）

服务端：server_host（osd）

客户端：client_host（mons、mgrs）

服务端需安装gdb，客户端需安装fio（都自行解决），下载[gdbprof](https://github.com/liupan1111/gdbprof)

## 实验过程

###（1）简单查看下集群是否正常：
**$ceph osd tree**
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/ceph_cluster_health1.png)

**$ceph -s**
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/ceph_cluster_heath2.png)

###（2）查看所有osd对应进程及进程号
**$ps aux|grep ceph-osd**
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/ceph_cluster_process.png)


##实验一 求解耗系统资源的线程（单image和10-images）

客户端（server_host）操作：

fio 创建pool（或默认rbd）和image，并进行后续操作

**$ rbd create --pool ymg --image img01 --size 40G**

先进行填充实验

**$fio -direct=1 -iodepth=256 -ioengine=rbd -pool=rbd -rbdname=img01 -rw=write -bs=1M -size=40G -ramp_time=5 -group_reporting -name=full-fill**

再进行4k-randWrite的单image操作

**$ fio -direct=1 -iodepth=256 -ioengine=rbd -pool=rbd -rbdname=img01 -rw=randwrite -bs=4K -runtime=300 -numjobs=1 -ramp_time=5 -group_reporting -name=one_gdbprof**

与此同时，服务端机器(client_host)需要并行执行top命令

**$top -p 7922 -H**

也即追踪进程7922(随意选一个)在过程中的所有线程对资源消耗情况，结果如下所示：
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/1_image_sys_load.png)


`对比实验`：10个images的4k-randWrite的实验操作：

**$ BS=4k RW=randwrite fio images.fio**

得先创建10个images

**$rbd create ymg/img00 -s 40G**

images.fio内容如下所示：
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/10_images_fio.png)

服务端执行 **$top -p 7922 -H**

结果如下所示
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/10_images_sys_load.png)

`总结`

由上面的一对儿实验可知：4k-randwrite过程中，比较消耗资源的线程有：`msgr-worker-0(7926)/msgr-worker-1(7927)、bstore_kv_sync(7972)、rocksdb::bg0(21792)、finisher(7971)、bstore_kv_final(7973)、log(7924)、tp_osd_tp(8109)`

`接下来，我们需要具体看这些线程中具体哪些函数最消耗时间。这就是gdbprof的亮相了。`


##实验二 求解耗系统资源的具体函数（单image）

在客户端进行4k-randWrite操作同时（fio命令），服务端要跑gdbprof工具（可下面参考原文中的.docx文件），切换到gdbprof目录，有gdbprof.py，

**$vim gdbprof.py**

line140，调节采样运行时间（无需太短或长），据说单位为秒，但感觉是采样次数，也就是样本个数，本人设过500，1000，2000，5000(等待时间太长)，如下所示：
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/gdbprof_config.png)

**$sudo gdb -ex 'set pagination off' -ex 'attach 7922' -ex 'source gdbprof.py' -ex 'profile begin' -ex 'quit'**

注：在运行过程中，可能会出现一些下面的debuginfo信息

**$yum install ceph-debuginfo.x86_64 --enablerepo didi_deph**
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/miss_debuginfo.png)

运行结束后就可以看到打印出来的结果，共有69个线程的信息

先找出打印出的线程函数信息与上面线程的对应关系：

**$gdb**

**(gdb) attach 7922**

**(gdb) info threads**

![](/img/2018-03-17-gdbprof-analysis-ceph-performance/threads_info1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/threads_info2.png)

接着，贴出6个较为耗资源的线程的结果：`Thread: 2===21792===rokcsdb::bg0、Thread: 28===8109===tp_osd_tp、Thread: 51===7973===bstore_kv_final、Thread: 52===7972===bstore_kv_sync、Thread: 53===7971===finisher、Thread: 68===7926===msgr_worker_0`，可以看出哪些函数较为消耗资源，如下所示：

具体文件可下载附件中文件1-image-important-threads.txt

由上面结果可以知道6个线程中哪些函数较为消耗资源，如下所示：
`（1）Thread: 2===21792===rokcsdb::bg0：`
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread2-1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread2-2.png)

`（2）Thread: 28===8109===tp_osd_tp：`
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread28-1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread28-2.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread28-3.png)

`（3）Thread: 51===7973===bstore_kv_final`
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread51-1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread51-2.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread51-3.png)

`（4）Thread: 52===7972===bstore_kv_sync`
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread52-1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread52-2.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread52-3.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread52-4.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread52-5.png)

`（5）Thread: 53===7971===finisher`
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread53-1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread53-2.png)

`（6）Thread: 68===7926===msgr_worker_0`
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread68-1.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread68-2.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread68-3.png)
![](/img/2018-03-17-gdbprof-analysis-ceph-performance/thread68-4.png)

**想得到所有69个线程的函数堆栈情况，可见附件中文件 1-image-all-threads.txt**


