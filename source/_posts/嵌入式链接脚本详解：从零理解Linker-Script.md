---
title: 嵌入式链接脚本详解：从零理解Linker Script
date: 2026-08-01 15:17:43
tags:
  - 嵌入式
  - 链接脚本
  - Linker Script
  - GCC
  - STM32
  - 汇编
---

在嵌入式开发中，我们编译完 C/汇编代码后，得到的 `.o` 目标文件并不能直接烧录到单片机里。把它们"拼装"成最终可执行文件、并决定每一段代码和数据放在芯片哪个地址的任务，就由**链接器**（Linker）完成，而它的"施工图纸"就是**链接脚本**（Linker Script）。

很多初学者把链接脚本当成一个"复制粘贴即可"的黑盒，一旦出现 `section .isr_vector will not fit in region FLASH` 之类的报错就束手无策。本文将从零开始，把链接脚本的语法、概念、实战与调试技巧一次讲透。

<!-- more -->

## 1. 为什么嵌入式需要链接脚本？

### 1.1 链接器在干什么

以 GCC 工具链为例，从源码到固件的完整链路是：

```
.c/.s 文件 --编译器--> .o 目标文件 --链接器(ld)--> .elf --objcopy--> .hex/.bin
```

编译器负责**翻译**，链接器负责**拼装**：

1. **符号解析（Symbol Resolution）**：把各个 `.o` 中互相引用的符号（函数、全局变量）联系起来；
2. **重定位（Relocation）**：计算每个符号的最终内存地址，回填到代码中的跳转指令、变量引用处；
3. **段合并（Section Merging）**：把所有 `.o` 中相同性质的段合并成可执行文件中的段。

> PC 端程序通常无需关心链接脚本，因为标准库和操作系统已经定义了默认布局。但裸机（bare-metal）嵌入式程序没有 OS 兜底，芯片的 Flash/RAM 地址范围必须由我们自己指定——这正是链接脚本存在的根本原因。

### 1.2 两个关键概念：VMA 与 LMA

链接脚本中反复出现两个地址，务必先分清：

| 概念 | 全称 | 含义 | 例子 |
|---|---|---|---|
| **VMA** | Virtual Memory Address | **运行地址**，程序执行时符号所在的地址 | `.text` 在运行时位于 0x08000000（Flash） |
| **LMA** | Load Memory Address | **加载地址**，程序烧录/存放的地址 | 初始化数据 `init_data` 烧录在 Flash，运行时复制到 RAM |

对于不搬移的段（如代码段），`VMA == LMA`；而对于需要从 Flash 拷到 RAM 的段（如全局变量初始化值），两者不同。这也是"启动时为什么要拷贝 .data、清 .bss"的根源。

---

## 2. 链接脚本的语法基础

链接脚本由**命令**和**表达式**组成，行尾用 `;` 结尾，`/* */` 是注释。

### 2.1 一个最简脚本

```ld
/* 最小可用的链接脚本 */
ENTRY(Reset_Handler)          /* 指定入口点 */

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
    .text :
    {
        *(.isr_vector)
        *(.text*)
        *(.rodata*)
        _etext = .;
    } > FLASH

    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
    } > RAM AT> FLASH

    .bss :
    {
        _sbss = .;
        *(.bss*)
        _ebss = .;
    } > RAM
}
```

这个脚本包含了链接脚本的三大核心命令：`ENTRY`、`MEMORY`、`SECTIONS`。下面逐一拆解。

---

## 3. `MEMORY`：定义芯片内存布局

`MEMORY` 命令告诉链接器目标芯片上有哪些可用的内存区域及其属性：

```
MEMORY
{
    区域名 (属性) : ORIGIN = 起始地址, LENGTH = 长度
}
```

### 3.1 属性字符

| 属性 | 含义 |
|---|---|
| `r` | 只读（Read-only） |
| `w` | 可写（Writable） |
| `x` | 可执行（Executable） |
| `a` | 已分配（Allocatable） |
| `!` | 取反，如 `!x` 表示不可执行 |

常用组合：

- `FLASH (rx)` —— 可读可执行，存放代码和只读数据
- `RAM (rwx)` 或 `RAM (rw)` —— 可读可写，存放变量
- 有些芯片还有 CCM RAM、TCM、SDRAM 等，各自定义区域即可

