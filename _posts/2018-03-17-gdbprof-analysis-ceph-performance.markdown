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

接着，贴出6个较为耗资源的线程的结果：`Thread: 2===21792===rokcsdb::bg0、Thread: 28===8109===tp_osd_tp、Thread: 51===7973===bstore_kv_final、Thread: 52===7972===bstore_kv_sync、Thread: 53===7971===finisher、Thread: 68===7926===msgr_worker_0`，可以看出哪些函数较为消耗资源，由于结果较长，部分如下所示：
```
{
  Thread: 2===21792===rokcsdb::bg0

+ 100.00% clone
  + 100.00% start_thread
    + 100.00% None
      + 100.00% rocksdb::ThreadPoolImpl::Impl::BGThreadWrapper
        + 100.00% rocksdb::ThreadPoolImpl::Impl::BGThread
          + 69.00% rocksdb::DBImpl::BackgroundCallCompaction
          | + 69.00% rocksdb::DBImpl::BackgroundCompaction
          |   + 69.00% rocksdb::CompactionJob::Run
          |     + 69.00% rocksdb::CompactionJob::ProcessKeyValueCompaction
          |       + 56.80% rocksdb::CompactionIterator::Next
          |       | + 42.20% rocksdb::MergingIterator::Next
          |       | | + 42.00% Next
          |       | | | + 25.00% b::(anonymous namespace)::TwoLevelIterator::Next
          |       | | | | + 25.00% Next
          |       | | | |   + 24.40% b::(anonymous namespace)::TwoLevelIterator::SkipEmptyDataBlocksForward
          |       | | | |   | + 24.40% b::(anonymous namespace)::TwoLevelIterator::InitDataBlock
          |       | | | |   |   + 24.20% rocksdb::BlockBasedTable::BlockEntryIteratorState::NewSecondaryIterator
          |       | | | |   |   | + 24.20% rocksdb::BlockBasedTable::NewDataBlockIterator
          |       | | | |   |   |   + 23.60% b::(anonymous namespace)::ReadBlockFromFile
          |       | | | |   |   |   | + 23.40% rocksdb::ReadBlockContents
          |       | | | |   |   |   | | + 23.40% ReadBlock
          |       | | | |   |   |   | |   + 22.40% rocksdb::RandomAccessFileReader::Read
          |       | | | |   |   |   | |   | + 22.40% b::(anonymous namespace)::ReadaheadRandomAccessFile::Read
          |       | | | |   |   |   | |   |   + 22.40% ReadIntoBuffer
          |       | | | |   |   |   | |   |     + 22.40% BlueRocksRandomAccessFile::Read
          |       | | | |   |   |   | |   |       + 22.40% read_random
          |       | | | |   |   |   | |   |         + 22.40% BlueFS::_read_random
          |       | | | |   |   |   | |   |           + 22.40% KernelDevice::read_random
          |       | | | |   |   |   | |   |             + 22.40% pread
          |       | | | |   |   |   | |   |               + 22.40% pread64
          |       | | | |   |   |   | |   + 1.00% Value
          |       | | | |   |   |   | |     + 1.00% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       | | | |   |   |   | |       + 1.00% Fast_CRC32
          |       | | | |   |   |   | |         + 1.00% Slow_CRC32
          |       | | | |   |   |   | + 0.20% rocksdb::Block::Block(rocksdb::BlockContents&&, unsigned long, unsigned long, rocksdb::Statistics*)
          |       | | | |   |   |   |   + 0.20% BlockContents
          |       | | | |   |   |   |     + 0.20% operator=
          |       | | | |   |   |   + 0.40% rocksdb::BlockBasedTable::MaybeLoadDataBlockToCache
          |       | | | |   |   |     + 0.40% rocksdb::BlockBasedTable::GetDataBlockFromCache
          |       | | | |   |   |       + 0.40% b::(anonymous namespace)::GetEntryFromCache
          |       | | | |   |   |         + 0.40% rocksdb::ShardedCache::Lookup
          |       | | | |   |   |           + 0.20% rocksdb::LRUCache::GetShard
          |       | | | |   |   + 0.20% b::(anonymous namespace)::TwoLevelIterator::SetSecondLevelIterator
          |       | | | |   + 0.40% rocksdb::BlockIter::ParseNextKey
          |       | | | |   | + 0.20% TrimAppend
          |       | | | |   | | + 0.20% EnlargeBufferIfNeeded
          |       | | | |   | |   + 0.20% tc_newarray
          |       | | | |   | + 0.20% DecodeEntry
          |       | | | |   + 0.20% b::(anonymous namespace)::TwoLevelIterator::Next
          |       | | | |     + 0.20% Next
          |       | | | + 17.00% b::(anonymous namespace)::TwoLevelIterator::SkipEmptyDataBlocksForward
          |       | | |   + 17.00% b::(anonymous namespace)::TwoLevelIterator::InitDataBlock
          |       | | |     + 17.00% rocksdb::BlockBasedTable::BlockEntryIteratorState::NewSecondaryIterator
          |       | | |       + 17.00% rocksdb::BlockBasedTable::NewDataBlockIterator
          |       | | |         + 16.80% b::(anonymous namespace)::ReadBlockFromFile
          |       | | |           + 16.80% rocksdb::ReadBlockContents
          |       | | |             + 16.80% ReadBlock
          |       | | |               + 16.60% rocksdb::RandomAccessFileReader::Read
          |       | | |               | + 16.60% b::(anonymous namespace)::ReadaheadRandomAccessFile::Read
          |       | | |               |   + 16.40% ReadIntoBuffer
          |       | | |               |   | + 16.40% BlueRocksRandomAccessFile::Read
          |       | | |               |   |   + 16.40% read_random
          |       | | |               |   |     + 16.40% BlueFS::_read_random
          |       | | |               |   |       + 16.40% KernelDevice::read_random
          |       | | |               |   |         + 16.40% pread
          |       | | |               |   |           + 16.40% pread64
          |       | | |               |   + 0.20% TryReadFromCache
          |       | | |               |     + 0.20% memcpy
          |       | | |               |       + 0.20% __memcpy_ssse3_back
          |       | | |               + 0.20% Value
          |       | | |                 + 0.20% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       | | |                   + 0.20% Fast_CRC32
          |       | | |                     + 0.20% Slow_CRC32
          |       | | + 0.20% replace_top
          |       | |   + 0.20% downheap
          |       | |     + 0.20% operator()
          |       | + 14.60% rocksdb::CompactionIterator::NextFromInput
          |       |   + 13.20% rocksdb::MergingIterator::Next
          |       |   | + 12.80% Next
          |       |   | | + 9.40% b::(anonymous namespace)::TwoLevelIterator::Next
          |       |   | | | + 9.40% Next
          |       |   | | |   + 9.20% b::(anonymous namespace)::TwoLevelIterator::SkipEmptyDataBlocksForward
          |       |   | | |   | + 9.20% b::(anonymous namespace)::TwoLevelIterator::InitDataBlock
          |       |   | | |   |   + 9.20% rocksdb::BlockBasedTable::BlockEntryIteratorState::NewSecondaryIterator
          |       |   | | |   |     + 9.20% rocksdb::BlockBasedTable::NewDataBlockIterator
          |       |   | | |   |       + 9.20% b::(anonymous namespace)::ReadBlockFromFile
          |       |   | | |   |         + 9.20% rocksdb::ReadBlockContents
          |       |   | | |   |           + 9.20% ReadBlock
          |       |   | | |   |             + 8.40% rocksdb::RandomAccessFileReader::Read
          |       |   | | |   |             | + 8.40% b::(anonymous namespace)::ReadaheadRandomAccessFile::Read
          |       |   | | |   |             |   + 8.20% ReadIntoBuffer
          |       |   | | |   |             |   | + 8.20% BlueRocksRandomAccessFile::Read
          |       |   | | |   |             |   |   + 8.20% read_random
          |       |   | | |   |             |   |     + 8.20% BlueFS::_read_random
          |       |   | | |   |             |   |       + 8.20% KernelDevice::read_random
          |       |   | | |   |             |   |         + 8.20% pread
          |       |   | | |   |             |   |           + 8.20% pread64
          |       |   | | |   |             |   + 0.20% memcpy
          |       |   | | |   |             |     + 0.20% __memcpy_ssse3_back
          |       |   | | |   |             + 0.80% Value
          |       |   | | |   |               + 0.80% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       |   | | |   |                 + 0.80% Fast_CRC32
          |       |   | | |   |                   + 0.80% Slow_CRC32
          |       |   | | |   + 0.20% b::(anonymous namespace)::TwoLevelIterator::Next
          |       |   | | |     + 0.20% Next
          |       |   | | |       + 0.20% rocksdb::BlockIter::ParseNextKey
          |       |   | | |         + 0.20% TrimAppend
          |       |   | | |           + 0.20% memcpy
          |       |   | | |             + 0.20% __memcpy_ssse3_back
          |       |   | | + 3.40% b::(anonymous namespace)::TwoLevelIterator::SkipEmptyDataBlocksForward
          |       |   | |   + 3.40% b::(anonymous namespace)::TwoLevelIterator::InitDataBlock
          |       |   | |     + 3.40% rocksdb::BlockBasedTable::BlockEntryIteratorState::NewSecondaryIterator
          |       |   | |       + 3.40% rocksdb::BlockBasedTable::NewDataBlockIterator
          |       |   | |         + 3.40% b::(anonymous namespace)::ReadBlockFromFile
          |       |   | |           + 3.40% rocksdb::ReadBlockContents
          |       |   | |             + 3.40% ReadBlock
          |       |   | |               + 3.20% rocksdb::RandomAccessFileReader::Read
          |       |   | |               | + 3.20% b::(anonymous namespace)::ReadaheadRandomAccessFile::Read
          |       |   | |               |   + 3.20% ReadIntoBuffer
          |       |   | |               |     + 3.20% BlueRocksRandomAccessFile::Read
          |       |   | |               |       + 3.20% read_random
          |       |   | |               |         + 3.20% BlueFS::_read_random
          |       |   | |               |           + 3.20% KernelDevice::read_random
          |       |   | |               |             + 3.20% pread
          |       |   | |               |               + 3.20% pread64
          |       |   | |               + 0.20% Value
          |       |   | |                 + 0.20% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       |   | |                   + 0.20% Fast_CRC32
          |       |   | |                     + 0.20% Slow_CRC32
          |       |   | + 0.40% replace_top
          |       |   |   + 0.40% downheap
          |       |   |     + 0.40% operator()
          |       |   |       + 0.40% rocksdb::InternalKeyComparator::Compare
          |       |   |         + 0.20% b::(anonymous namespace)::BytewiseComparatorImpl::Compare
          |       |   |           + 0.20% compare
          |       |   + 0.40% SetInternalKey
          |       |   | + 0.40% SetInternalKey
          |       |   |   + 0.40% SetKeyImpl
          |       |   |     + 0.40% memcpy
          |       |   |       + 0.40% __memcpy_ssse3_back
          |       |   + 0.20% rocksdb::RangeDelAggregator::ShouldDelete
          |       |   | + 0.20% ParseInternalKey
          |       |   + 0.20% rocksdb::MergingIterator::value
          |       |   | + 0.20% value
          |       |   |   + 0.20% b::(anonymous namespace)::TwoLevelIterator::value
          |       |   + 0.20% rocksdb::MergingIterator::key
          |       |   + 0.20% b::(anonymous namespace)::BytewiseComparatorImpl::Equal
          |       + 9.80% rocksdb::BlockBasedTableBuilder::Add
          |       | + 6.60% rocksdb::BlockBasedTableBuilder::Flush
          |       | | + 4.20% rocksdb::BlockBasedTableBuilder::WriteBlock
          |       | | | + 4.00% rocksdb::BlockBasedTableBuilder::WriteBlock
          |       | | | | + 4.00% rocksdb::BlockBasedTableBuilder::WriteRawBlock
          |       | | | |   + 2.40% Value
          |       | | | |   | + 2.40% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       | | | |   |   + 2.20% Fast_CRC32
          |       | | | |   |     + 2.20% Slow_CRC32
          |       | | | |   + 1.60% rocksdb::WritableFileWriter::Append
          |       | | | |     + 1.60% rocksdb::WritableFileWriter::Flush
          |       | | | |       + 1.00% rocksdb::WritableFileWriter::WriteBuffered
          |       | | | |       | + 1.00% BlueRocksWritableFile::Append
          |       | | | |       |   + 1.00% append
          |       | | | |       |     + 1.00% append
          |       | | | |       |       + 1.00% memcpy
          |       | | | |       |         + 1.00% __memcpy_ssse3_back
          |       | | | |       + 0.60% BlueRocksWritableFile::Flush
          |       | | | |         + 0.60% flush
          |       | | | |           + 0.60% lock_guard
          |       | | | |             + 0.60% lock
          |       | | | |               + 0.60% __gthread_mutex_lock
          |       | | | |                 + 0.60% pthread_mutex_lock
          |       | | | |                   + 0.60% _L_lock_812
          |       | | | |                     + 0.60% __lll_lock_wait
          |       | | | + 0.20% rocksdb::BlockBuilder::Finish
          |       | | |   + 0.20% PutFixed32
          |       | | + 2.40% rocksdb::BlockBasedFilterBlockBuilder::StartBlock
          |       | |   + 2.40% rocksdb::BlockBasedFilterBlockBuilder::GenerateFilter
          |       | |     + 2.20% b::(anonymous namespace)::BloomFilterPolicy::CreateFilter
          |       | |     | + 0.80% rocksdb::Hash
          |       | |     + 0.20% push_back
          |       | |       + 0.20% std::vector<unsigned int, std::allocator<unsigned int> >::emplace_back<unsigned int>(unsigned int&&)
          |       | |         + 0.20% construct<unsigned int, unsigned int>
          |       | |           + 0.20% _S_construct<unsigned int, unsigned int>
          |       | |             + 0.20% construct<unsigned int, unsigned int>
          |       | + 1.20% rocksdb::BlockBuilder::Add
          |       | | + 0.80% std::string::append(char const*, unsigned long)
          |       | | | + 0.80% __memcpy_ssse3_back
          |       | | + 0.20% std::string::_M_replace_safe(unsigned long, unsigned long, char const*, unsigned long)
          |       | | | + 0.20% std::string::_M_mutate(unsigned long, unsigned long, unsigned long)
          |       | | + 0.20% difference_offset
          |       | + 1.00% rocksdb::NotifyCollectTableCollectorsOnAdd
          |       | | + 0.40% rocksdb::InternalKeyPropertiesCollector::InternalAdd
          |       | | | + 0.20% ParseInternalKey
          |       | | |   + 0.20% IsExtendedValueType
          |       | | |     + 0.20% IsValueType
          |       | | + 0.20% ~Status
          |       | + 0.80% rocksdb::ShortenedIndexBuilder::AddIndexEntry
          |       | | + 0.40% rocksdb::InternalKeyComparator::FindShortestSeparator
          |       | | | + 0.20% std::basic_string<char, std::char_traits<char>, std::allocator<char> >::basic_string(char const*, unsigned long, std::allocator<char> const&)
          |       | | | | + 0.20% char* std::string::_S_construct<char const*>(char const*, char const*, std::allocator<char> const&, std::forward_iterator_tag)
          |       | | | + 0.20% PutFixed64
          |       | | |   + 0.20% std::string::append(char const*, unsigned long)
          |       | | |     + 0.20% std::string::reserve(unsigned long)
          |       | | |       + 0.20% std::string::_Rep::_M_clone(std::allocator<char> const&, unsigned long)
          |       | | |         + 0.20% std::string::_Rep::_S_create(unsigned long, unsigned long, std::allocator<char> const&)
          |       | | |           + 0.20% tc_new
          |       | | + 0.20% ~basic_string
          |       | | | + 0.20% _M_dispose
          |       | | |   + 0.20% tc_delete
          |       | | + 0.20% rocksdb::BlockBuilder::Add
          |       | |   + 0.20% std::string::append(char const*, unsigned long)
          |       | |     + 0.20% __memcpy_ssse3_back
          |       | + 0.20% ok
          |       |   + 0.20% rocksdb::BlockBasedTableBuilder::status
          |       + 0.60% rocksdb::CompactionJob::FinishCompactionOutputFile
          |       | + 0.60% rocksdb::WritableFileWriter::Sync
          |       |   + 0.40% rocksdb::WritableFileWriter::SyncInternal
          |       |   | + 0.40% BlueRocksWritableFile::Sync
          |       |   |   + 0.40% fsync
          |       |   |     + 0.40% BlueFS::_fsync
          |       |   |       + 0.40% BlueFS::_flush_bdev_safely
          |       |   |         + 0.20% clear
          |       |   |         | + 0.20% std::_List_base<aio_t, std::allocator<aio_t> >::_M_clear
          |       |   |         |   + 0.20% destroy<std::_List_node<aio_t> >
          |       |   |         |     + 0.20% ~_List_node
          |       |   |         |       + 0.20% ~aio_t
          |       |   |         |         + 0.20% ~list
          |       |   |         |           + 0.20% ~list
          |       |   |         |             + 0.20% ~_List_base
          |       |   |         |               + 0.20% std::_List_base<ceph::buffer::ptr, std::allocator<ceph::buffer::ptr> >::_M_clear
          |       |   |         |                 + 0.20% destroy<std::_List_node<ceph::buffer::ptr> >
          |       |   |         |                   + 0.20% ~_List_node
          |       |   |         |                     + 0.20% ~ptr
          |       |   |         |                       + 0.20% ceph::buffer::ptr::release
          |       |   |         |                         + 0.20% ceph::buffer::raw_posix_aligned::~raw_posix_aligned
          |       |   |         |                           + 0.20% ~raw_posix_aligned
          |       |   |         |                             + 0.20% tc_free
          |       |   |         |                               + 0.20% tcmalloc::PageHeap::Delete(tcmalloc::Span*)
          |       |   |         |                                 + 0.20% tcmalloc::PageHeap::MergeIntoFreeList(tcmalloc::Span*)
          |       |   |         |                                   + 0.20% tcmalloc::PageHeap::DecommitSpan(tcmalloc::Span*)
          |       |   |         |                                     + 0.20% TCMalloc_SystemRelease(void*, unsigned long)
          |       |   |         |                                       + 0.20% madvise
          |       |   |         + 0.20% BlueFS::wait_for_aio
          |       |   |           + 0.20% IOContext::aio_wait
          |       |   |             + 0.20% std::condition_variable::wait(std::unique_lock<std::mutex>&)
          |       |   |               + 0.20% pthread_cond_wait@@GLIBC_2.3.2
          |       |   + 0.20% rocksdb::WritableFileWriter::Flush
          |       |     + 0.20% BlueRocksWritableFile::Flush
          |       |       + 0.20% flush
          |       |         + 0.20% BlueFS::_flush
          |       |           + 0.20% BlueFS::_flush_range
          |       |             + 0.20% IOContext::aio_wait
          |       |               + 0.20% std::condition_variable::wait(std::unique_lock<std::mutex>&)
          |       |                 + 0.20% pthread_cond_wait@@GLIBC_2.3.2
          |       + 0.40% rocksdb::MergingIterator::SeekToFirst
          |       | + 0.40% SeekToFirst
          |       |   + 0.40% b::(anonymous namespace)::TwoLevelIterator::SeekToFirst
          |       |     + 0.40% b::(anonymous namespace)::TwoLevelIterator::InitDataBlock
          |       |       + 0.40% b::(anonymous namespace)::LevelFileIteratorState::NewSecondaryIterator
          |       |         + 0.40% rocksdb::TableCache::NewIterator
          |       |           + 0.40% rocksdb::TableCache::GetTableReader
          |       |             + 0.40% rocksdb::BlockBasedTableFactory::NewTableReader(rocksdb::TableReaderOptions const&, std::unique_ptr<rocksdb::RandomAccessFileReader, std::default_delete<rocksdb::RandomAccessFileReader> >&&, unsigned long, std::unique_ptr<rocksdb::TableReader, std::default_delete<rocksdb::TableReader> >*, bool) const
          |       |               + 0.40% rocksdb::BlockBasedTable::Open(rocksdb::ImmutableCFOptions const&, rocksdb::EnvOptions const&, rocksdb::BlockBasedTableOptions const&, rocksdb::InternalKeyComparator const&, std::unique_ptr<rocksdb::RandomAccessFileReader, std::default_delete<rocksdb::RandomAccessFileReader> >&&, unsigned long, std::unique_ptr<rocksdb::TableReader, std::default_delete<rocksdb::TableReader> >*, bool, bool, int)
          |       |                 + 0.20% rocksdb::BlockBasedTable::NewIndexIterator
          |       |                 | + 0.20% rocksdb::BlockBasedTable::CreateIndexReader
          |       |                 |   + 0.20% Create
          |       |                 |     + 0.20% b::(anonymous namespace)::ReadBlockFromFile
          |       |                 |       + 0.20% rocksdb::ReadBlockContents
          |       |                 |         + 0.20% ReadBlock
          |       |                 |           + 0.20% Value
          |       |                 |             + 0.20% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       |                 |               + 0.20% Fast_CRC32
          |       |                 |                 + 0.20% Slow_CRC32
          |       |                 + 0.20% Prefetch
          |       |                   + 0.20% b::(anonymous namespace)::ReadaheadRandomAccessFile::Prefetch
          |       |                     + 0.20% ReadIntoBuffer
          |       |                       + 0.20% BlueRocksRandomAccessFile::Read
          |       |                         + 0.20% read_random
          |       |                           + 0.20% BlueFS::_read_random
          |       |                             + 0.20% KernelDevice::read_random
          |       |                               + 0.20% pread
          |       |                                 + 0.20% pread64
          |       + 0.40% rocksdb::CompactionJob::SubcompactionState::ShouldStopBefore
          |       + 0.40% UpdateBoundaries
          |       | + 0.40% DecodeFrom
          |       |   + 0.40% std::string::_M_replace_safe(unsigned long, unsigned long, char const*, unsigned long)
          |       |     + 0.20% std::string::_M_mutate(unsigned long, unsigned long, unsigned long)
          |       |     + 0.20% __memcpy_ssse3_back
          |       + 0.20% rocksdb::VersionSet::MakeInputIterator
          |       | + 0.20% rocksdb::TableCache::NewIterator
          |       |   + 0.20% rocksdb::TableCache::GetTableReader
          |       |     + 0.20% rocksdb::BlockBasedTableFactory::NewTableReader(rocksdb::TableReaderOptions const&, std::unique_ptr<rocksdb::RandomAccessFileReader, std::default_delete<rocksdb::RandomAccessFileReader> >&&, unsigned long, std::unique_ptr<rocksdb::TableReader, std::default_delete<rocksdb::TableReader> >*, bool) const
          |       |       + 0.20% rocksdb::BlockBasedTable::Open(rocksdb::ImmutableCFOptions const&, rocksdb::EnvOptions const&, rocksdb::BlockBasedTableOptions const&, rocksdb::InternalKeyComparator const&, std::unique_ptr<rocksdb::RandomAccessFileReader, std::default_delete<rocksdb::RandomAccessFileReader> >&&, unsigned long, std::unique_ptr<rocksdb::TableReader, std::default_delete<rocksdb::TableReader> >*, bool, bool, int)
          |       |         + 0.20% rocksdb::BlockBasedTable::NewIndexIterator
          |       |           + 0.20% rocksdb::BlockBasedTable::CreateIndexReader
          |       |             + 0.20% Create
          |       |               + 0.20% b::(anonymous namespace)::ReadBlockFromFile
          |       |                 + 0.20% rocksdb::ReadBlockContents
          |       |                   + 0.20% ReadBlock
          |       |                     + 0.20% Value
          |       |                       + 0.20% b::crc32c::ExtendImpl<rocksdb::crc32c::Fast_CRC32>
          |       |                         + 0.20% Fast_CRC32
          |       |                           + 0.20% Slow_CRC32
          |       + 0.20% rocksdb::BlockBasedTableBuilder::FileSize
          |       + 0.20% reset
          |         + 0.20% operator()
          |           + 0.20% rocksdb::MergingIterator::~MergingIterator
          |             + 0.20% ~MergingIterator
          |               + 0.20% DeleteIter
          |                 + 0.20% b::(anonymous namespace)::TwoLevelIterator::~TwoLevelIterator
          |                   + 0.20% ~TwoLevelIterator
          |                     + 0.20% ~InternalIterator
          |                       + 0.20% rocksdb::Cleanable::~Cleanable
          |                         + 0.20% DoCleanup
          |                           + 0.20% rocksdb::BlockBasedTable::~BlockBasedTable
          |                             + 0.20% rocksdb::BlockBasedTable::~BlockBasedTable
          |                               + 0.20% rocksdb::BlockBasedTable::Close
          |                                 + 0.20% rocksdb::LRUCacheShard::Erase
          |                                   + 0.20% Free
          |                                     + 0.20% rocksdb::BinarySearchIndexReader::~BinarySearchIndexReader
          |                                       + 0.20% ~BinarySearchIndexReader
          |                                         + 0.20% ~unique_ptr
          |                                           + 0.20% operator()
          |                                             + 0.20% ~Block
          |                                               + 0.20% ~BlockContents
          |                                                 + 0.20% ~unique_ptr
          |                                                   + 0.20% operator()
          |                                                     + 0.20% tc_deletearray
          |                                                       + 0.20% tcmalloc::PageHeap::Delete(tcmalloc::Span*)
          |                                                         + 0.20% tcmalloc::PageHeap::MergeIntoFreeList(tcmalloc::Span*)
          |                                                           + 0.20% tcmalloc::PageHeap::DecommitSpan(tcmalloc::Span*)
          |                                                             + 0.20% TCMalloc_SystemRelease(void*, unsigned long)
          |                                                               + 0.20% madvise
          + 31.00% std::condition_variable::wait(std::unique_lock<std::mutex>&)
            + 31.00% pthread_cond_wait@@GLIBC_2.3.2


Thread: 28===8109===tp_osd_tp
......
......
......
}
```
具体文件可[下载](https://github.com/yinminggang/1-image-important-threads.txt)

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

**想得到所有69个线程的函数堆栈情况，可去[下载](https://github.com/yinminggang/1-image-all-threads.txt)**


