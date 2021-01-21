# 准备

工欲善其事，必先利其器。在这一章我们会准备好敲代码、测试和调试的整个环境。

敲代码的工具自然是可以自由选择，作为这个文档的编写者我推荐使用 Visual Studio Code ，微软开发的跨平台编辑器，装上几个插件可以说是非常好用了，这个系统的另几位编写者则有其他的编辑器使用，比如 [Kong Jun](https://github.com/kongjun18) 大佬就使用了他最趁手的 Neovim 代码编辑器。

> **小知识：vi 编辑器家族**  
> 
> vi是一种计算机文本编辑器，由美国计算机科学家比尔·乔伊（Bill Joy）完成编写，并于1976年以BSD协议授权发布。  
> *—— [vi - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/Vi)*  
> 
> Vim是从vi发展出来的一个文本编辑器。其代码补完、编译及错误跳转等方便编程的功能特别丰富，在程序员中被广泛使用。和Emacs并列成为类Unix系统用户最喜欢的编辑器。  
> Vim的第一个版本由布莱姆·米勒在1991年发布。  
> 对于大多数用户来说，Vim有着一个比较陡峭的学习曲线。这意味着开始学习的时候可能会进展缓慢，但是一旦掌握一些基本操作之后，能大幅度提高编辑效率。  
> 
> Neovim是Vim的一个重构版本，致力于成为Vim的超集（superset）。Neovim和Vim配置文件采用相同的语法，所以Vim的配置文件也可以用于Neovim。Neovim的第一个版本在2015年12月发行，并且能够完全兼容Vim的特性。  
> Neovim项目从2014年发起，有许多来自Vim社区的开源开发者为其提供早期支持，包括更好的脚本支持、插件以及和更好地融合图形界面等。  
> 相比于Vim，Neovim的主要改进在于其支持异步加载插件。此外，Neovim的插件可以用任意语言编写，而Vim的插件仅能使用Vimscript进行编写。  
> *—— [Vim - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/Vim)*  
> 
> 更多资料：[Vim 和 Neovim 的前世今生 - jdhao's blog](https://jdhao.github.io/2020/01/12/vim_nvim_history_development/)

运行的环境自然也是可以选择自己喜欢的环境，我比较喜欢 [Ubuntu](https://zh.wikipedia.org/wiki/Ubuntu) 系的发行版，因此环境配置说明我就选择了基于 Ubuntu 平台的配置方法，其他的 Linux 发行版也可以自由选择，不要太古老就好，比如 [Kong Jun](https://github.com/kongjun18) 大佬喜欢使用他的 [Fedora](https://zh.wikipedia.org/wiki/Fedora_(%E4%BD%9C%E6%A5%AD%E7%B3%BB%E7%B5%B1)) 系统。另外喜欢使用 Windows 的同学也可以尝试使用 [WSL](https://docs.microsoft.com/zh-cn/windows/wsl/)。

> **小知识：适用于 Linux 的 Windows 子系统（Windows Subsystem for Linux，WSL）**  
> 
> 适用于 Linux 的 Windows 子系统（即 WSL）可让开发人员按原样运行 GNU/Linux 环境 - 包括大多数命令行工具、实用工具和应用程序 - 且不会产生传统虚拟机或双启动设置开销。  
> WSL 2 是适用于 Linux 的 Windows 子系统体系结构的一个新版本，它支持适用于 Linux 的 Windows 子系统在 Windows 上运行 ELF64 Linux 二进制文件。 它的主要目标是提高文件系统性能，以及添加完全的系统调用兼容性。  
> *—— [关于适用于 Linux 的 Windows 子系统 | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows/wsl/about)*  
> 注：上文中提到的 GNU/Linux 就是 Linux
> 
> 简单来说，Windows 和 Linux 的运行环境、系统调用、可执行文件的格式等都是不同的，对于习惯于 Windows 系统使用的我们来说，偶尔要使用一次 Linux，装一个 Linux 系统或者建一个 Linux 虚拟机似乎没有很大的必要，就可以使用 WSL 在 Windows 下运行 Linux 的程序——没错，我们的项目所用到的都是 Linux 下的程序。

那么，下面就让我们开始吧。