> 提示：内存区域名可以任意起，但建议语义化。`ORIGIN` 和 `LENGTH` 必须和芯片手册的存储器映射严格一致，填错会导致程序"跑飞"或烧录失败。

### 3.2 常见芯片内存映射示例

以 STM32F103 系列为例：

- **Flash**：0x08000000 ~ 0x0807FFFF（512KB）
- **SRAM**：0x20000000 ~ 0x2000FFFF（64KB）
- **系统存储器**：0x1FFFF000（Bootloader 区，一般不放进脚本）
- **外设寄存器**：0x40000000 起（由芯片头文件定义，不占链接脚本的 MEMORY）

---

## 4. `SECTIONS`：核心中的核心

`SECTIONS` 命令描述**输出段**（output section）如何由**输入段**（input section）构成，以及各输出段放在哪块内存。语法：

```
SECTIONS
{
    输出段名 [地址] : [AT(加载地址)] [ALIGN(...)]
    {
        子命令（输入段描述、符号赋值等）
    } [> 内存区域] [AT> 内存区域] [PHDRS ...]
}
```

### 4.1 输出段与输入段

- **输入段**：每个 `.o` 文件内部的段，由编译器产生。常见的有：
  - `.text` —— 代码
  - `.rodata` —— 只读数据（字符串、const 常量）
  - `.data` —— 已初始化全局/静态变量
  - `.bss` —— 未初始化全局/静态变量（不占文件空间，运行时清零）
  - `.isr_vector` —— 中断向量表（STM32 从 0x08000000 起始）
  - 自定义段（`__attribute__((section(".my_section")))`）
- **输出段**：链接脚本中定义的段，合并多个输入段后放入指定内存。

**输入段的通配符写法**：

```
*(.text)       /* 所有目标文件的 .text 段 */
*(.text*)      /* 所有以 .text 开头的段（含 .text.funcname 等） */
lib*.o(.data)  /* 只取 lib 前缀的目标文件的 .data 段 */
```

> `*(.text*)` 比 `*(.text)` 更常用，因为现代编译器（特别是 `-ffunction-sections`）会为每个函数生成独立段，如 `.text.main`、`.text.uart_init`。

### 4.2 位置计数器 `.` 与符号赋值

`.`（点号）是**位置计数器**（location counter），代表当前输出段内已经占用的字节数，始终跟随放置内容自动增长。

通过在脚本中给符号赋值，我们可以"导出"当前地址给 C 代码使用：

```ld
.data :
{
    _sdata = .;        /* 记录 .data 起始地址 */
    *(.data*)
    _edata = .;        /* 记录 .data 结束地址 */
} > RAM AT> FLASH
```

然后在启动文件或 C 代码中声明并取地址：

```c
extern uint32_t _sdata, _edata, _sbss, _ebss, _etext;
uint32_t *src = &_etext;              /* 数据在 Flash 中的源地址 */
uint32_t *dst = &_sdata;
while (dst < &_edata) {
    *dst++ = *src++;                  /* 拷贝 .data 到 RAM */
}
```

符号赋值时若符号是"纯地址"，建议用 `. = 定义` 或直接 `符号 = .` 形式，并在 `.h` 中用 `extern` 声明。注意链接脚本中的符号是"地址值"，在 C 里要用 `&符号` 取用。

### 4.3 常用段：.text / .data / .bss 的作用

| 段 | 内容 | 运行时位置 | 烧录位置 | 特点 |
|---|---|---|---|---|
| `.isr_vector` | 中断向量表 | Flash | Flash | 必须放 Flash 起始地址 |
| `.text` | 代码 + 立即数 | Flash | Flash | 只读、可执行 |
| `.rodata` | 只读常量 | Flash | Flash | 只读 |
| `.data` | 已初始化变量 | RAM | **Flash** | 需启动时拷 RAM |
| `.bss` | 未初始化变量 | RAM | **不烧录** | 需启动时清零 |
| `.heap` | 堆 | RAM | — | 动态内存 |
| `.stack` | 栈 | RAM | — | 函数调用栈 |

---

## 5. `> 区域` 与 `AT> 区域`：VMA 与 LMA 分离

`> RAM AT> FLASH` 是链接脚本中最优雅也最容易被忽略的语法：

```
.data : { ... } > RAM AT> FLASH
```

