# 附录：链接脚本 Linker Script

本文根据[文档 Linker Script](https://sourceware.org/binutils/docs/ld/Scripts.html) 翻译而成。

链接脚本（Linker Script）是用于控制链接器 `ld` 执行的命令脚本，主要用于描述多个输入的目标文件如何被映射到一个输出的目标文件中，并控制输出文件的内存布局。如果没有指定，则会使用自带的链接脚本来生成可执行文件。由于我们需要手动指定我们自己的操作系统内存布局，因此需要编写一个链接脚本。

在使用时，需要给 `ld` 加上 `-T` 参数以指定链接脚本。

## 基本概念

链接器将输入文件合并为单个输出文件。

- 目标文件格式*(object file format)*：每个输入文件和输出文件的一种特殊的数据格式。
- **目标文件*(object file)***：每个文件都称为目标文件。其中输出文件通常也被称为可执行文件。

每个目标文件都有一个列表记录了目标文件的各个**节*(sections)***。

- 输入节*(input section)*：输入文件中的一个节。
- 输出节*(output section)*：输出文件中的一个节。

目标文件中的每个节都有一个名称和大小。大多数节还有一个相关联的数据块，称为节内容*(section contents)*。节可以标记为可加载的*(loadable)*，这意味着在运行输出文件时，节内容应该加载到内存中。没有内容的节可能是可分配的*(allocatable)*，这意味着应在内存中预留一个区域，但是无需填写内容(在某些情况下须填零)。既不可加载也不可分配的节通常包含某种调试信息。

每个可加载或可分配的输出节有两个地址。第一个是 VMA，即虚拟内存地址(virtual memory address)。这是该节在运行输出文件时将拥有的地址。第二个是 LMA，即加载内存地址(load memory address)。这是将要加载节的地址。在大多数情况下，两个地址是相同的。

通过使用带有`-h`选项的`objdump`程序，可以查看目标文件中的节。

每个目标文件都有一个**符号*(symbol)***列表，称为**符号表*(symbol table)***。符号可以是已定义的，也可以是未定义的。每个符号都有一个名称，每个**已定义**的符号都有一个地址。如果将C或C++程序编译成目标文件，则每个已定义的函数和全局变量或静态变量对应一个已定义的符号。输入文件中引用的每个未定义的函数或全局变量都将成为未定义的符号。

可以通过使用 `nm` 程序或使用带有 `-t` 选项的 `objdump` 程序来查看目标文件中的符号。

## 链接脚本格式

它是一个含有一系列命令的文本文件，每个命令是一个可有参数的关键字，或对一个符号赋值。命令可用分号分隔，空格将被忽略。

通常可以直接输入文件名或格式名等字符串。如果文件名包含分隔符（如逗号），则可以将文件名放在双引号中。文件名中不得有双引号。

链接脚本的注释由`/*`和`*/`包围。

## 简单示例

最简单的链接脚本只有一个命令：`SECTIONS`。您可以使用`SECTIONS`命令来描述输出文件的内存布局。

`SECTIONS`命令是一个强大的命令。假设程序只包含代码、已初始化的数据和未初始化的数据。这些分别在 `.text`, `.data` 与 `.bss` 节中，并进一步假设输入文件中仅有这些节。

对于本例，假设代码应该加载到地址`0x10000`，已初始化的数据和未初始化的数据应该从地址`0x8000000`开始并相连。相应的链接脚本如下：

```linker-script
SECTIONS
{
  . = 0x10000;
  .text : { *(.text) }
  . = 0x8000000;
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```

`SECTIONS`命令以关键字`SECTIONS`开始，然后是一系列符号赋值和用大括号括起来的输出节描述。

上例中`SECTIONS`命令的第一行用于设置特殊符号`.`的值，这是位置计数器。如果不以其他方式指定输出节的地址（其他方式将在后面介绍），则将根据位置计数器的当前值设置地址。然后，位置计数器按输出节的大小递增。若不额外设置，在`SECTIONS`命令的开头，位置计数器的值为`0`。

第二行定义了一个输出节`.text`，其中冒号是必需的语法，目前可忽略。在输出节名称后的大括号内，列出了应放入此输出节的输入节的名称。`*` 是一个与任何文件名匹配的通配符。`*(.text)`表示所有输入文件中的`.text`输入节。

由于定义输出节`.text`时，位置计数器为`0x10000`，链接器将设置输出文件中的`.text`节的地址为`0x10000`。

剩余的行定义输出文件中的`.data`和`.bss`节。链接器将在`0x8000000`地址处放置`.data`输出节。链接器放置`.data`输出节后，位置计数器的值将为 $  `0x8000000`+`.data`输出节的大小 $ ，因此链接器将在`.data`后立即放置`.bss`输出节。

链接器将确保每个输出节都具有所需的对齐方式，必要时增加位置计数器。在本例中，`.text`和`.data`节的指定地址满足任何对齐约束，但链接器可能必须在`.data`和`.bss`节之间创建一个小的间隙以满足部分对齐要求。

## 简单命令

### 入口点

在程序中执行的第一条指令称为**入口点*(Entry Point)***。您可以使用`ENTRY`命令设置入口点，参数是一个符号名称：

`ENTRY(symbol)`

有几种方法可以设置入口点。链接器将通过按顺序（即优先级从高到低）尝试以下每个方法来设置入口点直至成功：

- 命令行通过 `-e` 传参;
- 链接脚本中的`ENTRY(symbol)`命令;
- 平台相关的符号的值（若有）。对于多数平台来说是 `start`，但基于PE和BeOS的系统会检查可能的入口符号列表，匹配找到的第一个。
- 代码节的第一个字节的地址，如果存在并且正在创建一个可执行文件。代码节通常是`.text`，也可以是其他名称;
- 地址 `0`。

### 文件命令

```linker-script
INCLUDE filename
```

在该命令处调用另一个链接脚本。

将在当前目录以及使用`-L`选项指定的目录中搜索该文件。嵌套调用最多10级。

`INCLUDE`指令可放在顶层、`MEMORY`或`SECTIONS`命令中，或放在输出节的描述中。

```linker-script
INPUT(file, file, …)
INPUT(file file …)
```

`INPUT` 命令表示在链接中包含这些文件，就像在命令行参数中被指定一样。

例如总希望在链接时包含 `subr.o`，但是无法将其放在每个链接命令的参数中，此时就可以在链接脚本中加入`INPUT (subr.o)`。

实际上，甚至可以在链接脚本中列出所有输入文件，然后使用`-t`选项指定链接脚本而不在参数中引用任何输入文件。

如果配置了 `sysroot prefix` 目录，并且文件名以 `/` 字符开头，并且正在处理的脚本位于 `sysroot prefix` 目录，则将在 `sysroot prefix` 目录中查找文件。也可以让文件名路径以`=`开头，或者使用`$SYSROOT`作为文件名路径的前缀来强制使用 `sysroot prefix`。另请参阅[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)中`-L`参数的描述。

如果未使用 `sysroot prefix`，则链接器将尝试在包含链接脚本的目录中打开文件。若未找到，将搜索当前目录。如果仍未找到，链接器将在库的搜索路径中查找。

如果使用 `INPUT (-lfile)`，`ld` 会将名称转换为 `libfile.a`，与命令行参数`-l`一样。

当您在隐式链接脚本中使用`INPUT`命令时，文件将在链接脚本被引用时才被加入。这可能会影响库的搜索。

```linker-script
GROUP(file, file, …)
GROUP(file file …)
```

`GROUP` 命令与 `INPUT` 类似，只是被指定的文件都应该是库文件，并且要反复搜索这些文件，直到没有创建新的未定义的引用。请参见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)中 `-(` 的说明。

