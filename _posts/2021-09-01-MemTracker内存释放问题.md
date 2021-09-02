---
layout: post
title:  "排查 Doris Mem Tracker 内存释放问题 "
date:   2021-09-01 20:56:30 +0800
categories: jekyll update
---

## Mem Tracker 析构失败——内存没被释放

  在开发 Doris 并发导出查询结果集功能的时候，意外遇到一个 be 宕机的情况。从 be 的 ```log/be.out``` 日志中看到，是一个与 MemTracker 对象析构有关的问题。

```bash
start time: 2021年 08月 31日 星期二 14:13:10 CST
*** Check failure stack trace: ***
    @          0x2efd7cd  google::LogMessage::Fail()
    @          0x2eff4e1  google::LogMessage::SendToLog()
    @          0x2efd3c2  google::LogMessage::Flush()
    @          0x2effae9  google::LogMessageFatal::~LogMessageFatal()
    @          0x220503c  doris::MemTracker::~MemTracker()
    @          0x1f707be  std::_Sp_counted_ptr<>::_M_dispose()
    @          0x1cc9211  std::_Sp_counted_base<>::_M_release()
    @          0x1cc7429  std::__shared_count<>::~__shared_count()
    @          0x1cd8804  std::__shared_ptr<>::~__shared_ptr()
    @          0x1cd8846  std::shared_ptr<>::~shared_ptr()
    @          0x2ab06e0  doris::DataStreamSender::~DataStreamSender()
    @          0x2abbb22  doris::ResultFileSink::~ResultFileSink()
    @          0x2abbb3e  doris::ResultFileSink::~ResultFileSink()
```

## 1. 通过 gdb，精确找到 core 的具体位置

  由于 Doris 默认编译会开启 O3 优化，如果要通过 core 文件精确定位到错误的位置，首先需要开启 DEBUG 编译。

```bash
BUILD_TYPE=debug ./build.sh --be
```

  这时候生成的 core 文件通过 gdb 工具就可以精确定位到具体出错的位置了。通过下面命令，打开 core 栈发现是 ```mem_tracker.cpp:270``` 这行出现的错误。

```bash
gdb lib/palo_be core.12344
Program terminated with signal SIGABRT, Aborted.
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:50
50	../sysdeps/unix/sysv/linux/raise.c: No such file or directory.
(gdb) bt
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:50
#1  0x00007f813b21c523 in __GI_abort () at abort.c:79
#2  0x0000000002f0584a in google::DumpStackTraceAndExit () at src/utilities.cc:147
#3  0x0000000002efd7bd in google::LogMessage::Fail () at src/logging.cc:1599
#4  0x0000000002eff4d1 in google::LogMessage::SendToLog() () at src/logging.cc:1553
#5  0x0000000002efd3b2 in google::LogMessage::Flush() () at src/logging.cc:1422
#6  0x0000000002effad9 in google::LogMessageFatal::~LogMessageFatal (this=<optimized out>, __in_chrg=<optimized out>) at src/logging.cc:2125
#7  0x000000000220503c in doris::MemTracker::~MemTracker (this=0xc347540, __in_chrg=<optimized out>) at ../src/runtime/mem_tracker.cpp:270
```

  结合代码具体行我发现，是析构 MemTracker 的时候有内存 consumption 不为 0，导致的析构错误。
![mem_tracker_code.png](/assets/mem_tracker_code.png)

## 2. 精确定位是内存没被释放的问题

  通过 gdb 命令进入具体的层级，确认 consumption 的值。

```bash
(gdb) frame 7
#7  0x000000000220503c in doris::MemTracker::~MemTracker (this=0xc347540, __in_chrg=<optimized out>) at ../src/runtime/mem_tracker.cpp:270
270	        DCHECK(consumption() == 0) << "Memory tracker " << debug_string()
(gdb) p consumption_
$1 = {<std::__shared_ptr<doris::RuntimeProfile::HighWaterMarkCounter, (__gnu_cxx::_Lock_policy)2>> = {<std::__shared_ptr_access<doris::RuntimeProfile::HighWaterMarkCounter, (__gnu_cxx::_Lock_policy)2, false, false>> = {<No data fields>}, _M_ptr = 0x8e585e0, _M_refcount = {_M_pi = 0x8e585d0}}, <No data fields>}
(gdb) p *((RuntimeProfile::HighWaterMarkCounter*)0x8e585e0)
$3 = {<doris::RuntimeProfile::Counter> = {_vptr.Counter = 0x41b2cc8 <vtable for doris::RuntimeProfile::HighWaterMarkCounter+16>, _value = {_value = 12416}, _type = doris::TUnit::BYTES},
  current_value_ = {_value = 12288}}
```

  consumption = 12288 = 4096 + 4096 + 4096。这明显是一个内存申请后，未释放导致的问题。

  *这里需要注意一点的是，如果栈中 consumption 的值虽然 !=0 但是是一个负数，或者一个异常值的话，那就不一定是单纯的内存未释放的问题了。*