- `> RAM` 指定**VMA**（运行地址）在 RAM 中；
- `AT> FLASH` 指定 **LMA**（加载地址）在 Flash 中。

这意味着：`.data` 段初始值的**文件映像**被烧录进 Flash，但程序访问它时用的是 RAM 地址。所以启动代码必须把它从 Flash 拷贝到 RAM。

### 5.1 如何知道 LMA 地址？

链接脚本会导出 `LOADADDR(.data)`，也可以直接在 C 里用 `__etext`（紧接 `.text` 结尾的符号）定位数据在 Flash 中的源位置。启动汇编代码中通常这样写：

```assembly
; STM32 启动代码片段（拷贝 .data）
ldr r0, =_etext        ; Flash 中的源
ldr r1, =_sdata        ; RAM 中的目的
ldr r2, =_edata
copy_data:
    cmp r1, r2
    bge data_done
    ldr r3, [r0], #4
    str r3, [r1], #4
    b copy_data
data_done:
```

---

## 6. 位置计数器的高级操作

### 6.1 `ALIGN()` 对齐

为了性能和 CPU 对齐要求，段和符号通常需要对齐：

```ld
. = ALIGN(4);          /* 当前位置对齐到 4 字节边界 */
_sdata = .;
```

`ALIGN(n)` 返回把当前位置向上取整到 n 的倍数后的值。常用 4、8、16。若不显式对齐，编译器产生的数据可能错位，导致某些平台（如 Cortex-M 的未对齐访问）进入 HardFault。

### 6.2 `KEEP()`：防止被垃圾回收

当开启 `--gc-sections`（丢弃未引用段以减小固件体积）时，即使脚本里写了的输入段也可能被丢弃。用 `KEEP()` 包裹可强制保留：

```ld
.isr_vector :
{
    KEEP(*(.isr_vector))   /* 向量表必须保留！ */
} > FLASH
```

典型应用：中断向量表、构造函数表（`__init_array_start`）、看门狗复位入口等。

### 6.3 自定义段

想把自己的变量固定到某地址，可定义一个专属段：

```c
__attribute__((section(".backup_ram"), aligned(4)))
uint32_t backup_flag;         /* 声明放到 .backup_ram 段 */
```

链接脚本：

```ld
.backup_ram (NOLOAD) :   /* NOLOAD: 不烧录，只保留 RAM 空间 */
{
    *(.backup_ram*)
} > RAM
```

`NOLOAD` 关键字告诉链接器：该段只分配空间、不生成烧录数据，非常适合"掉电保持区"。

---

## 7. `ENTRY` 与启动流程配合

`ENTRY(Reset_Handler)` 指定可执行文件的入口点，即 CPU 复位后跳转的第一条指令。虽然对 Cortex-M 来说，真正的入口是**向量表首元素（初始 SP）和第二元素（Reset_Handler 地址）**，但 `ENTRY` 仍然是调试器和静态分析工具识别的关键信息。

一份完整的 STM32 启动流程与链接脚本的配合关系：

1. **上电**：CPU 从 Flash 起始地址取初始栈指针（向量表第 1 项）；
2. **跳转**：跳转到 Reset_Handler（向量表第 2 项）；
3. **拷贝 .data**：`_etext` → `_sdata`，长度到 `_edata`；
4. **清零 .bss**：从 `_sbss` 到 `_ebss`；
5. **初始化堆栈指针**（如有 `__stack` 符号）；
6. **调用 `SystemInit`、`main`**。

可见，启动代码里那些"魔法符号"（`_sdata`、`_etext`、`_sbss`…）全部来自链接脚本。**启动文件和链接脚本必须成对维护**，这是嵌入式移植中最高频的 bug 来源之一。

---

## 8. 完整实战：以 STM32 为例的链接脚本精讲

下面是一份带注释的、可直接用于 STM32F103C8T6（64KB Flash / 20KB RAM）的完整脚本：