```linker-script
AS_NEEDED(file, file, …)
AS_NEEDED(file file …)
```

这个结构只能出现在 `INPUT` 或 `GROUP` 命令以及其他文件名中。列出的文件将像直接出现在 `INPUT` 或 `GROUP` 命令中一样进行处理，但 ELF 共享库除外，它们只有在实际需要时才添加这些文件。这个结构实质上为其中列出的所有文件启用了 `-as-needed` 选项，为了恢复以前编译环境，之后需设置 `--no-as-needed`。

```linker-script
OUTPUT(filename)
```

`OUTPUT` 命令设置输出文件的文件名。在链接脚本中使用`OUTPUT(filename)`与在命令行中使用`-o filename`完全相同（请参见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)），且命令行选项优先。

可以使用`OUTPUT`命令设置默认输出文件名，而不是通常的默认名称`a.out`。

```linker-script
SEARCH_DIR(path)
```

`SEARCH_DIR`命令将`path`添加到`ld`查找库的路径列表中。使用`SEARCH_DIR(path)`与在命令行上使用`-L path`完全相同（请参见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）。如果两者都使用，那么链接器将搜索两者的路径。首先搜索使用命令行选项指定的路径。

```linker-script
STARTUP(filename)
```

`STARTUP` 命令与 `INPUT` 命令类似，只是 `filename` 将成为第一个要链接的输入文件，就好像它是在命令行中首先指定的一样。在使用那些入口点始终是第一个文件的起始处的系统时将会很有用。

