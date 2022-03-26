# 用户栈与内核栈

在进程运行在用户态时，发生的函数调用都是由进程内部自己产生的，因此函数的调用栈也是由进程自己管理，此时的栈上存储的内容与进程的活动有关。

当发生中断时，进程的活动将会被暂停，由操作系统内核接管完成操作，此时栈上存储的内容与内核的活动有关。因此在这两种不同的特权级下，一个进程需要有两个互不干扰的栈，以避免进程/内核对栈进行操作时影响内核/进程的状态。另外，同一个进程在内核态与用户态使用同一个页表，使用两段独立内存空间也避免了某一特权级对栈内存缺少访问权限。

切换栈的操作并不会由 CPU 自动完成，我们需要在特权级切换的时候一并完成栈的切换。在用户态发生特权级切换一定是因为发生了中断，因此我们在中断处理程序中对栈完成切换。在中断处理的过程中，有一个 `sscratch` 寄存器是 CPU 提供用于暂存一个字大小的数据的，因此我们可以将它拿来存放栈地址并指示当前处于何种特权级，我们规定：

- 处于 U 态时，`sscratch` 保存内核栈地址（`sp` 保存用户栈地址）
- 处于 S 态时，`sscratch` 为 0（`sp` 保存内核栈地址，用户栈地址存放于内核栈上）。

## 中断发生

我们在保存上下文之前不可随意修改通用寄存器，以免数据丢失。首先将 `sscratch` 与 `sp` 交换，随后通过 `sp` 的值判断中断发生前处于何种状态(注：CSR寄存器不可直接使用条件判断指令，需要读出至通用寄存器判断)。

- 若中断前是内核态，则将 `sscratch` 值（交换后为 `sp` 的值，即中断发生前的内核栈地址）重新读回 `sp`
  - 此时 `sp` 为内核栈地址，`sscratch` 也为内核栈地址
- 若中断前是用户态，不作处理
  - 此时 `sp` 为内核栈地址，`sscratch` 为用户栈地址

随后和前面的实验一样，扩张内核栈空间（模拟压入栈），保存所有通用寄存器的值至内核栈上，除了 x2（即sp）寄存器，sp 本应保存的是发生中断前的值，这个值目前被交换到了 `sscratch` 中，因此留到后面处理。

在保存完所有通用寄存器后，这些寄存器已可被随意使用，此时读取 `sscratch` 到 `s0`，再赋 `sscratch = 0`（在内核态中，需要确保 `sscratch` 为 0，借 `s0` 中转 `sscratch` 内的中断前栈地址存至内核栈上）。

`SAVE_ALL` 宏完整代码如下：

```assembly
# 定义宏：保存上下文（所有通用寄存器及额外的CSR寄存器）
# 我们规定：当 CPU 处于 U-Mode 时，sscratch 保存内核栈地址；处于 S-Mode 时，sscratch 为 0 。
.macro SAVE_ALL

    # 交换 sp 和 sscratch 寄存器
    csrrw sp, sscratch, sp

    # 可以通过sp是否为0判断原先是否为内核态
    # 如果中断来自用户态，此时 sp 已经指向内核栈，直接跳转到 trap_from_user 保存寄存器
    bnez sp, trap_from_user

# 否则为0，原本就是内核态，再次交换，继续往下执行，保存上下文
trap_from_kernel:

    # 将sscratch中的值读到sp中，此时 sscratch = 发生中断前的 sp（内核栈）
    csrr sp, sscratch

# 不为0，原本是用户态，不做交换，直接保存上下文
trap_from_user:
    # 将栈指针下移为TrapFrame预留足够的空间用于将所有通用寄存器值存入栈中（36个寄存器的空间）
    addi sp, sp, -36*XLENB
    # 保存所有通用寄存器，除了 x2 (x2就是sp，sp 本应保存的是发生中断前的值，这个值目前被交换到了 sscratch 中，因此留到后面处理。)
    STORE x1, 1
    STORE x3, 3
    STORE x4, 4
    STORE x5, 5
    STORE x6, 6
    STORE x7, 7
    STORE x8, 8
    STORE x9, 9
    STORE x10, 10
    STORE x11, 11
    STORE x12, 12
    STORE x13, 13
    STORE x14, 14
    STORE x15, 15
    STORE x16, 16
    STORE x17, 17
    STORE x18, 18
    STORE x19, 19
    STORE x20, 20
    STORE x21, 21
    STORE x22, 22
    STORE x23, 23
    STORE x24, 24
    STORE x25, 25
    STORE x26, 26
    STORE x27, 27
    STORE x28, 28
    STORE x29, 29
    STORE x30, 30
    STORE x31, 31

    # 保存完x0-x31之后这些寄存器就可以随意使用了，下面马上用到：
    # 读取sscratch到s0，赋 sscratch = 0（在内核态中，时刻保持sscratch为0，让s0存原有的sp）
    csrrw s0, sscratch, x0

    # 读取 sstatus, sepc, stval, scause寄存器的值到s0-s4（x1-x31中的特定几个）寄存器
    csrr s1, sstatus
    csrr s2, sepc
    csrr s3, stval
    csrr s4, scause

    # 存储 sp, sstatus, sepc, sbadvaddr, scause 到栈中
    # 其中把s0存到x2（sp）的位置以便于返回时直接恢复栈
    STORE s0, 2
    STORE s1, 32
    STORE s2, 33
    STORE s3, 34
    STORE s4, 35
.endm
```

