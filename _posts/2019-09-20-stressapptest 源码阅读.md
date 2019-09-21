---
published: true
layout: post
author: persuez
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
  - stressapptest
---
### 一、 ParseArgs()
| cmd | Sat member | how to interpret
| --- | --- | ---
| --coarse_grain_lock | 粗粒度锁 | ARG_KVALUE
| -M | size_mb_ | ARG_IVALUE
| --reserve_memory | reserve_mb_ | ARG_IVALUE
| -H | min_hugepages_mbytes_ | ARG_IVALUE
| -s | runtime_seconds_ | ARG_IVALUE
| -m | memory_threads_ | ARG_IVALUE
| -i | invert_threads_ | ARG_IVALUE
| -c | check_threads_ | ARG_IVALUE
| --cc_inc_count | cc_inc_count_ | ARG_IVALUE
| --cc_line_count | cc_cacheline_count_ | ARG_IVALUE
| --cc_line_size | cc_cacheline_size_ | ARG_IVALUE
| --cc_test | cc_test_ | ARG_KVALUE
| --cpu_freq_test | cpu_freq_test_ | ARG_KVALUE
| --cpu_freq_threshold | cpu_freq_threshold_ | ARG_IVALUE
| --cpu_freq_round | cpu_freq_round_ | ARG_IVALUE
| -C | cpu_stress_threads_ | ARG_IVALUE
| -l | logfilename_ | ARG_SVALUE
| -v | verbosity_ | ARG_IVALUE


没有直接改变 disk_pages_（每个文件的  page 数），它和 filesize 有关，filesize =  page_length_ * disk_pages_; page_length_ 可以通过指定 -p 修改， filesize 通过 --filesize 修改，之后通过 disk_pages = filesize / page_length_ 修改，如果 disk_pages_ 为 0，则将它强制变为 1。

channels_.size() 最多为 2，多了不支持。每个 channels_[i].size() 要相同，并且要为 2^n 形式，和 channels_[0].size() 比较，不一样会  “Channels 0 and %d have a different count of dram modules.”。

channel_width_ 不能小于 16， 且不能为 2^n 形式。（channel 的位数）

chip width = channel_width_ / channels_[0].size(), chip size 不能小于 8。

channel_hash_ 可以指定，如果channels_.size() 为 1，则强制 channel_hash_ 为 0

logprintf priority 大于 verbosity 时，什么都不做。



1. ARG_KVALUE(argument, variable, value)
2. ARG_IVALUE(argument, variable) 

### 二、Initialize()

InitializeLogfile 通过 use_logfile_ 判断是否使用 logfile，使用的话则打开 logfilename_，基础 flag 是 O_WRONLY | O_CREAT, mode 是 S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH
use_logfile_ 初始为 0， 在 ParseArgs() 中判断 -l 指定的 logfilename_ 不为 ""，则 use_logfile = 1

logfile_ 是打开 logfilename_ 的 fd

lseek 和 O_APPEND，单进程选择 lseek(SEEK_END)

条件变量为什么要和互斥量一起用

    In Thread1:

    pthread_mutex_lock(&m_mutex);   
    pthread_cond_wait(&m_cond,&m_mutex);   
    pthread_mutex_unlock(&m_mutex);  

     

    In Thread2:

    pthread_mutex_lock(&m_mutex);   
    pthread_cond_signal(&m_cond);   
    pthread_mutex_unlock(&m_mutex);  

     

    为什么要与pthread_mutex 一起使用呢？ 这是为了应对 线程1在调用pthread_cond_wait()但线程1还没有进入wait cond的状态的时候，此时线程2调用了 cond_singal 的情况。 如果不用mutex锁的话，这个cond_singal就丢失了。加了锁的情况是，线程2必须等到 mutex 被释放（也就是 pthread_cod_wait() 释放锁并进入wait_cond状态 ，此时线程2上锁） 的时候才能调用cond_singal.

string 的 data() 没有空字符结尾，c_str() 有

指定 use_logfile_ 和有效的 logfilename_ 之后，也会在 stdout 上面输出

reserve_mb_ 只有在不使用 hugepages 时才会产生影响。


### 三、内存

其实从外观上就可以看出来小张的内存条由很多海力士的内存颗粒组成。从内存控制器到内存颗粒内部逻辑，笼统上讲从大到小为：channel＞DIMM＞rank＞chip＞bank＞row/column，如下图：

![channel＞DIMM＞rank＞chip＞bank＞row/column](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/09/21/a23c4587d2f2d.jpg)
一个现实的例子是：

![channel、dimm、rank、chip](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/09/21/44fcb5d997043.jpg)


在这个例子中，一个i7 CPU支持两个Channel（双通道），每个Channel上可以插俩个DIMM,而每个DIMM由两个rank构成，8个chip组成一个rank。由于现在多数内存颗粒的位宽是8bit,而CPU带宽是64bit，所以经常是8个颗粒可以组成一个rank。所以小张的内存条2R X 8的意思是由2个rank组成，每个rank八个内存颗粒(为啥我们以后讲)。由于整个内存是4GB，我们可以算出单个内存颗粒是256MB。

DQS就是DQ数据的时钟，主控read数据时，DQS和DQ边沿对齐，write数据时，DQS处于DQ中间位置