### 格式命令

```linker-script
OUTPUT_FORMAT(bfdname)
OUTPUT_FORMAT(default, big, little)
```

`OUTPUT_FORMAT` 命令指定用于输出文件的 BFD 格式（请参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)）。使用 `OUTPUT_FORMAT(bfdname)` 与在命令行上使用 `--oformat bfdname` 完全相同（请参见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)），且命令行选项优先。

根据`-EB`和`-EL`命令行选项，可以使用带有三个参数的 `OUTPUT_FORMAT` 来使用不同的格式。这允许链接脚本根据所需的大小端设置输出格式。

如果既不使用`-EB`也不使用`-EL`，则输出格式将是第一个参数（`default`）。如果使用`-EB`，输出格式将是第二个参数(`big`)。如果使用`-EL`，则输出格式将是第三个参数(`little`)。

例如，MIPS ELF 目标平台的默认链接脚本使用以下命令：

`OUTPUT_FORMAT(elf32-bigmips, elf32-bigmips, elf32-littlemips)`

这表示输出文件的默认格式为`elf32-bigmips`，但如果用户使用`-EL`命令行选项，则输出文件将以 `elf32-littlemips` 格式创建。


```linker-script
TARGET(bfdname)
```

`TARGET` 命令指定读取输入文件时使用的 BFD 格式。它影响后续的 `INPUT` 和 `GROUP` 命令。这个命令类似于在命令行上使用`-b bfdname`（请参见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）。如果使用 `TARGET` 命令而不使用 `OUTPUT_FORMAT`，那么最后一个 `TARGET` 命令也用于设置输出文件的格式。参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)。

### REGION_ALIAS

为内存区域指定别名。

别名可以添加到使用 MEMORY 命令创建的现有内存区域中。每个名称最多对应一个内存区域。

`REGION_ALIAS(alias, region)`

`REGION_ALIAS` 函数为内存区域 `region` 创建别名 `alias`。可将输出节灵活地映射到内存区域。

一个简单的例子：

假设我们有一个嵌入式系统的应用程序，它带有各种内存存储设备。它们都有一个通用的易失性内存 RAM，可以执行代码或存储数据。有些可能有一个只读的，非易失的 ROM，允许代码执行和只读数据访问。最后一种是只读的、非挥发性记忆体的 ROM2，具有只读数据访问，但无代码执行能力。我们有四个输出节:

```
.text 程序代码;
.rodata 只读数据;
.data 可写的已初始化的数据;
.bss 可写的填零的数据.
```

我们的目标是提供一个链接器命令文件，其中包含一个定义输出节的系统无关部分，以及一个将输出节映射到系统上可用内存区域的系统相关部分。我们的嵌入式系统有三种不同的内存设置A、B和C：

| Section | Variant A | Variant B | Variant C |
| ------- | --------- | --------- | --------- |
| .text   | RAM       | ROM       | ROM       |
| .rodata | RAM       | ROM       | ROM2      |
| .data   | RAM       | RAM/ROM   | RAM/ROM2  |
| .bss    | RAM       | RAM       | RAM       |

符号 RAM/ROM 或 RAM/ROM2 表示该节分别加载到区域 ROM 或 ROM2 中。请注意，`.data` 的加载地址在三个不同的方案中都在 `.rodata` 的结尾处开始。

下面是处理输出节的基本链接脚本。它包括依赖于系统的描述内存布局的 `linkcmds.memory` 文件：

