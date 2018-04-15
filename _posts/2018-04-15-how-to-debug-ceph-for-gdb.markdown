---
layout:     post
title:      "使用gdb来调试ceph"
subtitle:   "在debug模式下重新编译ceph，再利用gdb来调试ceph的rbd"
date:       2018-04-15 15:00:00
author:     "YMG"
tags:
    - rbd
    - gdb
    - ceph编译
---
> 首先重新编译ceph，在debug模式下，使用gdb来调试rbd的运行过程

## 编译安装ceph
参照ceph的github[官方教程](https://github.com/ceph/ceph)<br>
**$git clone git://github.com/ceph/ceph**<br>
**$git submodule update --init --recursive**<br>
checkout到v12.2.2<br>
**$git checkout v12.2.2**<br>
开始安装依赖包<br>
**$cd ceph**<br>
**$./install-deps.sh**<br>
设置在debug模式下进行编译<br>
**$./do_cmake.sh -DCMAKE_EXPORT_COMPILE_COMMANDS=ON CMAKE_BUILD_TYPE="Debug" CMAKE_CXX_FLAGS_DEBUG="-g2 -ggdb" -DBUILD_SHARED_LIBS=OFF**[可参照](https://jiyou.github.io/blog/2016/10/24/ceph/ceph-cmake-debug/)<br>
或者**$vim CMakeLists.txt**<br>
添加两行<br>
**SET(CMAKE_BUILD_TYPE "DEBUG")
SET(CMAKE_C_FLAGS "-O0 -Wall -g")**如下：<br>
<center>
  <img src="/img/2018-04-15-how-to-debug-ceph-for-gdb/debug_config.png">
</center>
**$cd build**<br>
**$make -j8 && make install**
## gdb调试
具体gdb调试入门[详见](http://jqjiang.com/linux/gdb/)<br>
先读取rbd程序 **$gdb rbd**<br>
记载rbd的程序入口处，设置断点  **$b main**<br>
运行程序   **$r**<br>
一行一行的执行   **$r**<br>
结果如下图所示：
<center>
  <img src="/img/2018-04-15-how-to-debug-ceph-for-gdb/debug_run.png">
</center>
