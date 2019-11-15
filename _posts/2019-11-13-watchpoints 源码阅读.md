---
published: true
tags:
  - 调试
author: persuez
---

## 硬件断点和软件断点
断点分为两类：硬件断点和软件断点。硬件断点需要目标 CPU 的支持，其使用硬件提供的断点寄存器来与设置断点的地址进行匹配，断点寄存器的数量是有限的，因此硬件断点的数量也是有限的，但因为用的是这种匹配的模式的设置断点，因此它可以设置任何位置的地址，包括 ROM 和 RAM。软件断点是通过修改相应地址的值，当然在修改值之前会进行代码备份，因此其数量没有限制，但是因为有的地址是不能修改的，所以并不是所有地址都可以设置软件断点，软件断点实现的具体方法如在需要设置断点的位置设置某个特征值，一般为 0x0000 等不易与代码混淆的值，GDB 的做法是修改地址处的指令为 INT3 指令，这个指令是 1 字节编码指令，其机器码为 0xCC，因此又称为 CC 指令，程序执行到这条指令后会触发 SIGTRAP 信号，GDB 则捕获这个信号，根据目标程序当前位置查询 GDB 维护的断点链表，若发现该地址确实存在断点，则可判定为断点命中。

## watchpoints
这篇文章的 watchpoints 源码见 [链接](https://github.com/BenjaminSchubert/watchpoints.git)，它是实现的硬件断点调试，实现方式是通过 linux 内核驱动的方式，因此这个 watchpoints 的本质上是一个模块化的驱动。它在 /dev 目录下创建一个杂项设备，也称为混杂设备，这是一种比较特殊的字符设备，Linux 包含了许多的设备驱动类型，而不管分类有多细，总会有些漏网的，这就是我们经常说到的“其他的”等等。在Linux里面，把无法归类的五花八门的设备定义为混杂设备（用 misc device 结构体来描述）。Linux/内核所提供的 misc device 有很强的包容性。如 NVRAM，看门狗，DS1286 等实时时钟，字符 LCD , AMD 768 随机数发生器。watchpoints 使用混杂设备来注册断点，如要设置一个断点，就打开 /dev/watchpoints 文件，通过 ioctl 来注册。watchpoints 用 /proc 文件系统来存储相关的调试信息，因此 watchpoints 中会涉及到 /proc 文件系统中一些很有意思的操作，涨姿势~

## 驱动初始化
linux 中驱动一般通过 [xxx_initcall、xxx_initcall_sync 或 module_init](https://blog.csdn.net/wh_19910525/article/details/11524729) 来初始化驱动，watchpoints 中通过 module_init 初始化，我阅读驱动代码是从这里开始的。
```c
module_init(watchpoint_init);
```
因此，接着来看 watchpoint_init 函数。
```c
static int __init watchpoint_init(void)
{
	proc_watchpoints = proc_mkdir("watchpoints", NULL); ----------------------------------------------------- /* 1 */

	INIT_LIST_HEAD(&tracked_data.list); --------------------------------------------------------------------- /* 2 */
	misc_register(&watchpoints_misc); ----------------------------------------------------------------------- /* 3 */
	pr_info("module successfully loaded\n");
	return 0;
}
```
1. `proc_watchpoints` 是一个全局变量。一开始就遇到了和 /proc 相关的调用，我们来看看 `proc_mkdir` 的原型：
   ```c
   struct proc_dir_entry *proc_mkdir(const char *name, struct proc_dir_entry *parent)；
   ```
   - 功能：在 /proc 下创建一个目录
   - name：要创建的目录名
   - parent：父目录，如果为 NULL，则表示直接在 /proc 下创建目录
   因此，`proc_watchpoints` 指向目录 `/proc/watchpoints`。
2. `tracked_data` 的声明如下：
   ```c
   struct tracked_pid_list tracked_data;
   ```
   tracked_pid_list 结构体记录每个使用 watchpoints 设置硬件断点的进程信息，其 list 成员将所有的进程链成链表，而链表头则为全局变量 `tracked_data`，至于如果觉得竟然是在数据中包含 list 成员，而不是 list 结构体包含数据成员这点比较疑惑，可以去看看内核的链表的操作，有意思的很。
3. `misc_register` 注册一个混杂设备，混杂设备的主设备号统一为 10，只需要指定其次设备号就可以。具体看看 `watchpoints_misc` 变量：
   ```c
   struct miscdevice watchpoints_misc = {
        .minor = MISC_DYNAMIC_MINOR,
        .name = DEVICE_NAME,
        .fops = &ctrl_fops
    };
   ```
   - `MISC_DYNAMIC_MINOR` 表示动态分配次设备号。
   - `DEVICE_NAME` 是在 `watchpoints.h` 文件中定义的一个宏，它的值为 "watchpoints"，因此驱动初始化完成之后，在 `/dev` 下可以发现有一个名为 `/dev/watchpoints` 的设备。
   - `ctrl_fops` 定义了操作设备 `/dev/watchpoints` 的接口，主要是定义了 `ioctl` 的接口函数：
        ```c
        const struct file_operations ctrl_fops = {
            .owner = THIS_MODULE,
            .read = NULL,
            .write = NULL,
            .unlocked_ioctl = watchpoints_ioctl,
            .open = NULL,
            .release = NULL,
        };
        ```

































struct pid
{
    atomic_t count;
    unsigned int level;
    struct hlist_head tasks[PIDTYPE_MAX];
    struct rcu_head rcu;
    struct upid numbers[1];
};
其中 count是指向该数据结构的引用次数。

通过 pid 号（pid_t 类型）找到 struct pid 的方法：find_vpid 和 find_get_pid, find_get_pid 是通过 find_vpid 实现的，但是 find_get_pid 会使 struct pid 的 count 成员 +1。

Prevent the compiler from merging or refetching accesses. The compiler is also forbidden from reordering successive instances of ACCESS_ONCE(), but only when the compiler is aware of some particular ordering. One way to make the compiler aware of ordering is to put the two invocations of ACCESS_ONCE() in different C statements.

ACCESS_ONCE will only work on scalar types. For union types, ACCESS_ONCE on a union member will work as long as the size of the member matches the size of the union and the size is smaller than word size.

The major use cases of ACCESS_ONCE used to be (1) Mediating communication between process-level code and irq/NMI handlers, all running on the same CPU, and (2) Ensuring that the compiler does not fold, spindle, or otherwise mutilate accesses that either do not require ordering or that interact with an explicit memory barrier or atomic instruction that provides the required ordering.

If possible use READ_ONCE()/WRITE_ONCE() instead.