```linker-script
INCLUDE linkcmds.memory

SECTIONS
  {
    .text :
      {
        *(.text)
      } > REGION_TEXT
    .rodata :
      {
        *(.rodata)
        rodata_end = .;
      } > REGION_RODATA
    .data : AT (rodata_end)
      {
        data_start = .;
        *(.data)
      } > REGION_DATA
    data_size = SIZEOF(.data);
    data_load_start = LOADADDR(.data);
    .bss :
      {
        *(.bss)
      } > REGION_BSS
  }
```

现在我们需要三个不同的 `linkcmds.memory` 文件来定义内存区域和别名名称。对于这三种方案，`linkcmds.memory` 的内容分别是：

A：全都放入 RAM

```linker-script
MEMORY
  {
    RAM : ORIGIN = 0, LENGTH = 4M
  }

REGION_ALIAS("REGION_TEXT", RAM);
REGION_ALIAS("REGION_RODATA", RAM);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

B：代码和只读数据放入 ROM，可写数据放入 RAM。已初始化数据的映像被加载到 ROM 中，并在系统启动期间被复制到 RAM 中。

```linker-script

MEMORY
  {
    ROM : ORIGIN = 0, LENGTH = 3M
    RAM : ORIGIN = 0x10000000, LENGTH = 1M
  }

REGION_ALIAS("REGION_TEXT", ROM);
REGION_ALIAS("REGION_RODATA", ROM);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

C：代码放入 ROM，只读数据放入 ROM2，可写数据放入 RAM。已初始化数据的映像被加载到 ROM2 中，并在系统启动期间被复制到 RAM 中。

```linker-script

MEMORY
  {
    ROM : ORIGIN = 0, LENGTH = 2M
    ROM2 : ORIGIN = 0x10000000, LENGTH = 1M
    RAM : ORIGIN = 0x20000000, LENGTH = 1M
  }

REGION_ALIAS("REGION_TEXT", ROM);
REGION_ALIAS("REGION_RODATA", ROM2);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

如果需要，可以编写通用的系统初始化例程以将来自 ROM 或 ROM2 的 `.data` 节复制到RAM中：

```c
#include <string.h>

extern char data_start [];
extern char data_size [];
extern char data_load_start [];