## 中断恢复

在中断处理程序结束时，需要恢复上下文与特权级。首先从内存中读入中断发生时 `sstatus` 与 `sepc` 的值至通用寄存器。

我们可以借助 `sstatus.SPP` 是否为 1 来判断中断前的特权级，1为内核态，0为用户态。

- 若回到用户态，则需要将发生中断前的内核栈地址（`sp` 扩张前的值，模拟弹出栈）重新存入 `sscratch` 以符合我们的规定。
- 若回到内核态，则跳过这一步，保持值为 0。

随后从通用寄存器中将中断发生时 `sstatus` 与 `sepc` 的值读至 `sstatus` 与 `sepc` 寄存器。

恢复通用寄存器时因为 `LOAD` 宏使用了扩张后的栈地址 `sp`，因此同样将 `sp` 留到最后恢复，最后执行 `sret` 从内核态中断返回。

`RESTORE_ALL` 宏完整代码如下：

```assembly
# 定义宏：恢复寄存器
.macro RESTORE_ALL
    LOAD s1, 32             # s1 = sstatus
    LOAD s2, 33             # s2 = sepc
    andi s0, s1, 1 << 8     # 根据 sstatus.SPP 是否为 1 来判断中断前的特权级，1为内核态，0为用户态
    bnez s0, _to_kernel     # s0 = 是否会到内核态
# 若回到用户态
_to_user:
    addi s0, sp, 36*XLENB      # 计算出中断前的内核态sp，先存放在s0中
    csrw sscratch, s0         # 将s0中的内核态sp存入sscratch。根据规定，回到用户态后sscratch = 内核态sp
# 若回到内核态，跳过回到用户态的sscratch修改
_to_kernel:
    # 恢复 sstatus, sepc
    csrw sstatus, s1
    csrw sepc, s2

    # 恢复除了 x2 (sp) 以外的其余通用寄存器
    LOAD x1, 1
    LOAD x3, 3
    LOAD x4, 4
    LOAD x5, 5
    LOAD x6, 6
    LOAD x7, 7
    LOAD x8, 8
    LOAD x9, 9
    LOAD x10, 10
    LOAD x11, 11
    LOAD x12, 12
    LOAD x13, 13
    LOAD x14, 14
    LOAD x15, 15
    LOAD x16, 16
    LOAD x17, 17
    LOAD x18, 18
    LOAD x19, 19
    LOAD x20, 20
    LOAD x21, 21
    LOAD x22, 22
    LOAD x23, 23
    LOAD x24, 24
    LOAD x25, 25
    LOAD x26, 26
    LOAD x27, 27
    LOAD x28, 28
    LOAD x29, 29
    LOAD x30, 30
    LOAD x31, 31
    # 最后恢复栈指针(x2, sp)为原指针（无论是用户态还是内核态）
    LOAD x2, 2
.endm
```

思考：为什么保存上下文时不用 `sstatus.SPP` 判断中断前状态？

# 系统调用

从用户态主动进入内核态的方式就是通过系统调用进行。它在指令集层面的实现原理是在 U 态发出 `ecall` 指令，这时 CPU 会收到一个 `CAUSE_USER_ECALL` 异常，进入内核态处理异常。

