# “主”函数的编写

## 关于“主”函数的一些说明

通常我们的 C 语言程序都会有一个主函数（main 函数）作为程序的入口，在这里我们也先写一个 main 函数作为我们的内核的“主”函数，之所以在“主”函数上加引号，是因为实际上这个函数并不是整个系统真正的入口，而 main 的名字也是可以任意取的，这一细节后续还会再提到，在这里还是遵循我们的习惯，将它取名为 main。

## 相关函数的介绍

在编写操作系统时是有很多地方与编写普通程序有很大的不同的，首先就是写代码的时候无法使用一些常用的头文件，如 `stdio.h`, `stdlib.h` 等，因为这些头文件是依赖于系统的实现的，换句话说，这些头文件的功能并不是 C 语言或编译器天生就具备的，而是由操作系统提供了底层支持，由操作系统完成了内部的一些函数和定义。

既然不能使用这些头文件，我们就无法使用 `printf`, `scanf`, `puts`, `gets`, `putchar`, `getchar` 之类的函数。好在，sbi 为我们提供了一些字符输入输出的环境调用功能，但是需要我们使用汇编语言来调用，因此我们会对这些功能做 C 语言的函数封装，这样我们就能借助我们自己封装的 C 语言函数使用它们了。

在 `main` 函数中，我们使用了这几个封装：

- sbi_probe_extension
  - 探测 SBI 扩展的可用情况
- sbi_console_putchar
  - 打印字符
- sbi_get_impl_id
  - 获取当前的 SBI（Supervisor 二进制接口）实现 ID

同样我们也自己定义了一个符合 sbi 标准的结构体类型`struct sbiret`，用于表示一些 sbi 环境调用的返回值。在 `main` 函数中，主要是 `sbi_probe_extension` 函数使用了这个结构体，返回的值中若成员 value 的值不为0，则表示功能扩展可用，这些标准都会在后续介绍到，此处暂且不提。

另外我们又对 `sbi_console_putchar` 做了进一步封装，封装后函数名为 `kputs`，可以直接输出字符串，并在最后输出换行，封装代码如下：

```c
static void
kputs(const char *msg)
{
    for ( ; *msg != '\0'; ++msg )
    {
        sbi_console_putchar(*msg);
    }
    sbi_console_putchar('\n');
}
```

## “主”函数代码与功能介绍

在这里给出 `main` 函数的代码：

```c
int 
main()
{
    struct sbiret ret;
    ret = sbi_probe_extension(TIMER_EXTENTION);
    if (ret.value != 0)
        kputs("TIMER_EXTENTION: available");
    else 
        kputs("TIMER_EXTENTION: unavailable");

    ret = sbi_probe_extension(SHUT_DOWN_EXTENTION);
    if (ret.value != 0)
        kputs("SHUT_DOWN_EXTENTION: available");
    else 
        kputs("SHUT_DOWN_EXTENTION: unavailable");

    ret = sbi_probe_extension(HART_STATE_EXTENTION);
    if (ret.value != 0)
        kputs("HART_STATE_EXTENTION: available");
    else 
        kputs("HART_STATE_EXTENTION: unavailable");

    ret = sbi_get_impl_id();
    switch (ret.value)
    {
        case BERKELY_BOOT_LOADER:
            kputs("Implemention ID: Berkely boot loader");
            break;
        case OPENSBI:
            kputs("Implemention ID: OpenSBI");
            break;
        case XVISOR:
            kputs("Implemention ID: XVISOR");
            break;
        case KVM:
            kputs("Implemention ID: KVM");
            break;
        case RUSTSBI:
            kputs("Implemention ID: RustSBI");
            break;
        case DIOSIX:
            kputs("Implemention ID: DIOSIX");
            break;
        default:
            kputs("Implemention ID: Unkonwn");
    }
    kputs("Hello LZU OS");
    while (1)
        ; /* infinite loop */
    return 0;
}
```

根据前面的说明，我们很容易读通这个程序，它所做的不过就是检测`TIMER_EXTENTION`（时钟）、`SHUT_DOWN_EXTENTION`（关机）、`HART_STATE_EXTENTION`（硬件级线程）这几个 sbi 扩展是否可用 (available) ，检测当前 sbi 的实现，最后输出"Hello LZU OS"，随后**陷入死循环**。

> **HART 硬件级线程**  
>
> hart 是硬件线程(hardware thread)的缩略形式。 我们用该术语将它们与大多数程序员熟悉的软件线程区分开来。软件线程在 harts 上进行分时复用。 大多数处理器核都只有一个hart。  
> *—— 摘自[《RISC-V 手册》](http://riscvbook.com/chinese/RISC-V-Reader-Chinese-v2p1.pdf)*
>   
> 如果一个部件包含了一个独立的取指令单元，则该部件被称为核心(core)。一个RiscV兼容的核心能够通过多线程技术(或者说超线程技术)支持多个RiscV兼容硬件线程(harts)，harts这儿就是指硬件线程， hardware thread的意思。所谓超线程技术，就是在一个硬件核中，实现多份硬件线程，每个硬件线程都有自己独立的寄存器组等上下文资源，但大多数的运算资源都被所有线程复用，因此面积效率很高。超线程最早出现是在Intel的处理器中。  
> *—— 摘自 [Risc-V简要概括](https://www.cnblogs.com/mikewolf2002/p/11177988.html) ( [archive.is互联网存档](https://archive.is/nckyI) )*  
>
> 简而言之，硬件级线程和现在常见的 x86 处理器上的“m 核 n 线程”中的“n 线程”对应，若处理器设计时没有做超线程，那么一个核心就是一个硬件及线程，若有做超线程，那么某些核心可以有多个线程。

为什么 main 函数到最后是陷入死循环而不是退出呢？我们在讲完完整的启动流程后再来回顾这个问题。
