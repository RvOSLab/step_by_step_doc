# 驱动 QEMU 的 RTC

## 介绍

## 寄存器数据结构设计

在这里，我们使用结构体 `struct goldfish_rtc_regs` 来表示这些寄存器偏移，使结构体成员变量在内存中的布局与硬件端口相对起始地址的偏移的布局一致。随后将硬件的起始地址强制类型转换后赋值给 `struct goldfish_rtc_regs *` 结构体指针的一个变量，这样就可以通过结构体中每个成员变量的偏移来表示硬件的对应端口了。

## 程序设计
