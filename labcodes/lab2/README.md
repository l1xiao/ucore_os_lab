#关于地址的基本概念
- CPU收到一个指令，这个指令中有对一个内存单元的读操作，所以指令包含了内存单元的地址，这个地址是虚拟地址（或逻辑地址）。
- 这个虚拟地址通过段机制（即全局描述符表中的段描述符（其索引来源于DS,...））的转换，形成了线性地址。
- 这个线性地址通过页机制（即页表，或者TLB）的转换，形成了物理地址，然后通过内存地址总线，把这个物理地址对应的内存单元从内存读入到cache,再读入到寄存器,让CPU进行相应的操作。

#lab2 make qemu后发生的事
```
|>  bootloader
|   | 一些初始化，此时virt addr = linear addr = phy addr
|...|
|   |>  kern_entry
|   |   |>  entry.S
|   |   |   | 建立好c语言运行环境，此时virt addr - 0xC0000000 = linear addr = phy addr
|   |   |>  kern_init
|...|...|
|   |   |>  pmm_init
|   |   |   |>  init_pmm_manager
|   |   |   |   | 获取default_pmm.c中的pmm，在这里实现了分配、释放算法
|   |   |   |>  page_init(建立空闲的page链表，这样就可以分配以页（4KB）为单位的空闲内存了)
|   |   |   |   | 从bootloader给出的内存布局信息(memmap)找出最大的物理内存地址maxpa, nr_map是数组大小
|   |   |   |   | 计算需要的管理的物理页个数
|   |   |   |   | 从地址0到地址pages+ sizeof(struct Page) * npage结束的物理内存空间设定为已占用物理内存空间
|   |   |   |   | 地址pages+ sizeof(struct Page) * npage)以上的空间为空闲物理内存空间，这时的空闲空间起始地址为freemem
|   |   |   |   | 从freemem开始知道最大页数，用init_memmap标记空闲空间
|   |   |   |>  check_alloc_page
|   |   |   |   | 检查alloc/free实现的正确性
|   |   |   | 创建初始化的Page Directory Table
|   |   |   | 自映射
|   |   |   |>  boot_map_segment
|   |   |   |   | 建立页表，填充pte
|   |   |   |   | 此时页表virt addr - 0xC0000000 = linear addr  = phy addr + 0xC0000000
|   |   |   |   | 建立临时映射 virt addr - 0xC0000000 = linear addr  = phy addr # 物理地址在0~4MB之内的三者映射关系
|   |   |   |>  enable_paging
|   |   |   |   | 使能页机制，此条指令之后的地址（其实只是0~4M）全都是逻辑地址，之前的都是线性地址
|   |   |   |>  gdt_init
|   |   |   |   | 重置gdt，使虚拟地址与线性地址直接映射
|   |   |   | 恢复临时映射
|   |   |   | 检查virtual memory map正确性
|...|
```