void copy_data(void)
{
  if (data_start != data_load_start)
    {
      memcpy(data_start, data_load_start, (size_t) data_size);
    }
}
```

### 其他命令

其他命令有：

```linker-script
ASSERT(exp, message)
EXTERN(symbol symbol …)
FORCE_COMMON_ALLOCATION
INHIBIT_COMMON_ALLOCATION
FORCE_GROUP_ALLOCATION
INSERT [ AFTER | BEFORE ] output_section
NOCROSSREFS(section section …)
NOCROSSREFS_TO(tosection fromsection …)
OUTPUT_ARCH(bfdarch)
LD_FEATURE(string)
```

其中我们项目中只用到了`OUTPUT_ARCH(bfdarch)`，因此不对其余命令做介绍，有兴趣可自行阅读文档。

`OUTPUT_ARCH(bfdarch)` 用于指定特定的输出体系结构。参数是 BFD 库使用的名称之一（请参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)）。使用 `objdump` 程序加 `-f` 参数可以看到目标文件的体系结构。

## 为符号赋值

可以在链接脚本中为符号赋值。这将定义符号并将其放置到具有全局作用域的符号表中。

### 简单赋值

可以使用C语言中的任何给符号赋值的操作符，末位必须有分号：

```linker-script
symbol = expression ;
symbol += expression ;
symbol -= expression ;
symbol *= expression ;
symbol /= expression ;
symbol <<= expression ;
symbol >>= expression ;
symbol &= expression ;
symbol |= expression ;
```

特殊的符号名称 `.` 表示位置计数器。您只能在 `SECTIONS` 命令中使用此符号。见[位置计数器](https://sourceware.org/binutils/docs/ld/Location-Counter.html)。

表达式在下面定义（请[参阅表达式](https://sourceware.org/binutils/docs/ld/Expressions.html)）。

符号赋值可以作为命令本身编写，或者作为 `SECTIONS` 命令中的语句编写，或者作为 `SECTIONS` 命令中的输出节描述的一部分。

下面的例子展示了三个可以使用符号赋值的不同地方:

```linker-script
floating_point = 0;
SECTIONS
{
  .text :
    {
      *(.text)
      _etext = .;
    }
  _bdata = (. + 3) & ~ 3;
  .data : { *(.data) }
}
```

在此示例中，符号`floating_point`将被定义为 0。符号`_etext`将被定义为最后一个`.text`输入节结束之后的地址。符号`_bdata`将被定义为`.text`输出节结束之后的地址向上对齐至4字节边界。

除此之外，还有 `HIDDEN`、`PROVIDE`、`PROVIDE_HIDDEN` 命令可供选用，项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Assignments.html)自行理解。

### 在程序源代码中引用链接脚本符号

从程序源代码中访问链接脚本定义的变量并不直观。尤其是链接脚本符号并不等同于高级语言中的变量声明，而是一个没有值的符号。

请注意当编译器将源代码中的名称存储在符号表中时，它们通常会将变量转换为不同的名称。例如，Fortran 编译器通常会添加下划线，而 C++ 会执行广泛的重整(name mangling)。因此，在源代码中使用的变量的名称和与链接脚本中定义相同的变量的名称可能存在差异。例如，在 C 中，链接脚本变量可能被引用为：

`extern int foo;`

但在链接脚本中它可能被定义为:

`_foo = 1000;`

假设没有发生名称转换，当用高级语言（如C）声明一个符号时，编译器首先会在程序内存中预留空间来保存符号的值，然后在符号表中创建条目来保存符号的地址。例如：

`int foo = 1000;`

编译器会在符号表中创建一个名为`foo`的条目，保存了存储数字 1000 的`int`大小的内存块的地址。

当程序引用符号时，编译器生成的代码会首先访问符号表以查找符号的地址，然后读取该地址所存的值。所以 `foo = 1;` 查找符号表中的符号`foo`，获取关联的地址，然后将值 1 写入该地址。

而 `int * a = & foo;` 查找符号表中的符号`foo`，获取其地址，然后将此地址复制到与变量`a`关联的内存块中（即赋值给`a`）。

相比之下，链接脚本符号声明命令会在符号表中创建一个条目，但不为它们分配任何内存。因此，它们是一个没有值的地址。例如，链接脚本定义 `foo = 1000;` 将会在符号表中创建一个名为`foo`的条目，该条目保存内存位置1000的地址，但在地址1000处未存储任何特殊内容。这意味着无法访问链接脚本定义的符号的值（它没有值），所能做的就是访问链接脚本定义的符号的地址。

因此，当在源代码中使用链接脚本定义的符号时，应该始终获取符号的地址，而不要使用其值。例如要将`.ROM`节的内容复制到`.FLASH`，且链接脚本中包含这些声明：

```linker-script
start_of_ROM   = .ROM;
end_of_ROM     = .ROM + sizeof (.ROM);
start_of_FLASH = .FLASH;
```

则复制内容的C语言代码应这样写，注意其中的`&`。：

```c
extern char start_of_ROM, end_of_ROM, start_of_FLASH;

memcpy (& start_of_FLASH, & start_of_ROM, & end_of_ROM - & start_of_ROM);
```

或者使用数组表示法：

```c
extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];

memcpy (start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);
```

## SECTIONS 命令

`SECTIONS` 命令告诉链接器如何将输入节映射到输出节，以及如何将输出节放入内存。

`SECTIONS` 命令的格式为：

```linker-script
SECTIONS
{
  sections-command
  sections-command
  …
}
```

每个 `sections-command` 可以是以下命令之一：

- `ENTRY` 命令（参见[输入命令](https://sourceware.org/binutils/docs/ld/Entry-Point.html)）
- 符号赋值（请参见[赋值](#为符号赋值)）
- 输出节描述
- 覆盖描述

为了方便在这些命令中使用位置计数器，允许在 `SECTIONS` 命令中使用 `ENTRY` 命令和符号赋值。这还可以使链接脚本更容易理解，因为在输出文件布局中有意义的地方可以使用这些命令。

输出节描述和覆盖描述如下：

如果在链接脚本中不使用 `SECTIONS` 命令，链接器将按照在输入文件中首先遇到的顺序，将每个输入节放入一个同名的输出节中。例如，如果第一个文件中存在所有输入节，则输出文件中节的顺序将与第一个输入文件中的顺序匹配。第一个节将在地址 0 处。

### 输出节描述

一个完整的输出节描述如下：

```linker-script
section [address] [(type)] :
  [AT(lma)]
  [ALIGN(section_align) | ALIGN_WITH_INPUT]
  [SUBALIGN(subsection_align)]
  [constraint]
  {
    output-section-command
    output-section-command
    …
  } [>region] [AT>lma_region] [:phdr :phdr …] [=fillexp] [,]