```ld
/* memory.x —— STM32F103C8T6 链接脚本 */
ENTRY(Reset_Handler)

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}

/* 栈顶：指向 RAM 顶端，向下增长 */
_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
    /* 1. 中断向量表：必须在 Flash 起始，且不能被 gc-sections 删掉 */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } > FLASH

    /* 2. 代码段 */
    .text :
    {
        . = ALIGN(4);
        *(.text*)
        *(.rodata*)
        _etext = .;          /* .data 在 Flash 中的源地址 */
        . = ALIGN(4);
    } > FLASH

    /* 3. 已初始化数据：VMA 在 RAM，LMA 在 Flash */
    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
        . = ALIGN(4);
    } > RAM AT> FLASH

    /* 4. 未初始化数据：只占 RAM，启动时清零 */
    .bss :
    {
        _sbss = .;
        *(.bss*)
        *(COMMON)            /* 合并编译器公共块 */
        _ebss = .;
        . = ALIGN(4);
    } > RAM

    /* 5. 堆（可选） */
    .heap :
    {
        __heap_start = .;
        . = . + 2K;          /* 预留 2KB */
        __heap_end = .;
    } > RAM
}
```

**几个值得注意的细节**：

- `_estack = ORIGIN(RAM) + LENGTH(RAM);` 直接由 MEMORY 计算栈顶，无需硬编码；
- `.data` 段刻意放在 `.text` 之后，保证 `_etext` 紧邻 Flash 中数据的源头；
- `*(COMMON)` 合并未初始化的公共符号，避免遗漏；
- 用 `ALIGN(4)` 保证 Cortex-M 的 4 字节对齐。

---

## 9. 常用链接脚本命令速查

| 命令/关键字 | 作用 |
|---|---|
| `ENTRY(sym)` | 指定入口点 |
| `MEMORY {...}` | 定义可用内存区域 |
| `SECTIONS {...}` | 定义输出段布局 |
| `AT(addr)` / `AT> 区域` | 指定 LMA |
| `> 区域` | 指定 VMA |
| `ALIGN(n)` | 对齐到 n 的倍数 |
| `KEEP(sec)` | 阻止 gc-sections 丢弃 |
| `NOLOAD` | 只分配空间不烧录 |
| `ORIGIN(region)` / `LENGTH(region)` | 查询区域信息 |
| `.` | 位置计数器 |
| `LOADADDR(sec)` | 某段的 LMA |
| `PROVIDE(sym = exp)` | 符号缺省定义（弱定义） |
| `ASSERT(cond, msg)` | 链接期断言，条件不满足报错 |

### 9.1 `PROVIDE` 的妙用

```ld
PROVIDE(__stack = _estack);
```

如果目标文件中没有定义 `__stack`，链接器就提供它；如果用户代码已定义则跳过。这给第三方库（如某些 RTOS 启动文件）提供了"缺省但不覆盖"的友好接口。

### 9.2 `ASSERT`：把错误挡在烧录之前

```ld
ASSERT(_ebss <= ORIGIN(RAM) + LENGTH(RAM),
       "RAM 溢出：.bss 超出 RAM 范围！")
```

链接时报错比运行时死机好排查一万倍，善用 `ASSERT`。

---

## 10. 常见报错与调试技巧

### 10.1 经典报错对照表

| 报错信息 | 原因 | 解决办法 |
|---|---|---|
| `section .text will not fit in region FLASH` | 代码超出 Flash | 优化代码/开启 `-Os`，检查有无死代码 |
| `region RAM overflowed by N bytes` | RAM 超限 | 查数组、全局变量；`.bss` 过大 |
| `undefined reference to _sdata` | 启动文件与脚本符号不一致 | 对照启动文件逐一核对符号名 |
| `relocation truncated to fit: R_ARM_PREL31 ...` | 跳转超出范围/段地址不对 | 检查 Flash 起始地址、向量表位置 |
| `cannot find entry symbol Reset_Handler` | `ENTRY` 写错/函数名拼写不一致 | 确认 `ENTRY` 与启动文件函数名一致 |
| `multiple definition of xxx` | 全局变量重复定义 | 检查 `-fno-common` 与头文件定义 |

### 10.2 调试工具：`nm`、`objdump`、`size`

排查链接脚本最有效的三板斧：

```bash
# 1. 查看符号最终地址
arm-none-eabi-nm -n firmware.elf

# 2. 反汇编，检查跳转目标是否合理
arm-none-eabi-objdump -d firmware.elf

# 3. 查看各段占用
arm-none-eabi-size -A firmware.elf

# 4. 完整段布局与 VMA/LMA（最有价值）
arm-none-eabi-objdump -h firmware.elf
```

