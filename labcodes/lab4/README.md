##练习1：分配并初始化一个进程控制块（需要编码）
alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。
```
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&(proc->context), 0, sizeof(struct context));
        proc->tf = NULL;
        proc->cr3 = boot_cr3;
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN);
```
- 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）

>context：进程的上下文，用于进程切换（参见switch.S）。在 uCore中，所有的进程在内核中也是相对独立的（例如独立的内核堆栈以及上下文等等）。使用 context 保存寄存器的目的就在于在内核态中能够进行上下文之间的切换。实际利用context进行上下文切换的函数是在kern/process/switch.S中定义switch_to。
 
>tf：中断帧的指针，总是指向内核栈的某个位置：当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。除此之外，uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的trapframe，uCore 在内核栈上维护了 tf 的链，可以参考trap.c::trap函数做进一步的了解。

##练习2：为新创建的内核线程分配资源（需要编码）
 
创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：
 
- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 将新进程添加到进程列表
- 唤醒新进程
- 返回新进程号
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。
```
// 1. call alloc_proc to allocate a proc_struct
if ((proc = alloc_proc()) == NULL) {
	goto fork_out;
}
proc->parent = current;
// 2. call setup_kstack to allocate a kernel stack for child process		
if (setup_kstack(proc) != 0) {
	goto bad_fork_cleanup_kstack;
}
// 3. call copy_mm to dup OR share mm according clone_flag
if (copy_mm(clone_flags, proc) != 0) {
	goto bad_fork_cleanup_proc;
}
// 4. call copy_thread to setup tf & context in proc_struct
copy_thread(proc, stack, tf);
// 5. insert proc_struct into hash_list && proc_list

bool intr_flag;
local_intr_save(intr_flag);
{
	proc->pid = get_pid();
	hash_proc(proc);
	list_add(&proc_list, &(proc->list_link));
	nr_process ++;
}
local_intr_restore(intr_flag);

wakeup_proc(proc);

ret = proc->pid;
```
进程号唯一性是由
```
local_intr_save(intr_flag);
{
	proc->pid = get_pid();
	hash_proc(proc);
	list_add(&proc_list, &(proc->list_link));
	nr_process ++;
}
local_intr_restore(intr_flag);
```
这里关中断，因此可以保证唯一性

##练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）
请在实验报告中简要说明你对proc_run函数的分析。并回答如下问题：
在本实验的执行过程中，创建且运行了几个内核线程？
>2个，一个idle一个init_main

语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由
>关中断，保证切换上下文时，不会被中断打断