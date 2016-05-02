#练习1: 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题（不需要编码）
 
请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。
>信号量设计主要是由sem.c文件完成。
```
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
 
void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);
``` 
>semaphore_t是信号量数据结构，实际上就是是数值和一个等待队列。其中` sem_init(semaphore_t *sem, int value)`用于初始化信号量，`up(semaphore_t *sem)`对应`V()`, `down(semaphore_t *sem)`对应`P()`.

>大致流程：
down():
先关闭中断，判断value是否大于0，如果大于0，说明此时有资源，则直接恢复中断，退出down();
如果小于或等于0，则将value减一，将当前进程加入等待队列，执行调度
当调度完成回来时，将当前进程从等待队列中删除
up():
先关闭中断，判断现在等待队列中是否有进程，如果没有value加一，表明资源增加1，然后直接退出。
如果有等待进程，则唤醒进程。

请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。
 
#练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）
 
首先掌握管程机制，然后基于信号量实现完成条件变量实现，然后用管程机制实现哲学家就餐问题的解决方案（基于条件变量）。
>需要实现四个文件
```
// Unlock one of threads waiting on the condition variable. 
void     cond_signal (condvar_t *cvp);
// Suspend calling thread on a condition variable waiting for condition atomically unlock mutex in monitor,
// and suspends calling thread on conditional variable after waking up locks mutex.
void     cond_wait (condvar_t *cvp);
void phi_take_forks_condvar(int i);
void phi_put_forks_condvar(int i);
``` 
请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程。
```
int philosopher_using_condvar(void * arg) { /* arg is the No. of philosopher 0~N-1*/
 
    int i, iter=0;
    i=(int)arg;
    cprintf("I am No.%d philosopher_condvar\n",i);
    while(iter++<TIMES)
    { /* iterate*/
        cprintf("Iter %d, No.%d philosopher_condvar is thinking\n",iter,i); /* thinking*/
        do_sleep(SLEEP_TIME);
        phi_take_forks_condvar(i);
        /* need two forks, maybe blocked */
        cprintf("Iter %d, No.%d philosopher_condvar is eating\n",iter,i); /* eating*/
        do_sleep(SLEEP_TIME);
        phi_put_forks_condvar(i);
        /* return two forks back*/
    }
    cprintf("No.%d philosopher_condvar quit\n",i);
    return 0;   
}
```
>思路就是进入管城时，先尝试获取叉子`phi_take_forks_condvar(int i)`，如果获取不到，就调用`cond_wait()`.如果有人尝试获取叉子成功，那么在其结束进餐时，放回刀叉时`phi_put_forks_condvar(int i)`,调用`phi_test_condvar (i)`唤醒左右临近的人。这时如果符合条件，在等待队列中的哲学家就会被唤醒，并拿到叉子。这里有个区别是是用Hansen还是Hoare，如果是Hansen就用while循环来判断条件变量，如果是Hoare就用if来判断条件变量

请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。