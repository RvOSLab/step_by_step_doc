# 准备

工欲善其事，必先利其器。在这一章我们会准备好敲代码、测试和调试的整个环境。

敲代码的工具自然是可以自由选择，作为这个文档的编写者我推荐使用 Visual Studio Code ，微软开发的跨平台编辑器，装上几个插件可以说是非常好用了。

运行的环境自然也是可以选择自己喜欢的环境，我比较喜欢 [Ubuntu](https://zh.wikipedia.org/wiki/Ubuntu) 系的发行版，因此环境配置说明我就选择了基于 Ubuntu 平台的配置方法，其他的 Linux 发行版也可以自由选择，不要太古老就好，比如 [Kong Jun](https://github.com/kongjun18) 大佬喜欢使用他的 [Fedora](https://zh.wikipedia.org/wiki/Fedora_(%E4%BD%9C%E6%A5%AD%E7%B3%BB%E7%B5%B1)) 系统。另外喜欢使用 Windows 的同学更推荐使用 [WSL](https://docs.microsoft.com/zh-cn/windows/wsl/)。

> **小知识：适用于 Linux 的 Windows 子系统（Windows Subsystem for Linux，WSL）**  
> 
> 适用于 Linux 的 Windows 子系统（即 WSL）可让开发人员按原样运行 GNU/Linux 环境 - 包括大多数命令行工具、实用工具和应用程序 - 且不会产生传统虚拟机或双启动设置开销。  
> WSL 2 是适用于 Linux 的 Windows 子系统体系结构的一个新版本，它支持适用于 Linux 的 Windows 子系统在 Windows 上运行 ELF64 Linux 二进制文件。 它的主要目标是提高文件系统性能，以及添加完全的系统调用兼容性。  
> *—— [关于适用于 Linux 的 Windows 子系统 | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows/wsl/about)*  
> 注：上文中提到的 GNU/Linux 就是 Linux
> 
> 简单来说，Windows 和 Linux 的运行环境、系统调用、可执行文件的格式等都是不同的，对于习惯于 Windows 系统使用的我们来说，偶尔要使用一次 Linux，装一个 Linux 系统或者建一个 Linux 虚拟机似乎没有很大的必要，就可以使用 WSL 在 Windows 下运行 Linux 的程序——没错，我们的项目所用到的都是 Linux 下的程序。

那么，下面就让我们开始吧。
