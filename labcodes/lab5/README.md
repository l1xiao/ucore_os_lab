##练习1: 加载应用程序并执行（需要编码）
 
do_execv函数调用load_icode（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。

请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。
>do_execve这个函数通过load_icode()来加载elf文件到子进程的内存中，并修改了子进程对应的中断帧

```
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
```
>可以看到cs和ds都是用户态的基址，栈也转换成了用户栈。eip指向e_entry, 即hello中_start位置。此时就完成了子进程的创建。这个时候该进程就在可以等待调度了。
当被调度时，会调用proc_run(next), 切换当前栈到next的内核栈，切换当前的页目录为用户进程页目录。

```
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```

>通过switch_to恢复之前的寄存器, 并在最后将_start位置压入eip，退出switch_to时，跳转到要执行的指令

```
.text
.globl switch_to
switch_to:                      # switch_to(from, to)
 
    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
    ... 
    # restore to's registers
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
    ... 
    pushl 0(%eax)               # push eip
    ret
 
```
 
##练习2: 父进程复制自己的内存空间给子进程（需要编码）
 
创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。
 
请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。
 
>Copy-on-write（简称COW）的基本概念是指如果有多个使用者对一个资源A（比如内存块）进行读操作，则每个使用者只需获得一个指向同一个资源A的指针，就可以该资源了。若某使用者需要对这个资源A进行写操作，系统会对该资源进行拷贝操作，从而使得该“写操作”使用者获得一个该资源A的“私有”拷贝—资源B，可对资源B进行写操作。该“写操作”使用者对资源B的改变对于其他的使用者而言是不可见的，因为其他使用者看到的还是资源A。

##练习3: 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）
 
请在实验报告中简要说明你对 fork/exec/wait/exit函数的分析。并回答如下问题：
 
请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？
请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）


>fork最终调用到之前实验实现的do_fork函数，从而完成子进程的创建，资源的分配；
exec最终调用到do_execve函数，do_execve函数对于非内核进程首先要切换到内核空间，然后exit_mmap、

>put_pgdir和mm_destroy来回收当前进程在用户空间的资源，完成后再通过load_icode函数将新程序加载到用户空间，分配资源，设置各指针等，完成进程的改变。

>wait最终调用到do_wait函数，do_wait函数通过一个循环，不断的查找子进程中状态为PROC_ZOMBIE的，若找到则结束循环并清理子进程剩余的无法由其自身清理的资源，每一轮循环中若未找到则会先尝试调用schedule让出CPU。

>exit最终调⽤用到do_exit函数，do_exit对于非内核进程也是先切换到内核空间，然后和do_execve一样回收当前进程各种资源，接着设置进程状态为PROC_ZOMBIE，最后通过唤醒父进程来清理该进程其余未能回收的资源，并且对于其子进程需要转给initproc。