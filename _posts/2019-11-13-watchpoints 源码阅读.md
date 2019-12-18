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

但是这个 watchpoint 的实现是调用内核中封装的 register_user_hw_breakpoint 来注册断点的，这个函数似乎是为了兼容不同的硬件平台，因此它似乎只是 ARMv8 中 watchpoint 的最基本实现（不是很确定），譬如断点最大长度被限制为 HW_BREAKPOINT_LEN_8，而 ARMv8 中可支持最大长度为 2G。

### ARMv8 中的 watchpoints
[参考链接](https://persuez.me/2019/12/16/ARM-AArch64-Self-Hosted-Debug-Watchpoint/)

## 驱动初始化
linux 中驱动一般通过 [xxx_initcall、xxx_initcall_sync 或 module_init](https://blog.csdn.net/wh_19910525/article/details/11524729) 来初始化驱动，watchpoints 中通过 module_init 初始化，我阅读驱动代码是从这里开始的。
```c
module_init(watchpoint_init);
```
因此，接着来看 watchpoint_init 函数。
```c
static int __init watchpoint_init(void)
{
	proc_watchpoints = proc_mkdir("watchpoints", NULL); ------------------------- /* 1 */

	INIT_LIST_HEAD(&tracked_data.list); ----------------------------------------- /* 2 */
	misc_register(&watchpoints_misc); ------------------------------------------- /* 3 */
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
        值得一看的是，`unlocked_ioctl`，为什么不是 `ioctl`，这个和 Big Kernel Lock 有关，这个大概的情况可以去[链接](https://lwn.net/Articles/119652/)了解。现在应用层调用 `ioctl` 就会跑到执行 `watchpoints_ioctl` 函数。初始化到此结束，接着应用层会通过 `ioctl` 操作接口 `/dev/watchpoints`，因此我们接下来看 `watchpoints_ioctl` 函数。

## ioctl 接口
watchpoints 的 ioctl 接口函数是 `watchpoints_ioctl` 函数。
```c
static long watchpoints_ioctl(struct file *file, unsigned int cmd,
			      unsigned long ptr_message)
{
	struct watchpoint_message data; ------------------------------------------ /* 1 */
	long ret_val;

	/* TODO: check ret_val */
	copy_from_user(&data, (void *) ptr_message, sizeof(data));

	pr_info("Received from pid %d, ptr %p, size %ld\n", current->pid,
		data.data_ptr, data.data_size);

	switch (cmd) { ----------------------------------------------------------- /* 2 */
	case ADD_WATCHPOINT:
		ret_val = add_watchpoint(data);
		break;
	case REMOVE_WATCHPOINT:
		ret_val = remove_watchpoint(data);
		break;
	default:
		pr_info("Watchpoints was sent an unknown command %d\n",
			cmd);
		ret_val = -EINVAL;
		break;
	}

	return ret_val;
}
```
1. `watchpoint_message` 结构体是通过 watchpoints 设置断点所必须的结构体，也就是说，在应用层必须定义一个这样的结构体来和驱动进行数据传送。`watchpoint_message` 结构体的声明如下：
   ```c
    struct watchpoint_message {
        void *data_ptr;
        long data_size;
    };
   ```
   - `data_ptr` 成员是需要设置断点的地址
   - `data_size` 成员是地址大小范围。

2. `watchpoints_ioctl` 提供了两个 cmd：`ADD_WATCHPOINT` 和 `REMOVE_WATCHPOINT`。这里只关注 `ADD_WATCHPOINT`，也就是如何设置断点。在这里，只是调用了 `add_watchpoint` 函数，因此我们接下来转向 `add_watchpoint` 函数。

## add_watchpoint
先看看 `add_watchpoint` 函数的原型：`static long add_watchpoint(struct watchpoint_message data);`, 它的参数 `data` 是从用户空间传进来的，包含了断点地址和地址范围。
```c
static long add_watchpoint(struct watchpoint_message data)
{
	struct perf_event *perf_watchpoint; ----------------------------------- /* 1 */

	perf_watchpoint = initialize_watchpoint(data, current->pid);

	if (IS_ERR(perf_watchpoint)) {
		pr_info("Could not set watchpoint\n");
		return -EBUSY;
	}

	prepare_proc_entry(perf_watchpoint, data.data_size); ------------------ /* 2 */

	return 0;
}
```
1. `perf_event` 这个结构体具体的没仔细看，但是它和我们要注册的断点的上下文相关，注册断点返回的 `perf_watchpoint` 变量与断点的上下文，一些其他信息绑定了，这个在 `initialize_watchpoint` 函数中实现了。
2. `prepare_proc_entry` 函数以及之后的调用链是重点，之后再分别讲述。因此 `add_watchpoint` 这个函数就调用了两个函数：`initialize_watchpoint` 和 `prepare_proc_entry` 函数，接下来分别讲这两个函数。

## initialize_watchpoint 函数
`initialize_watchpoint` 函数是真正初始化硬件断点的地方。
```c
static struct perf_event *initialize_watchpoint(struct watchpoint_message
						data, pid_t pid)
{
	struct perf_event *perf_watchpoint;
	struct task_struct *tsk;
	struct perf_event_attr attr;

	/* Initialize watchpoint */
	hw_breakpoint_init(&attr); -------------------- /* 1 */
	attr.bp_addr = (u64) data.data_ptr;
	attr.bp_len = HW_BREAKPOINT_LEN_4;
	attr.bp_type = HW_BREAKPOINT_W;

	tsk = pid_task(find_vpid(pid), PIDTYPE_PID); ------------- /* 2 */

	perf_watchpoint =
	    register_user_hw_breakpoint(&attr, watchpoint_handler, ------ /* 3 */
					NULL, tsk);

	return perf_watchpoint;
}
```
1. `hw_breakpoint_init` 函数初始化了一个 `perf_event_attr` 结构体:
   ![hw_breakpoint_init](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/12/17/b352b37989e68.jpg)
   重点是接下来的三条语句，它们分别设置了 watchpoint 的地址，长度和触发条件。
   ![attr](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/12/17/c86e2355e63db.jpg)
   注意，在这个 watchpoint 实现中，上层穿的 watchpoint_message 结构体的 data_size 成员并没有什么作用。
2. 这里边的知识点是在内核空间如何通过 pid 查找对应的 struct pid 和 struct task_struct(include <linux/sched.h>):
   查找 struct pid: 
   ```c
   	struct pid
	{
		atomic_t count;
		unsigned int level;
		struct hlist_head tasks[PIDTYPE_MAX];
		struct rcu_head rcu;
		struct upid numbers[1];
	};
	```
	其中 count是指向该数据结构的引用次数。

	通过 pid 号（pid_t 类型）找到 struct pid 的方法：find_vpid 和 find_get_pid, find_get_pid 是通过 find_vpid 实现的，但是 find_get_pid 会使 struct pid 的 count 成员 +1。

   查找 struct task_struct：`tsk = pid_task(find_vpid(pid), PIDTYPE_PID);`
3. `register_user_hw_breakpoint` 函数用于注册用户空间断点，`register_wide_hw_breakpoint` 用于内核空间。

说完 `initialize_watchpoint` 函数了，接着进入 `prepare_proc_entry` 函数。

## prepare_proc_entry 函数
开讲这个函数之前，先介绍下这个 watchpoints 实现中的三个重要结构体：
```c
/* tracks pids */
struct tracked_pid_list;

/* tracks pointers */
struct tracked_pointer_list;

/* tracks changes */
struct tracked_changes_list;
```
见名得意，首先这三货都是用于形成一个内核链表的，它们的变量是属于链表的一个结点。从上到下（和源码中声明的顺序相反），`tracked_pid_list` 跟踪不同的进程，`tracked_pointer_list` 跟踪同一进程内的不同地址，也就是不同断点，`tracked_changes_list` 跟踪每个地址的变化。也就是说，可以有多个 watchpoints(数量和硬件相关，ARMv8 最多 16 个)，管理方法是分进程、地址管理，管理结构体是链表。下面就讲讲 `prepare_proc_entry` 函数：

```c
static void prepare_proc_entry(struct perf_event *event, long size)
{
	struct tracked_pid_list *tracked_pid = NULL;
	struct tracked_pid_list *pid_list;

	list_for_each_entry(pid_list, &tracked_data.list, list) { ------------ /* 1 */
		if (pid_list->pid == current->pid) {
			tracked_pid = pid_list;
			break;
		}
	}

	if (!tracked_pid) { --------------- /* 2 */
		struct proc_dir_entry *pid_entry;
		struct tracked_pointer_list *pointers;
		char pid_name[30];

		tracked_pid = kmalloc(sizeof(*tracked_pid), 0);
		pointers = kmalloc(sizeof(*pointers), 0);

		sprintf(pid_name, "%d", current->pid);
		pid_entry = proc_mkdir(pid_name, proc_watchpoints);

		tracked_pid->pid = current->pid;
		tracked_pid->entry = pid_entry;
		tracked_pid->pointers = pointers;
		INIT_LIST_HEAD(&pointers->list);

		pr_debug("added pid entry: %d\n", tracked_pid->pid);
		list_add_tail(&(tracked_pid->list), &tracked_data.list);
	}

	add_ptr_entry_to_pid(tracked_pid, event, size); ------- /* 3 */
}
```
1. 从进程链表中查看是否有该进程的结点了，有则简单的赋值，跑到第三步，没有转第二步
2. 上一步在进程链表中没找到该进程结点，那么是新来的，就创建一个呗：分配内存空间，链入进程链表，初始化属于它自己的地址指针链表（每个进程一个），在 /proc/watchpoints/ 下创建一个以 pid 为名的目录，接着转第三步
3. 根据 `add_ptr_entry_to_pid` 其名知道，该函数是要为跟踪地址创建一个 changes 链表，和 `prepare_proc_entry` 的操作差不多。

## add_ptr_entry_to_pid 函数
```c
static void add_ptr_entry_to_pid(struct tracked_pid_list *tracked_pid,
				 struct perf_event *event, long size)
{
	struct tracked_pointer_list *ptr_entry;
	int ptr_entry_created = 0;

	list_for_each_entry(ptr_entry, &tracked_pid->pointers->list, list) {
		if (ptr_entry->event->attr.bp_addr == event->attr.bp_addr) {
			ptr_entry_created = 1;
			break;
		}
	}

	if (!ptr_entry_created) {
		struct proc_dir_entry *proc_ptr;
		struct tracked_pointer_list *ptr_entry;
		struct tracked_changes_list *changes;
		char ptr_value[2 * sizeof(long) + 1];

		changes = kmalloc(sizeof(*changes), 0);

		sprintf(ptr_value, "%016llx", event->attr.bp_addr);

		ptr_entry = kmalloc(sizeof(*ptr_entry), 0);
		ptr_entry->event = event;
		ptr_entry->size = size;
		ptr_entry->changes = changes;
		INIT_LIST_HEAD(&changes->list);

		pr_debug("added ptr entry: %s\n", ptr_value);

		list_add_tail(&(ptr_entry->list),
			      &tracked_pid->pointers->list);

		proc_ptr =
		    proc_create_data(ptr_value, 0, tracked_pid->entry, ---------- /* 1 */
				     &proc_fops, ptr_entry);
		ptr_entry->entry = proc_ptr;
	}
}
```
这个函数我们只关注这个地方 ---- `proc_create_data` 函数，因为其他位置的逻辑和 `prepare_proc_entry` 差不多。讲解参考[链接](http://tuxthink.blogspot.com/2013/10/creation-of-proc-entrt-using.html)

函数原型
```c
/**
 * name: The name of the proc entry 
 * mode: The access mode for proc entry 
 * parent: The name of the parent directory under /proc 
 * proc_fops: The structure in which the file operations for the proc entry will be created. 
 * data: If any data needs to be passed to the proc entry. 
 **/
struct proc_dir_entry *proc_create_data(const char *name, umode_t mode,struct proc_dir_entry *parent,const struct file_operations *proc_fops,void *data);
```
类似 `proc_mkdir` 的 parent 参数，如果 parent 为 NULL ，则直接在 /proc 下创建 entry。接下来需要关注的点有 mode、proc_fops 和 data 的处理。

1. 关于 mode：在 v5.4-rc8 内核中（不同版本位置可能不一样，但逻辑还是一样的），
   文件 include/uapi/linux/stat.h：
   ![mode1](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/12/18/ee266c1e77a01.jpg)
   文件 include/linux/stat.h：
   ![mode2](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/12/18/3bf64a6356b22.jpg)
   在 `proc_create_data` 的处理逻辑中，若是 `mode` 形参为 0，则会被指定会 `S_IFREG | (S_IRUSR|S_IRGRP|S_IROTH)`，`S_IFREG` 表示文件类型为普通文件，`(S_IRUSR|S_IRGRP|S_IROTH)`表示文件的权限为 user 可读、group 可读、other 可读。具体代码在 fs/proc/generic.c 文件的 `proc_create_reg` 函数中，这个函数会被 `proc_create_data` 调用，不过之前版本的内核代码似乎没有 `proc_create_reg` 函数，相关逻辑直接怼在 `proc_create_data` 函数中。
   ![proc_create_reg 代码](http://persuez-image.oss-cn-shenzhen.aliyuncs.com/2019/12/18/7d1fa6ebde96d.jpg)

2. 关于 proc_fops：在 watchpoints 源码的定义为：
   ```c
    /* file operations for the pointer entries in /proc/watchpoints/pid */
	const struct file_operations proc_fops = {
		.owner = THIS_MODULE,
		.open = proc_open,
		.read = seq_read,
		.llseek = seq_lseek,
		.release = single_release
	};
   ```
   以下内容[参考](https://www.ibm.com/developerworks/cn/linux/l-kerns-usrs2/index.html)
   >一般地，内核通过在 procfs 文件系统下建立文件来向用户空间提供输出信息，用户空间可以通过任何文本阅读应用查看该文件信息，但是 procfs 有一个缺陷，如果输出内容大于 1 个内存页，需要多次读，因此处理起来很难，另外，如果输出太大，速度比较慢，有时会出现一些意想不到的情况，Alexander Viro 实现了一套新的功能，使得内核输出大文件信息更容易，该功能出现在 2.4.15（包括 2.4.15 ）以后的所有 2.4 内核以及 2.6 内核中，尤其是在2.6内核中，已经大量地使用了该功能。

   上述引用所说的新功能指 seq_file。要想使用 seq_file 功能，开发者需要包含头文件 linux/seq_file.h。seq_file 实现了一种 procfs 基本的“读写”功能，这种“读写”并非真正的读写了该 proc/ 文件，你可以试下写它，然而文件大小依旧为 0。这套接口要怎么用呢？如下：

   1. 定义一个 show 函数，如 watchpoints 中 `proc_display` 函数：
		```c
		static int proc_display(struct seq_file *m, void *v)
		{
			struct tracked_pointer_list *pointer =
				(struct tracked_pointer_list *) m->private;
			struct tracked_changes_list *change;

			list_for_each_entry(change, &pointer->changes->list, list) {
				long counter;
				seq_printf(m, "%lu ", change->data_size);
				for (counter = 0; counter < change->data_size; counter++) {
					seq_printf(m, "%c", change->data[counter]);
				}
			}

			return 0;
		}
		```
		当读这个文件时，这个函数会被调用，如 cat、vim。比如 cat 时 `seq_printf` 会将“文件的内容”输出到屏幕中，用法参考 `proc_display` 函数。
	2. 定义 `open` 函数，如 watchpoints 中 `proc_open` 函数：
		```c
		static int proc_open(struct inode *inode, struct file *file)
		{
			return single_open(file, proc_display, PDE_DATA(inode));
		}
		```
		这个 `proc_open` 是一个统一的模板，其中的 `single_open` 中的第三个参数和 `proc_create_data` 函数中传进去的 `data` 形参相关，如果 `single_open` 的第三个参数为 NULL，那么在 `proc_display` 中就拿不到 `data` 的值，即 `m->private`。
	3. 定义 `struct file_operations` 结构体，如：
		```c
		/* file operations for the pointer entries in /proc/watchpoints/pid */
		const struct file_operations proc_fops = {
			.owner = THIS_MODULE,
			.open = proc_open,
			.read = seq_read,
			.llseek = seq_lseek,
			.release = single_release
		};
		```
		需要注意的是，因为 `proc_open` 中使用了 `single_open`，因此 `release` 成员必须使用 `single_release`。

	watchpoints 的该实现分析到此结束。

## 学到了什么
1. watchpoint 是什么，它可以做什么？
2. 一个基本驱动的写法
3. 如何通过 pid 得到 struct task_struct
4. proc 文件系统基本操作