系统调用可以有不定长的参数，参数可以使用寄存器传递，返回值也是用寄存器传递，具体可查看 [RISC-V ABI 规范](https://github.com/riscv-non-isa/riscv-elf-psabi-doc)，因此一个系统调用的入口函数可以写成这样：

```c
/**
 * @brief 通过系统调用号调用对应的系统调用
 *
 * @param number 系统调用号
 * @param ... 系统调用参数
 * @note 本实现中所有系统调用都仅在失败时返回负数，但实际上极小一部分 UNIX 系统调用（如
 *       `getpriority()`的正常返回值可能是负数的）。
 */
int64_t syscall(int64_t number, ...)
{
    va_list ap;
    va_start(ap, number);
    int64_t arg1 = va_arg(ap, int64_t);
    int64_t arg2 = va_arg(ap, int64_t);
    int64_t arg3 = va_arg(ap, int64_t);
    int64_t arg4 = va_arg(ap, int64_t);
    int64_t arg5 = va_arg(ap, int64_t);
    int64_t arg6 = va_arg(ap, int64_t);
    int64_t ret = 0;
    va_end(ap);
    if (number > 0 && number < NR_TASKS) {
        /* 小心寄存器变量被覆盖 */
        register int64_t a0 asm("a0") = arg1;
        register int64_t a1 asm("a1") = arg2;
        register int64_t a2 asm("a2") = arg3;
        register int64_t a3 asm("a3") = arg4;
        register int64_t a4 asm("a4") = arg5;
        register int64_t a5 asm("a5") = arg6;
        register int64_t a7 asm("a7") = number;
        __asm__ __volatile__ ("ecall\n\t"
                :"=r"(a0)
                :"r" (a1), "r" (a2), "r" (a3), "r" (a4), "r" (a5), "r" (a7)
                :"memory");
        ret = a0;
    } else {
        panic("Try to call unknown system call");
    }
    if (ret < 0) {
        errno = -ret;
        return -1;
    }
    return ret;
}
```

同样的，我们还需在中断处理程序中添加对 `CAUSE_USER_ECALL` 异常的处理，只需在原有的 `exception_handler` 函数内修改。可以直接调用一个新函数：

```c
case CAUSE_USER_ECALL:
    return syscall_handler(tf);
    break;
```

随后在 `syscall_handler` 函数中分派至不同的系统调用处理函数：

```c
/**
 * @brief 系统调用处理函数
 *
 * 检测系统调用号，调用响应的系统调用。
 * 当接收到错误的系统调用号时，设置 errno 为 ENOSY 并返回错误码 -1
 */
static struct trapframe* syscall_handler(struct trapframe* tf)
{
    uint64_t syscall_nr = tf->gpr.a7;
    if (syscall_nr >= NR_syscalls) {
        tf->gpr.a0 = -1;
        errno = ENOSYS;
    } else {
        tf->gpr.a0 = syscall_table[syscall_nr](tf);
    }
    tf->epc += INST_LEN(tf->epc); /* 执行下一条指令 */
    return tf;
}
```

# 初始化 0 号进程

我们需要首先尝试将特权级切换至 U 态（以下也成用户态，不做区分），这里我们采用伪造中断发生的方法来实现。

我们假设，我们正运行在用户态，`sscratch` 按照规定设置为内核栈地址。此时通过 `ecall` 指令**陷入内核态**请求一个系统调用，这时候 CPU 会自动将部分寄存器改写：

- scause: 被设置成 CAUSE_USER_ECALL
- sstatus: 
  - SPP 位设为发生异常之前的权限模式（0，表示 U 态）
  - SIE 位置 0 以禁用中断
  - SPIE 位被设置为原 SIE 位的值（1，开）
- sepc: 被设置为异常发生时的 pc 值
- pc: 被设置为 stvec(`__alltraps`)

因此我们要想**伪造**一个从用户态发出的系统调用，我们就需要伪造上述 CSR 的变化。此外，由于系统调用至少需要一个参数提供调用号，而我们不是通过真正执行 `syscall` 函数进入系统调用的，因此我们也需要伪造这个参数。

我们可以直接将值写入各寄存器后用 call 指令跳转至 `__alltraps` 处，避免函数调用影响栈。

整个 `init_task0` 函数都是在伪造这一过程：

```c
/**
 * @brief 初始化进程 0
 * 手动进入中断处理，调用 sys_init() 初始化进程0
 * @see sys_init()
 * @note 这个宏使用了以下三个 GNU C 拓展：
 *       - [Locally Declared Labels](https://gcc.gnu.org/onlinedocs/gcc/Local-Labels.html)
 *       - [Labels as Values](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)
 *       - [Statements and Declarations in Expressions](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html#Statement-Exprs)
 */
#define init_task0()                                                        \
({                                                                          \
    __label__ ret;                                                          \
    write_csr(scause, CAUSE_USER_ECALL);                                    \
    clear_csr(sstatus, SSTATUS_SPP);                                        \
    set_csr(sstatus, SSTATUS_SPIE);                                         \
    clear_csr(sstatus, SSTATUS_SIE);                                        \
    write_csr(sepc, &&ret - 4 - (SBI_END + LINEAR_OFFSET - START_CODE));    \
    write_csr(sscratch, (char*)&init_task + PAGE_SIZE);                     \
    register uint64_t a7 asm("a7") = 0;                                     \
    __asm__ __volatile__("call __alltraps \n\t" ::"r"(a7):"memory");        \
    ret: ;                                                                  \
})
```

其中较为复杂的部分是 `sepc` 和 `sscratch` 值的确定。

思考：为什么 `init_task0` 要用宏实现？

因为需要避免函数调用返回时弹栈（此时用户栈是空的）

# 附录

GCC提供一些标准 C 不支持的额外语法与功能，本节附录主要介绍代码中使用到的三个 GNU C 的拓展。

## Locally Declared Labels 扩展

**局部声明的标签**是作用域仅在*当前块及嵌套的块、嵌套的函数*内的标签，声明方式为`__label__ label1, label2, /* … */;`。局部标签的声明必须在代码块的开头所有其他普通声明与语句之前。局部标签的声明只声明其名称，具体的定义依然需要靠`label:`来实现。

在较为复杂的宏中局部标签使用 `goto` 跳出循环或获取某一地址，用局部标签会更方便，可以避免多次引用这一宏引发的标签重复定义的问题。

使用示例见 `sched.h` 中 `init_task0` 宏的定义，其中使用 `__label__ ret;` 声明了局部标签`ret`，并在 `write_csr(sepc, &&ret - 4 - (SBI_END + LINEAR_OFFSET - START_CODE));` 中使用了它的地址，而 `ret` 的具体定义在 `ret: ;` 处。

## Labels as Values 扩展

对于当前函数中定义的标签，可以使用 `&&` 运算符获取其地址，获取到的值的类型为 `void *`，例如：

```c
void *ptr;
/* … */
ptr = &&foo;
```

同时你可以用 `goto *ptr;` 来跳转至该标签所代表的地址。

注意：跨函数使用这一方法跳转会导致不可预知的后果。

在 `inline` 函数或 `clone` 函数中，对同一个标签获取到的值可能不同，如果需要保证它们相同，需要使用 `__attribute__((__noinline__,__noclone__))` 以避免函数被作为 `inline` 函数或 `clone` 函数。如果在静态变量初始值设定中使用 `&&foo`，则禁止 `inline` 或 `clone`。

## Statements and Declarations in Expressions 扩展

语句表达式：用括号括起来的复合语句可能在 GNU C 中被当做表达式。因此可以在表达式中使用循环、switch和局部变量。

复合语句是指由大括号包围的一系列语句，因此在上述结构中，圆括号可以包含大括号，例如：

```c
({ int y = foo();
   int z;
   if (y > 0) z = y;
   else z = - y;
   z; })
```

其中复合语句的最后一句应该是一个表达式，其返回值作为整个结构的值，否则返回类型为 `void`，并且没有返回值。

这一特性使其在确保宏定义的“安全”中较为有用，可用于确保操作数只运算一次。例如在`#define max(a,b) ((a) > (b) ? (a) : (b))`的定义中，a 和 b 会被计算两次，如果调用它的运算符有副作用（如++运算符），可能会导致错误的结果，在 GNU C 中，如果知道操作数的类型（这里取`int`），可以通过如下定义宏来避免此问题：

```c
#define maxint(a,b) \
  ({int _a = (a), _b = (b); _a > _b ? _a : _b; })
```

比如在如下示例中，`max` 宏会导致a、b自增两次，而 `maxint` 宏则不会：

```c
int a = 0;
int b = 1;
int c = max(a++, b++);
// int c = ((a++) > (b++) ? (a++) : (b++));
int d = maxint(a++, b++);
// int d = ({
//          int _a = (a++), _b = (b++);
//          _a > _b ? _a : _b;
//          });
```

---

请注意，引入变量声明（例如在 `maxint` 中）可能会和“主调”函数中原有的变量相互影响产生错误，例如在以下示例中

```c
int _a = 1, _b = 2;
int c = max(_a, _b);
// int c = ((_a) > (_b) ? (_a) : (_b));
int d = maxint(_a, _b);
// int d = ({int _a = (_a), _b = (_b); _a > _b ? _a : _b; });
// 展开后变量定义会导致错误
```

当我们像下例中递归地使用此模式时，也可能会发生此问题：

```c
#define maxint3(a, b, c) \
  ({int _a = (a), _b = (b), _c = (c); maxint (maxint (_a, _b), _c); })
```

---

常量表达式（如枚举常量的值、位域的宽度或静态变量的初始值）中不允许嵌入语句。

如果不知道操作数的类型，就需要使用 `typeof` 或 `__auto_type`。
