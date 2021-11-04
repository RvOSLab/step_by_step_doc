# 环境部署

这里我们主要安装几个上篇介绍过的开发环境，这里主要介绍 Ubuntu 环境下的配置安装过程，在 Ubuntu 18.04.5 LTS 以及 Ubuntu 20.04 LTS 上测试通过，其余发行版请根据自身情况安装相应软件包。

首先我们先来安装几个必要的软件包，其中一部分是用于编译 QEMU 和 GDB 的，这里也顺带安装了 `git`、`tmux`、`gcc-riscv64-linux-gnu` 等工具，其中`gcc-riscv64-linux-gnu`就是我们所用的交叉编译器。

```shell
sudo apt install -y build-essential gettext pkg-config libglib2.0-dev python3-dev libpixman-1-dev binutils libgtk-3-dev texinfo make gcc-riscv64-linux-gnu libncurses5-dev ninja-build tmux axel git
```

Ubuntu 的软件源中没有支持模拟RISC-V的QEMU和支持调试RISC-V的GDB，需要我们自行编译源码安装。下载 [QEMU 6.0.0](https://download.qemu.org/qemu-6.0.0.tar.xz) 和 [GDB 10.2](https://mirror.lzu.edu.cn/gnu/gdb/gdb-10.2.tar.xz) 的源码包。选择这两个版本也没有什么原因，就是系统编写时的最新稳定版罢了。

下载完成后解压即可：

```shell
tar xJf qemu-6.0.0.tar.xz
tar xJf gdb-10.2.tar.xz
```

进入解压后的 QEMU 源码目录，使用如下配置编译安装，这里配置了 QEMU 的模拟目标架构是32位和64位的 RISC-V，同时编译图形界面（使用 GTK 图形界面库，一个 Linux 下主流的图形界面库）。

```shell
./configure --target-list=riscv32-softmmu,riscv64-softmmu --enable-gtk
make -j$(nproc)
sudo make install
```

进入解压后的 GDB 源码目录，使用如下配置编译安装，配置了 GDB 的目标调试架构是 RISC-V 64位，同时编译 TUI 界面（文本用户界面）模式，添加 Python3 调试脚本支持。

```shell
./configure --target=riscv64-unknown-elf --enable-tui=yes -with-python=python3
make -j$(nproc)
sudo make install
```

这样安装就大致完成了，可以输入以下命令检测编译安装的软件的版本信息：

```shell
riscv64-unknown-elf-gdb -v
qemu-system-riscv64 --version
```

应该能看到这样的信息：

```
GNU gdb (GDB) 10.2
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

```
QEMU emulator version 6.0.0
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
```

执行 `qemu-system-riscv64 -machine virt -nographic`，可以看到如下输出：

```
OpenSBI v0.9
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : timer,mfdeleg
Platform HART Count       : 1
Firmware Base             : 0x80000000
Firmware Size             : 100 KB
Runtime SBI Version       : 0.2

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000080000000-0x000000008001ffff ()
Domain0 Region01          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x0000000000000000
Domain0 Next Arg1         : 0x0000000087000000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART ISA             : rv64imafdcsu
Boot HART Features        : scounteren,mcounteren,time
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 4
Boot HART PMP Address Bits: 54
Boot HART MHPM Count      : 0
Boot HART MHPM Count      : 0
Boot HART MIDELEG         : 0x0000000000000222
Boot HART MEDELEG         : 0x000000000000b109
```

表示QEMU可正常使用。

Qemu 可以使用 Ctrl + A 再按下 x 退出（注意要松开 Ctrl 和 A 再单独按x，这是一套组合按键，并不是只按 Ctrl + A 就可以的）。