## 3. 定位是哪个 MemTracker 发生的问题

  由于 Doris 的整个查询过程中，充实着非常多的 MemTracker。比如有 query 级别的，fragment 级别的，instance 级别的等等众多 MemTracker。所以定位 Doris 的内存释放问题，首先第一步就是要**确定是哪个 MemTracker 发生的问题。**

  core 栈中可以看出 ```DataStreamSender::~DataStreamSender ``` 析构的时候会析构一个  ```shared_ptr<MemTracker>``` 。对比 Doris 的代码发现：

```c++
class DataStreamSender : public DataSink {
protected:
    std::shared_ptr<MemTracker> _mem_tracker;
}
```

  ```_mem_tracker``` 嫌疑最大。为锁定嫌疑 ```MemTracker ``` ，我通过 gdb 方式启动 BE 进程，并且在析构的时候增加了断点，然后发现。
1. consumption = 12288
2. 此时的 ```MemTracker``` 指向的地址，和 DataStreamSender 中对应成员变量指向的地址一致。**出问题的就是 DataStreamSender 的成员变量 ```_mem_tracker```**

## 4. 谁忘记释放内存了

  锁定 MemTracker 后下一步就是找到，**谁申请了且忘记释放了**。

  + 首先有可能使用这个 MemTracker 申请资源的就是 DataStreamSender 了。但从实现上，虽然变量声明在类中，实现上却没有任何地方使用。
  + **怀疑子类** ResultFileSink。


  从代码中可以看到，唯一使用了这个 ```_mem_tracker``` 的对象就是 ResultFileSink 的 ```_output_batch```。

```c++
Status ResultFileSink::prepare(RuntimeState* state) {
    ...
    _output_batch = new RowBatch(_output_row_descriptor, 1024, _mem_tracker.get());
    ...
}
```
  而且在构造 ```_output_batch``` 中的内容时，确实申请了内存用于存储 batch 中的数据。
```c++
Status FileResultWriter::_fill_result_batch() {
    Tuple* tuple = (Tuple*)_output_batch->tuple_data_pool()->allocate(tuple_desc->byte_size());
}
```

  **怀疑 ```_output_batch``` 是那个忘记释放内存的罪魁祸首**

## 5. 释放内存

  有了怀疑的对象，接下来就是**找到释放内存的方式**并且修复他。```_output_batch``` 他作为一个 RowBatch 对象，唯一会释放内存的地方就是析构的时候。

```c++
RowBatch::~RowBatch() {
    clear();
}

void RowBatch::clear() {
    ...
    _mem_tracker->Release(_tuple_ptrs_size);
    ...
}
```

  那么问题就从**释放内存转化为析构RowBatch**。```_output_batch``` 作为一个指针他需要在所属类中手动 delete 才会被析构。**所以需要在 ResultFileSink 析构的时候主动 delete 即可解决问题。**

```c++
ResultFileSink:~ResultFileSink() {
    delete _output_batch;
}
```

## 6. 总结

  MemTracker 内存释放的问题一般有两个关键点：
  1. 精确定位是哪个 MemTracker 的问题。
  2. 谁 consume 了却没 release 。

  由于 Doris 中的 MemTracker 非常的多，所以第一点找对出错的 MemTracker 显得至关重要。

  C++ 中，new 和 delete 是成对出现的。只 new 不 delete 就可能**存在内存泄露**的问题。避免这种问题的出现，还有一种更好的方式：

  **使用智能指针 ```shared_ptr```**。 

## 附录

  gdb 调试中的一些简单用法：
1. 打开 core 栈：gdb lib/palo_be core.xxx
2. 进入其中一层：frame 1
3. 打印值：p *((类型*)地址)
4. 调试相关：b 断点，r 从头运行，c continue，n 单步，s step in