```

大多数节属性并不会被广泛使用。

`section` 是节的名字，必须满足输出的目标文件的格式限制。在仅支持有限数量节的格式中，名称必须是该格式支持的名称之一（例如，`a.out` 格式的输出文件只允许使用`.text`、`.data`或`.bss`）。

如果输出目标文件的格式支持任意数量的节，但只有数字（就像 `Oasys` 的情况一样），则名称应作为带引号的数字字符串提供。节名可以由任何字符序列组成，但包含任何不寻常字符（如逗号）的名称必须加引号。`/DISCARD/` 是个特殊的名字，参见[输出节丢弃](#输出节丢弃)。

`section` 两边的空白符是必须的，以保证节的名字不会混淆。冒号和花括号也是必须的。当使用了 `fillexp` 且下一个 `sections-command` 看起来像表达式的延续时需要结尾的逗号，换行和其他空白符是可选的。

每个 `output-section-command` 可以是以下命令之一：

- 符号赋值（请参阅[赋值](#为符号赋值)）
- 输入节描述（参见[输入节](#输入节)）
- 直接包含的数据值（请参阅输出节数据）
- 特殊输出节关键字（请参阅输出节关键词）

### 输出节地址

上一部分中，`address` 是输出节的VMA（虚拟内存地址）的表达式。此地址是可选的，但如果提供了此地址，则输出地址将完全照此设置。

如果未指定输出地址，则将根据下面的方法中尝试。此时地址会被调整以符合对齐要求。输出节的对齐要求是所有输入节中含有的对齐要求中最严格的一个。

- 如果为该区域设置了输出内存区域（上面的 `region` 字段），那么它将被添加到该区域，其地址将是该区域中的下一个空闲地址。
- 如果使用 `MEMORY` 命令创建一个内存区域列表，那么将选择具有与该节兼容的属性的第一个区域来包含它。该节的输出地址将是该区域的下一个空闲地址;。
- 如果没有指定内存区域，或者没有匹配节，则输出地址将基于当前位置计数器的值。

例如 `.text . : { *(.text) }` 和 `.text : { *(.text) }` 有些许不同，前一个将设置输出节的地址为当前位置计数器的值，而后一个将设置输出节的地址为当前位置计数器的值经过对齐后的值。

`address` 也可以是任意表达式。例如，如果想在 0x10 字节边界上对齐节，使节地址的最低四位为零，可以使用 `.text ALIGN(0x10) : { *(.text) }`，其中 `ALIGN` 返回当前位置计数器向上对齐至指定值。

指定节的 `address` 将更改位置计数器的值，前提是这一节是非空的。（空节会被忽略）。

### 输入节

输入节描述是最基本的链接脚本操作。输出节告诉链接器如何在内存中放置程序，而输入节描述告诉链接器如何将输入文件映射到内存布局中。

#### 输入节基础

输入节描述由文件名（可选）和括号中的节名列表组成。

文件名和节名可以是通配符模式（请参阅[输入节通配符](#输入节通配符)）

最常见的输入节描述是在输出节中包含具有特定名称的所有输入节。例如，包含所有输入文件的`.text`节：`*(.text)`。

这里 `*` 是一个匹配任何文件名的通配符。要从匹配文件名通配符中排除一些文件，则可以使用 `EXCLUDE_FILE` 排除。例如：`EXCLUDE_FILE (*crtend.o *otherfile.o) *(.ctors)` 将会包含除了 `crtend.o` 和 `otherfile.o` 之外的所有输入文件中的 `.ctors` 节。也可以放入节列表，如 `*(EXCLUDE_FILE (*crtend.o *otherfile.o) .ctors)` ，结果是相同的。如果有节列表有多个节，`EXCLUDE_FILE` 的两种语法就很有用了。

有两种方法可以包含多个输入节：

- 使用空格分隔节名列表，例如 `*(.text .rdata)`；
- 使用括号分隔节名列表，例如 `*(.text) *(.rdata)`。

这两者之间的区别在于 ‘.text’ 和 ‘.rdata’ 输入节放置在输出节中的顺序。前者将混在一起，出现的顺序与它们在链接器输入中出现的顺序相同。后者会先放所有的 `.text` 节，然后放所有的 `.rdata` 节。

当将 `EXCLUDE_FILE` 与多个节一起使用时，如果使用在节列表中，则排除仅适用于紧接着的节，例如 `*(EXCLUDE_FILE (*somefile.o) .text .rdata)` 会排除 `somefile.o` 中的 `.text` 节，但是会包含所有文件的 `.rdata` 节。如想同时排除 `somefile.o` 中的 `.rdata` 节，则可以使用 `*(EXCLUDE_FILE (*somefile.o) .text EXCLUDE_FILE (*somefile.o) .rdata)`，或者把 `EXCLUDE_FILE` 放在节列表外，例如 `EXCLUDE_FILE (*somefile.o) *(.text .rdata)`。

如果某些文件特定位置的特殊数据需要包含在内存中，则可以指定文件名以包含来自特定文件的节。例如：`data.o(.data)` 将会包含 `data.o` 中的 `.data` 节。

使用 `INPUT_SECTION_FLAGS` 可以通过输入节的节标志(section flags)来选择节，比如我们可以使用 ELF 节的节头标志(Section header flags)：

```linker-script
SECTIONS {
  .text : { INPUT_SECTION_FLAGS (SHF_MERGE & SHF_STRINGS) *(.text) }
  .text2 :  { INPUT_SECTION_FLAGS (!SHF_WRITE) *(.text) }
}
```

在此示例中，输出部分 `.text` 将由匹配名称 `*(.text)` 且节头标志设置了 `SHF_MERGE` 与 `SHF_STRINGS` 标志的所有输入节组成。输出部分 `.text2` 将由匹配名称 `*(.text)` 且节头标志未设置 `SHF_WRITE` 标志的所有输入节组成。

还可以通过编写与库文件匹配的模式来指定文件，项目中未使用，如有需要请参考[Input Section Basics](https://sourceware.org/binutils/docs/ld/Input-Section-Basics.html)

如果没有指定节列表而直接使用文件名，则输入文件中的所有节都将包含在输出节中。这不常见，但有时可能有用。例如：

`data.o`

当使用的文件名不是`archive:file`说明符且不包含任何通配符时，链接器将首先查看是否还在链接器命令行或输入命令中指定了文件名。如果没有，链接器将尝试将该文件作为输入文件打开，就像它出现在命令行上一样。请注意，这与 `INPUT` 命令不同，因为链接器不会在库搜索路径中搜索文件。

#### 输入节通配符

在输入节描述中，文件名和节名都可以用通配符。在许多示例中看到的 `*` 文件名是简单的通配符模式。

- `*` 匹配任何文件名。
- `?` 匹配任何单字符。
- `[chars]` 匹配任何字符集合中的一个元素，其中 `-` 符号可以用来表示一个范围，如 `[a-z]` 匹配所有的小写字母。
- `\` 可以用来转义字符，如 `\*` 匹配 `*`。

文件名通配符只匹配在命令行或 `INPUT` 命令上明确指定的文件。链接器不搜索目录以扩展通配符。

如果文件名与多个通配符模式匹配成功，或者如果文件名被显式地指定且同时由通配符模式匹配成功，则链接器将使用链接脚本中的第一个匹配项。例如，以下命令可能出错，因为 `data.o` 规则不会被使用：

```linker-script
.data : { *(.data) }
.data1 : { data.o(.data) }
```

通常，链接器将以在链接期间发现的顺序排列通配符匹配的文件和节。 `SORT_BY_NAME`（或`SORT`)关键词可以更改这一顺序，该关键字放置在括号中的通配符模式前（例如 `SORT_BY_NAME(.text*)`）。使用 `SORT_BY_NAME` 关键字时，链接器会按文件或节名升序排序放入输出文件。

同样的，`SORT_BY_ALIGNMENT` 会将节按对齐方式降序排列，大的对齐被放在小的对齐前面可以减少所需的填充量。

`SORT_BY_INIT_PRIORITY` 按照编码在节名中的 `GCC init_priority` 属性升序排列节，在 `.init_array.NNNNN` 和 `.fini_array.NNNNN` 中，`NNNNN` 就是 `init_priority`。在 `.ctors.NNNNN` 和 `.dtors.NNNNN` 中，`NNNNN` 是 65535 减去 `init_priority`.

当链接器脚本中有嵌套节排序命令时，最多可以嵌套一层：

- `SORT_BY_NAME (SORT_BY_ALIGNMENT (wildcard section pattern))` 先按名，同名时再按对齐方式排序。
- `SORT_BY_ALIGNMENT (SORT_BY_NAME (wildcard section pattern))` 先按对齐方式，同对齐时再按名排序。
- `SORT_BY_NAME (SORT_BY_NAME (wildcard section pattern))` 和单层 `SORT_BY_NAME` 相同。
- `SORT_BY_ALIGNMENT (SORT_BY_ALIGNMENT (wildcard section pattern))` 和单层 `SORT_BY_ALIGNMENT` 相同。
- 其他嵌套方式均无效。

当命令行排序选项和脚本排序选项同时使用时，脚本排序选项优先。

如果链接脚本没有排序嵌套，命令行选项也可能导致排序的嵌套：

- `SORT_BY_NAME(wildcard section pattern)` 与 `--sort-sections alignment` 共用等价于 `SORT_BY_NAME (SORT_BY_ALIGNMENT (wildcard section pattern))`
- `SORT_BY_ALIGNMENT(wildcard section pattern)` 与 `--sort-section name` 共用等价于 `SORT_BY_ALIGNMENT (SORT_BY_NAME (wildcard section pattern))`

如果链接脚本中的排序命令已经发生嵌套，则将忽略命令行选项。

使用 `SORT_NONE` 可以忽略命令行节排序选项。

如果对输入节在输出文件中的布局感到困惑，请使用`-M`链接器选项生成 map 文件。map 文件精确地显示了如何将输入节映射到输出节。

下面这个例子中，链接器会将所有输入的 `.text` 节放入输出 `.text` 节，并且将所有输入的 `.bss` 节放入输出 `.bss` 节。所有大写字母开头的输入文件中的 `.data` 节都会被放入输出 `.DATA` 节。对于其他文件，链接器会将它们的 `.data` 节放入输出 `.data` 节。

```linker-script
SECTIONS {
  .text : { *(.text) }
  .DATA : { [A-Z]*(.data) }
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```

#### 输入节通用符号

请参考[Input Section for Common Symbols](https://sourceware.org/binutils/docs/ld/Input-Section-Common.html)

#### 输入节与垃圾回收

当使用链接时垃圾回收功能 `--gc-sections` 时，通常应标记不应删除的节，此时请使用 `KEEP()`，比如 `KEEP(*(.init))` 或 `KEEP(SORT_BY_NAME(*)(.ctors)).`

#### 输入节示例

请参考[Input Section Example](https://sourceware.org/binutils/docs/ld/Input-Section-Example.html)

### 输出节数据

项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Output-Section-Data.html)自行理解。

### 输出节关键词

项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Output-Section-Keywords.html)自行理解。

### 输出节丢弃

项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Output-Section-Discarding.html)自行理解。

### 输出节属性

项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Output-Section-Attributes.html)自行理解。

### 覆盖描述
项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Overlay-Description.html)自行理解。

## 其余命令

除此之外，还有 `MEMORY`、`PHDRS`、`VERSION` 命令可供使用，项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Scripts.html)自行理解。

## 链接脚本中的表达式

链接脚本语言中表达式的语法与 C 表达式的语法相同，只是在某些地方需要空格来解决语法歧义。所有表达式都作为整数计算。所有表达式都以相同的大小计算，如果主机和目标都是 32 位，则为 32 位，否则为 64 位。

可以在表达式中使用和设置符号值。

链接器定义了几个在表达式中使用的专用内置函数。

项目中暂时用不到，可以参考[文档](https://sourceware.org/binutils/docs/ld/Expressions.html)自行理解。

## 隐式链接脚本

如果指定了链接器输入文件，但链接器无法将其识别为目标文件或库文件，那么将尝试将该文件读取为链接脚本。如果无法将文件解析为链接脚本，则链接器将报告错误。

隐式链接脚本不会替换默认链接脚本。

通常，隐式链接脚本只包含符号赋值，或 `INPUT`、`GROUP`、`VERSION` 命令。

由隐式链接脚本而读取的任何输入文件都将在读取隐式链接脚本的命令行位置读取。这可能会影响库的搜索。