`objdump -h` 会列出每个段的 **VMA 和 LMA**，一眼就能看出 `.data` 是否正确地"住在 RAM、源于 Flash"：

```
Sections:
Idx Name      Size    VMA       LMA       File off  Algn
  0 .isr_vector 000000c0 08000000 08000000  00010000  2**2
  1 .text     000023c8 080000c0 080000c0  000100c0  2**2
  2 .data     00000010 20000000 08002488   ...
  3 .bss      00000230 20000010 20000010   ...
```

### 10.3 打印链接 Map 文件

编译时加 `-Wl,-Map=output.map` 生成 Map 文件，这是排查内存布局的"说明书"，包含所有符号地址、段归属、废弃段等完整信息：

```bash
arm-none-eabi-gcc ... -Wl,-Map=firmware.map
```

---

## 11. 进阶：-ffunction-sections / -fdata-sections / --gc-sections

为了裁剪固件体积，GCC 提供了"每函数/每变量独立段 + 垃圾回收"的组合拳：

```makefile
CFLAGS  += -ffunction-sections -fdata-sections
LDFLAGS += -Wl,--gc-sections
```

- `-ffunction-sections`：每个函数单独成段（`.text.uart_init`）；
- `-fdata-sections`：每个数据对象单独成段（`.data.g_value`）；
- `--gc-sections`：链接时删除没有被引用到的段。

配合 `KEEP()` 保留必须的段（向量表、构造函数表），可以在不牺牲功能的情况下显著减小固件。

> ⚠️ 注意：用了 `-ffunction-sections` 后，脚本中应使用 `*(.text*)` 而非 `*(.text)`，否则段匹配不上，会得到"空段"。

---

## 12. 与其他工具链的异同

虽然本文以 GCC `ld` 为主线，但链接脚本思想是通用的：

| 工具链 | 链接脚本机制 |
|---|---|
| **GCC (arm-none-eabi)** | `.ld` / `.x` 文件，`ld` 语法（本文内容） |
| **IAR** | `.icf` 文件，`place in` 等关键字 |
| **Keil MDK** | `.sct` 文件，`LR_` / `ER_` 加载区与执行区 |
| **GNU ld with LLD** | 兼容 `ld` 语法的子集 |

IAR 的 `.icf`：

```
define memory FLASH with size = 512K, start = 0x08000000;
define memory RAM   with size = 128K, start = 0x20000000;
place in FLASH { readonly section .text };
place in RAM   { readwrite section .data };
```

Keil 的 `.sct`：

```
LR_IROM1 0x08000000 0x00080000 {
  ER_IROM1 0x08000000 0x00080000 {
    *.o (RESET, +First)
    *(InRoot$$Sections)
    .ANY (+RO)
  }
  RW_IRAM1 0x20000000 0x00020000 {
    .ANY (+RW +ZI)
  }
}
```

**换了 IDE 就换一套链接描述语言，但"VMA / LMA / 段 / 位置计数器"这些概念完全相通**——理解 GCC 脚本后，切换到 IAR/Keil 只是换语法而已。

---

## 13. 总结

链接脚本是嵌入式工程师从"会写 C"走向"懂底层"的分水岭。回顾本文核心要点：

1. **链接器**负责符号解析与重定位，裸机程序必须用链接脚本定义内存布局；
2. **MEMORY** 定义芯片可用的内存区域与属性；
3. **SECTIONS** 定义输出段如何由输入段组成、放在哪块区域；
4. **VMA 与 LMA** 是理解 `.data` 拷贝、启动代码的钥匙；
5. **`.`（位置计数器）、`ALIGN`、`KEEP`、`NOLOAD`** 是日常最常用的控制手段；
6. **启动文件与链接脚本是一体两面**，符号必须严格对应；
7. 善用 `ASSERT`、`Map 文件`、`objdump -h` 把内存问题扼杀在链接阶段。

理解链接脚本之后，你会发现自己对"程序到底怎么跑起来的"有了更清晰的画面：从 Flash 读取向量表 → 进 Reset_Handler → 搬数据 → 清 BSS → 跳转 main……而这一切的幕后导演，正是这份被无数人"复制粘贴"却鲜有人真正读懂的 **Linker Script**。

---

*参考文档：GNU ld 手册（`info ld`）、STM32 启动文件源码、Cortex-M3 权威指南 存储器映射章节。*
