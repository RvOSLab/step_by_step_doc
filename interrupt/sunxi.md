# 移植至哪吒 D1

哪吒 D1 使用了全套全志的 sunxi 硬件设备，关于哪吒 D1 所用的硬件设备，我们都需要参考全志官网给出的*D1_User_Manual_V0.1(Draft Version)*。

D1 的中断控制器采用与 QEMU 相同的 clint + plic 的设计，因此移植工作量就少了很多。

