##make qemu 后发生的事
BIOS和bootloader基本不变，bootloader增加一处变化
|.. |
|   |> bootasm.asm
|   |   | 探测物理地址，并保存在e820map中
|.. |
|   |> entry.S(lab1这里直接跳转到init)
|   |   | 为c语言设置堆栈
|   |> init.c
|.. |   |
|   |   |> pmm.c
|   |   |   |