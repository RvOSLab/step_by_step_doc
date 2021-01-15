# 环境部署

这里我们主要安装几个上篇介绍过的开发环境，这里主要介绍 Ubuntu 环境下的配置安装过程，在 Ubuntu 18.04.5 LTS 以及 Linux Mint 20.1 上测试通过，其余发行版请根据自身情况安装相应软件包。

首先我们先来安装几个必要的软件包，其中一部分是用于编译 QEMU 和 GDB 的，这里也顺带安装了 `git`、`tmux`、`gcc-riscv64-linux-gnu` 等工具，其中`gcc-riscv64-linux-gnu`就是我们所用的交叉编译器。

```shell
sudo apt install -y git build-essential pkg-config libglib2.0-dev libpixman-1-dev binutils libgtk-3-dev texinfo make libncurses5-dev tmux gcc-riscv64-linux-gnu
```

Ubuntu 的软件源中没有支持模拟RISC-V的QEMU和支持调试RISC-V的GDB，需要我们自行编译源码安装。下载 [QEMU 5.1.0](https://gitee.com/Hanabichan/lzu-oslab-resource/attach_files/521696/download/qemu-5.1.0.tar.xz) 和 [GDB 10.1](https://gitee.com/Hanabichan/lzu-oslab-resource/attach_files/521695/download/gdb-10.1.tar.xz) 的源码包，可以选择官网下载，也可以下载前面给出的我们放在 gitee 上的源码包（国内下载稍快些）。选择这两个版本也没有什么原因，就是系统编写时的最新稳定版罢了，其中 QEMU 5.1.0 恰好也比 QEMU 5.0.0 多了OpenSBI中对 `HART_STATE_EXTENTION` 的支持。

下载完成后解压即可：

```shell
tar xJf qemu-5.1.0.tar.xz		#解压qemu包
tar xJf gdb-10.1.tar.xz			#解压gdb包
```

进入解压后的 QEMU 源码目录，使用如下配置编译安装，这里配置了 QEMU 的模拟目标架构是32位和64位的 RISC-V，同时编译 GTK 图形界面模式。

```shell
./configure --target-list=riscv32-softmmu,riscv64-softmmu --enable-gtk
make -j$(nproc)		#编译
sudo make install	#安装
```

进入解压后的 GDB 源码目录，使用如下配置编译安装，配置了 GDB 的目标调试架构是 RISC-V 64位，同时编译 TUI 界面（文本用户界面）模式。

```shell
./configure --target=riscv64-unknown-elf --enable-tui=yes
make -j$(nproc)
sudo make install	#安装
```

这样安装就大致完成了，可以输入以下命令检测编译安装的软件的版本信息：

```shell
riscv64-unknown-elf-gdb -v		# 打印gdb版本信息
qemu-system-riscv64 --version	# 打印qemu版本信息
```

应该能看到这样的信息：

```
QEMU emulator version 5.1.0
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
```

```
GNU gdb (GDB) 10.1
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

执行 `qemu-system-riscv64 -machine virt -nographic`，可以看到如下输出：

```
OpenSBI v0.7
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name          : QEMU Virt Machine
Platform HART Features : RV64ACDFIMSU
Current Hart           : 0
Firmware Base          : 0x80000000
Firmware Size          : 128 KB
Runtime SBI Version    : 0.2

MIDELEG : 0x0000000000000222
MEDELEG : 0x000000000000b109
PMP0    : 0x0000000080000000-0x000000008001ffff (A)
PMP1    : 0x0000000000000000-0xffffffffffffffff (A,R,W,X)
```

表示QEMU可正常使用。退出可以使用快捷键 Ctrl+A ，随后输入 x 。
