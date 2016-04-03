##lab1 make qemu后发生的事
```
|>    硬件控制
|    | 系统加电后，寄存器CS：EIP初值是0xFFFFFFF0
|>    BIOS控制
|    | 地址0xFFFFFFF0是一个长跳转指令，跳转至BIOS起始地址
|    | BIOS进行计算机硬件自检和初始化，会选择一个启动设备（例如软盘、硬盘、光盘等），并且读取该设备的第一扇区(即主引导扇区或启动扇区)到内存一个特定的地址0x7c00处
|>    bootloader bootasm控制
|    | 第一个扇区是bootloader，控制权交给bootloader
|    | bootloader关闭中断，确保boot.s是当前唯一执行的程序，然后打开A20使之能够自由访问内存
|    | 加载gdt并通过长跳转使能保护模式 
|    | 设置好栈空间，为调用c函数作准备
|>    bootmain.c 控制
|    | bootasm调用bootmain，控制权交给bootmain
|    | bootmain读取接下来8个扇区（ELF文件在内），并存在0x10000
|    | 解析ELF文件，找到所有的程序头(可执行文件的程序头部是一个program header结构的数组， 每个结构描述了一个段或者系统准备程序执行所必需的其它信息。)，并将每个程序头对应的程序放入指定的地址
|    | 调用ELF文件的入口地址
|>    init.c控制
|    | 初始化console
|    | 初始化物理内存控制（lab2）
|    | 跳转至trap.c,开始初始化全局中断表
|    |>    trap.c
|    |    |>    vector.S
|    |    |设置ISR（中断服务例程）
|    |    |设置ISR到IDT（中断描述表）中
|    |    |调用lidt使CPU知道IDT在哪
|    | 初始化时钟中断
|    | 使能中断
|    |>    trap.c
|    | 中断触发后由dispath负责响应
```
