---
title: 嵌入式链接脚本详解：以英飞凌TC375为例
date: 2026-08-01 15:17:43
tags:
  - 嵌入式
  - 链接脚本
  - Linker Script
  - 英飞凌
  - TC375
  - AURIX
  - TriCore
  - GCC
  - 汇编
---

在嵌入式开发中，我们编译完 C/汇编代码后，得到的 `.o` 目标文件并不能直接烧录到芯片里。把它们"拼装"成最终可执行文件、并决定每一段代码和数据放在芯片哪个地址的任务，就由**链接器**（Linker）完成，而它的"施工图纸"就是**链接脚本**（Linker Script）。

很多初学者把链接脚本当成一个"复制粘贴即可"的黑盒，一旦出现 `section .text will not fit in region pfls0` 之类的报错就束手无策。本文以英飞凌 **TC375**（AURIX TC3xx 系列，三核 TriCore 1.6.2P 架构，300MHz）为具体例子，从零开始，把链接脚本的语法、概念、多核内存布局、实战与调试技巧一次讲透。

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

| 概念 | 全称 | 含义 | 例子（TC375） |
|---|---|---|---|
| **VMA** | Virtual Memory Address | **运行地址**，程序执行时符号所在的地址 | `.text` 在运行时位于 0x80000000（PFLASH0） |
| **LMA** | Load Memory Address | **加载地址**，程序烧录/存放的地址 | 初始化数据 `init_data` 烧录在 PFLASH，运行时复制到 DSPR/LMU |

对于不搬移的段（如代码段），`VMA == LMA`；而对于需要从 Flash 拷到 RAM 的段（如全局变量初始化值），两者不同。这也是"启动时为什么要拷贝 .data、清 .bss"的根源。

---

## 2. 链接脚本的语法基础

链接脚本由**命令**和**表达式**组成，行尾用 `;` 结尾，`/* */` 是注释。

### 2.1 一个最简脚本

```ld
/* 最小可用的链接脚本（TC375 单核示意） */
OUTPUT_FORMAT("elf32-tricore")
OUTPUT_ARCH(tricore)
ENTRY(_start)             /* 指定入口点 */

MEMORY
{
    pfls0 (rx!p) : ORIGIN = 0x80000000, LENGTH = 3M   /* 程序 Flash0 */
    dsram0 (w!xp) : ORIGIN = 0x70000000, LENGTH = 240K /* CPU0 本地数据 RAM */
}

SECTIONS
{
    .text :
    {
        *(.text*)
        *(.rodata*)
        _etext = .;
    } > pfls0

    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
    } > dsram0 AT> pfls0

    .bss :
    {
        _sbss = .;
        *(.bss*)
        _ebss = .;
    } > dsram0
}
```

这个脚本包含了链接脚本的三大核心命令：`ENTRY`、`MEMORY`、`SECTIONS`。下面逐一拆解。

> 注意 `pfls0 (rx!p)` 中的 `p` 属性：它是 TriCore 链接脚本中特有的属性，`!p` 表示"不可分页/非物理可寻址"区域（AURIX GCC 扩展），用于区分缓存与非缓存映射。

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
| `p` | TriCore 扩展：物理/可寻址标志（AURIX GCC） |

常用组合：

- `pfls0 (rx!p)` —— 程序 Flash，可读可执行，非物理可寻址
- `dsram0 (w!xp)` —— CPU 本地数据 RAM，可读可写可执行
- 有些芯片还有 CCM RAM、TCM、SDRAM 等，各自定义区域即可

> 提示：内存区域名可以任意起，但建议语义化。`ORIGIN` 和 `LENGTH` 必须和芯片手册的存储器映射严格一致，填错会导致程序"跑飞"或烧录失败。

### 3.2 TC375 的多核内存映射（核心重点）

TC375 是 **3 核 TriCore 1.6.2P** 架构，PFLASH **6MB**、DFLASH **384KB**（DF0 256KB + DF1 128KB）、总 SRAM **992KB**。它的内存映射比单核 MCU 复杂得多——每个核都有自己的私有 RAM，还有共享 RAM 与多组 Flash。下表是依据**英飞凌官方 `AURIX TC37x 用户手册 Appendix` 第2章（MEMMAP）**核对的内存映射（与 `Lcf_Gnuc_Tricore_Tc.lsl` 链接脚本一致）：

| 区域名 | 起始地址 | 长度 | 用途 | 缓存 |
|---|---|---|---|---|
| `pfls0` | 0x8000_0000 | 3MB | 程序 Flash0（PFI0，代码/只读数据） | 可缓存 |
| `pfls0_nc` | 0xA000_0000 | 3MB | PFLASH0 非缓存镜像 | 非缓存 |
| `pfls1` | 0x8030_0000 | 3MB | 程序 Flash1（PFI1，第二块代码区） | 可缓存 |
| `pfls1_nc` | 0xA030_0000 | 3MB | PFLASH1 非缓存镜像 | 非缓存 |
| `dfls0` | 0xAF00_0000 | 256KB | 数据 Flash0（EEPROM 模拟） | 非缓存 |
| `ucb` | 0xAF40_0000 | 24KB | 用户配置块 UCB（含 BMHD0-3 引导头） | 非缓存 |
| `dfls1` | 0xAFC0_0000 | 128KB | 数据 Flash1（HSM EEPROM） | 非缓存 |
| `brom` | 0x8FFF_0000 | 64KB | Boot ROM（DMU，SSW 启动软件） | 可缓存 |
| `dsram0` | 0x7000_0000 | 240KB | **CPU0** 数据 ScratchPad RAM（DSPR0） | — |
| `psram0` | 0x7010_0000 | 64KB | **CPU0** 程序 ScratchPad RAM（PSPR0） | — |
| `dsram1` | 0x6000_0000 | 240KB | **CPU1** 数据 ScratchPad RAM（DSPR1） | — |
| `psram1` | 0x6010_0000 | 64KB | **CPU1** 程序 ScratchPad RAM（PSPR1） | — |
| `dsram2` | 0x5000_0000 | 96KB | **CPU2** 数据 ScratchPad RAM（DSPR2，注意只有 96KB） | — |
| `psram2` | 0x5010_0000 | 64KB | **CPU2** 程序 ScratchPad RAM（PSPR2） | — |
| `cpu0_dlmu` | 0x9000_0000 | 64KB | 全局共享 RAM（DLMU0，LMU 低段） | 可缓存 |
| `cpu1_dlmu` | 0x9001_0000 | 64KB | 全局共享 RAM（DLMU1） | 可缓存 |
| `cpu2_dlmu` | 0x9002_0000 | 64KB | 全局共享 RAM（DLMU2） | 可缓存 |
| `cpu0_dlmu_nc` | 0xB000_0000 | 64KB | DLMU0 非缓存镜像 | 非缓存 |
| `cpu1_dlmu_nc` | 0xB001_0000 | 64KB | DLMU1 非缓存镜像 | 非缓存 |
| `cpu2_dlmu_nc` | 0xB002_0000 | 64KB | DLMU2 非缓存镜像 | 非缓存 |
| `dam_ram` | 0x9040_0000 | 32KB | DAM SRAM（全局直接访问内存） | 可缓存 |
| `olda` | 0x8FE0_0000 | 512KB | Online Data Acquisition（调试用） | — |

**AURIX 独有的地址映射机制**：

- **全局地址 vs 本地地址**：每个核的 DSPR 除了全局地址（0x7000_0000 / 0x6000_0000 / 0x5000_0000），还有一个**本地地址 0xD000_0000**。本地地址是指令效率最高的访问方式（单周期、无需 SRI 总线仲裁），因为它是与核直接耦合的 Scratchpad RAM。链接脚本通过 `REGION_MAP` 把二者映射起来：
  ```ld
  REGION_MAP(CPU0, ORIGIN(dsram0_local), LENGTH(dsram0_local), ORIGIN(dsram0))
  REGION_MAP(CPU1, ORIGIN(dsram1_local), LENGTH(dsram1_local), ORIGIN(dsram1))
  ```
- **缓存/非缓存镜像**：同一物理 Flash/RAM 有可缓存地址（如 0x8000_0000）和非缓存地址（0xA000_0000）两个别名。涉及 DMA、多核共享数据时要放到非缓存区，避免 cache 一致性问题。链接脚本用 `REGION_MIRROR` 处理：
  ```ld
  REGION_MIRROR("pfls0", "pfls0_nc")
  REGION_MIRROR("cpu0_dlmu", "cpu0_dlmu_nc")
  ```
- **16 个 256MB 段**：TriCore 4GB 地址空间按 `A[31:28]` 分为 16 段（Segment 0~15）。TC375 中：**段 1/3-7** 放 PSPR/DSPR/PCACHE/DCACHE，**段 8** 缓存访问 PFLASH/BROM，**段 9** 缓存访问 LMU，**段 10** 非缓存访问 PFLASH/DFLASH/BROM，**段 11** 非缓存访问 LMU，**段 15** 是 SPB/SRI 外设寄存器空间（如 F880_0000 起是 CPU 内核特殊功能寄存器）。链接脚本中 `pfls0`（0x8...）、`pfls0_nc`（0xA...）、`cpu0_dlmu`（0x9...）、`cpu0_dlmu_nc`（0xB...）正好落在这些段上。

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

- **输入段**：每个 `.o` 文件内部的段，由编译器产生。TriCore 工具链常见的有：
  - `.text` —— 代码
  - `.rodata` —— 只读数据（字符串、const 常量）
  - `.data` —— 已初始化全局/静态变量
  - `.bss` —— 未初始化全局/静态变量（不占文件空间，运行时清零）
  - **`.zdata` / `.zbss` / `.zrodata`** —— TriCore 特有的"近地址小数据段"，用 18 位绝对寻址（abs18），可被一条指令直接访问，性能极高。链接时需放在低 512KB 地址范围内，且需要导出基址寄存器符号 `_SMALL_DATA_`
  - `.csa` —— Context Save Area（上下文保存区），TriCore 中断/函数调用切换上下文用，必须按 64 字节对齐
  - `.intvec` —— 中断向量（每个 8 字节，跳转到中断服务程序）
  - `.traptab` —— 陷阱（异常）向量表
  - `.start_tc0` —— CPU0 复位入口段
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
} > dsram0 AT> pfls0
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
| `.start_tc0` | CPU0 复位入口 | PFLASH | PFLASH | 必须放 PFLASH 起始附近 |
| `.text` | 代码 + 立即数 | PFLASH | PFLASH | 只读、可执行 |
| `.rodata` | 只读常量 | PFLASH | PFLASH | 只读 |
| `.zrodata/.zdata/.zbss` | 近地址小数据 | PFLASH/RAM | PFLASH/RAM | abs18 寻址，需导出 `_SMALL_DATA_` |
| `.data` | 已初始化变量 | DSPR/LMU | **PFLASH** | 需启动时拷 RAM |
| `.bss` | 未初始化变量 | DSPR/LMU | **不烧录** | 需启动时清零 |
| `.csa` | 上下文保存区 | DSPR | — | 64 字节对齐，每核独占 |
| `.heap` | 堆 | DSPR/LMU | — | 动态内存 |
| `.ustack/.istack` | 用户栈/中断栈 | DSPR | — | 每个核各一个 |

---

## 5. `> 区域` 与 `AT> 区域`：VMA 与 LMA 分离

`> dsram0 AT> pfls0` 是链接脚本中最优雅也最容易被忽略的语法：

```
.data : { ... } > dsram0 AT> pfls0
```

- `> dsram0` 指定**VMA**（运行地址）在 CPU0 本地 RAM 中；
- `AT> pfls0` 指定 **LMA**（加载地址）在程序 Flash 中。

这意味着：`.data` 段初始值的**文件映像**被烧录进 PFLASH，但程序访问它时用的是 DSPR 地址。所以启动代码必须把它从 Flash 拷贝到 RAM。

### 5.1 如何知道 LMA 地址？

链接脚本会导出 `LOADADDR(.data)`，也可以直接在 C 里用 `_etext`（紧接 `.text` 结尾的符号）定位数据在 Flash 中的源位置。AURIX 的启动代码（`_start`）中通常这样写（TriCore 汇编）：

```assembly
; AURIX _start 启动代码片段（拷贝 .data，CPU0）
    movh.a  %a2, hi:_etext      ; Flash 中的源
    lea     %a2, [%a2] lo:_etext
    movh.a  %a3, hi:_sdata      ; DSPR 中的目的
    lea     %a3, [%a3] lo:_sdata
copy_data:
    ld.w    %d0, [%a2+]4        ; 加载 4 字节
    st.w    [%a3+]4, %d0        ; 存储 4 字节
    jlt.u   %a3, _edata, copy_data
```

---

## 6. 位置计数器的高级操作

### 6.1 `ALIGN()` 对齐

为了性能和 CPU 对齐要求，段和符号通常需要对齐。**TriCore 的 CSA 要求 64 字节对齐**，栈要求 8 字节对齐：

```ld
. = ALIGN(4);          /* 当前位置对齐到 4 字节边界 */
_sdata = .;
```

`ALIGN(n)` 返回把当前位置向上取整到 n 的倍数后的值。常用 4、8、64。若不显式对齐，编译器产生的数据可能错位，TriCore 会对未对齐访问触发陷阱（Trap）。

### 6.2 `KEEP()`：防止被垃圾回收

当开启 `--gc-sections`（丢弃未引用段以减小固件体积）时，即使脚本里写了的输入段也可能被丢弃。用 `KEEP()` 包裹可强制保留：

```ld
.start_tc0 :
{
    KEEP(*(.start))       /* 复位入口必须保留！ */
    KEEP(*(.interface_const))
} > pfls0_nc
```

典型应用：复位入口、中断向量表、陷阱向量表、BMHD 引导头、构造函数表（`__init_array_start`）等。

### 6.3 自定义段

想把自己的变量固定到某地址（比如放到 LMU 共享区让多核访问），可定义一个专属段：

```c
__attribute__((section(".lmu_data"), aligned(4)))
uint32_t shared_flag;         /* 声明放到 .lmu_data 段 */
```

链接脚本：

```ld
.lmu_data (NOLOAD) :   /* NOLOAD: 不烧录，只保留 RAM 空间 */
{
    *(.lmu_data*)
} > cpu0_dlmu
```

`NOLOAD` 关键字告诉链接器：该段只分配空间、不生成烧录数据，非常适合"共享 RAM / 掉电保持区"。

---

## 7. `ENTRY` 与 AURIX 多核启动流程

`ENTRY(_start)` 指定可执行文件的入口点，即 CPU 复位后跳转的第一条指令。

### 7.1 AURIX 的三级启动链

TC375 的上电启动比普通 MCU 复杂，分成多级：

1. **硬件复位**：CPU0 复位后先运行芯片内置的 **Boot ROM（BROM）**（不可修改的固件，位于 0x8FFF_0000，64KB）；
2. **BootROM 引导**：BootROM 读取 UCB 区（0xAF40_0000，24KB）中的 **BMHD（Boot Mode Header）**，校验后根据 BMHD 中的地址跳转到用户的 `_start`（CPU0 复位入口）；
3. **CPU0 用户启动**：执行链接脚本中的 `.start_tc0` 段代码（`_start`），完成：
   - 设置栈指针（`__USTACK0`）和 CSA 基址（`__CSA0`）；
   - 拷贝 `.data`（`_etext` → `_sdata`）；
   - 清零 `.bss`（`_sbss` → `_ebss`）；
   - 初始化 abs18 基址寄存器 A0（`_SMALL_DATA_`）；
   - 调用 `main`（core0_main）；
4. **CPU1/CPU2 启动**：CPU0 软件释放 CPU1/CPU2 的复位，后两者从各自的 `.start_tc1`/`.start_tc2` 段（`_start_cpu1`/`_start_cpu2`）开始，**每个核运行一份独立的 C 运行时启动代码**，各自有自己的 DSPR、栈、CSA，然后进入 `core1_main`/`core2_main`。

### 7.2 BMHD（Boot Mode Header）：启动链的"钥匙"

BMHD 是写入 **UCB（User Configuration Block）** 中的信息结构，告诉 BootROM"用户代码在哪、用什么模式启动"。TC375 在 UCB 中定义了 **4 组 BMHD（BMHD0~BMHD3）**，每组包含**原档（ORIG）和副本（COPY）**：

| BMHD 位置 | 地址 | 说明 |
|---|---|---|
| `UCB_BMHD0_ORIG` | 0xAF40_0000 | BMHD0 原档 |
| `UCB_BMHD1_ORIG` | 0xAF40_0200 | BMHD1 原档 |
| `UCB_BMHD2_ORIG` | 0xAF40_0400 | BMHD2 原档 |
| `UCB_BMHD3_ORIG` | 0xAF40_0600 | BMHD3 原档 |
| `UCB_BMHD0_COPY` | 0xAF40_1000 | BMHD0 副本 |
| ... | ... | 依次类推 |

> 实际烧录工具（如英飞凌 MemTool）会自动生成 BMHD 并写入上述地址，链接脚本中通常无需手动放置，但开发者应了解其结构。

**BMHD 结构**（依据 Part1 用户手册 Table 45）：

| 字段 | 大小 | 含义 |
|---|---|---|
| **BMI** | 16bit | Boot Mode Index：`HWCFG=111` 内部 Flash 启动、`011` ASC BSL、`100` 通用 BSL、`110` 备用启动模式 ABM |
| **BMHDID** | 16bit | 标识符，必须为 `B359H` |
| **STAD** | 32bit | **用户代码起始地址**（必须位于 PFLASH 内、字对齐）——这就是 BootROM 跳转的目标 |
| **CRCBMHD** | 32bit | 整个 BMHD 的 CRC 校验值 |
| **CRCBMHD_N** | 32bit | CRC 校验值取反 |

**SSW 校验流程（17 步）**：BootROM 依次检查 BMHD 状态（CONFIRMED/UNLOCKED）→ BMHDID → BMI 合法性 → STAD 是否为 PFLASH 内字对齐地址 → CRC 校验 → 判断是否需要由 HWCFG 引脚选择启动模式 → 最终得到 `BOOT_CFG` 并跳转到 `STAD`。

> **与链接脚本的联系**：`STAD` 值必须与链接脚本中 `.start_tc0` 段（`_start`）的实际地址一致。官方 AURIX 脚本把 `.start_tc0` 固定在 `0xA000_0000`（PFLASH0 非缓存镜像起始），因此烧录工具默认把 STAD 写成 `0xA0000000`。**改动了链接脚本的启动段地址，就必须同步更新 BMHD.STAD**，否则上电后 BootROM 会跳到错误位置。

### 7.3 每个核要独立维护的东西

| 项目 | CPU0 | CPU1 | CPU2 |
|---|---|---|---|
| 复位入口段 | `.start_tc0` | `.start_tc1` | `.start_tc2` |
| 陷阱向量表 | `.traptab_tc0` | `.traptab_tc1` | `.traptab_tc2` |
| 中断向量表 | `.inttab_tc0_xxx` | `.inttab_tc1_xxx` | `.inttab_tc2_xxx` |
| 用户栈 | `__USTACK0` | `__USTACK1` | `__USTACK2` |
| 中断栈 | `__ISTACK0` | `__ISTACK1` | `__ISTACK2` |
| CSA | `__CSA0` | `__CSA1` | `__CSA2` |
| 私有数据 RAM | DSPR0 (0x70000000) | DSPR1 (0x60000000) | DSPR2 (0x50000000) |
| 私有程序 RAM | PSPR0 (0x70100000) | PSPR1 (0x60100000) | PSPR2 (0x50100000) |

可见，**AURIX 链接脚本是"多核并行"的**：同样的 `.data/.bss/.csa/.stack` 结构要为每个核复制一份（通常用宏 `CORE_SEC` 批量生成）。启动代码里那些"魔法符号"全部来自链接脚本，**启动文件、链接脚本、每核的 CMake/链接配置必须成对维护**，这是嵌入式移植中最高频的 bug 来源之一。

---

## 8. 完整实战：TC375 多核链接脚本精讲

下面是一份参照英飞凌官方 `Lcf_Gnuc_Tricore_Tc.lsl` 简化后的 TC375 链接脚本（AURIX GCC 格式），核心是理解**多核段 + 缓存镜像 + 本地/全局地址映射**：

```ld
/* Lcf_Gnuc_Tricore_Tc.lsl —— TC375 多核链接脚本（简化版） */
OUTPUT_FORMAT("elf32-tricore")
OUTPUT_ARCH(tricore)
ENTRY(_start)

/* ========== 1. 常量定义：各核栈/CSA/堆大小 ========== */
LCF_CSA0_SIZE = 8k;
LCF_USTACK0_SIZE = 2k;
LCF_ISTACK0_SIZE = 1k;
LCF_HEAP_SIZE = 4k;

LCF_DSPR0_START = 0x70000000;
LCF_DSPR0_SIZE = 240k;
LCF_DSPR1_START = 0x60000000;
LCF_DSPR1_SIZE = 240k;
LCF_DSPR2_START = 0x50000000;
LCF_DSPR2_SIZE = 96k;

/* ========== 2. MEMORY：定义全部物理内存区域 ========== */
MEMORY
{
    /* ---- 程序 Flash：3MB + 3MB，缓存与非缓存镜像 ---- */
    pfls0    (rx!p) : org = 0x80000000, len = 3M
    pfls0_nc (rx!p) : org = 0xa0000000, len = 3M
    pfls1    (rx!p) : org = 0x80300000, len = 3M
    pfls1_nc (rx!p) : org = 0xa0300000, len = 3M

    /* ---- 数据 Flash 与用户配置块 ---- */
    dfls0    (rx!p) : org = 0xaf000000, len = 256K  /* DF0：EEPROM 模拟 */
    ucb      (rx!p) : org = 0xaf400000, len = 24K   /* UCB：BMHD0-3 + 副本 */

    /* ---- CPU0 本地 RAM：DSPR + PSPR ---- */
    dsram0_local (w!xp) : org = 0xd0000000, len = 240K  /* 本地地址 */
    dsram0       (w!xp) : org = 0x70000000, len = 240K  /* 全局地址 */
    psram0       (w!xp) : org = 0x70100000, len = 64K
    psram_local  (w!xp) : org = 0xc0000000, len = 64K

    /* ---- CPU1 本地 RAM ---- */
    dsram1_local (w!xp) : org = 0xd0000000, len = 240K
    dsram1       (w!xp) : org = 0x60000000, len = 240K
    psram1       (w!xp) : org = 0x60100000, len = 64K

    /* ---- CPU2 本地 RAM ---- */
    dsram2_local (w!xp) : org = 0xd0000000, len = 96K
    dsram2       (w!xp) : org = 0x50000000, len = 96K
    psram2       (w!xp) : org = 0x50100000, len = 64K

    /* ---- 全局共享 RAM（LMU/DLMU） ---- */
    cpu0_dlmu    (w!xp) : org = 0x90000000, len = 64K
    cpu0_dlmu_nc (w!xp) : org = 0xb0000000, len = 64K
    cpu1_dlmu    (w!xp) : org = 0x90010000, len = 64K
    cpu1_dlmu_nc (w!xp) : org = 0xb0010000, len = 64K
    cpu2_dlmu    (w!xp) : org = 0x90020000, len = 64K
    cpu2_dlmu_nc (w!xp) : org = 0xb0020000, len = 64K
}

/* ========== 3. 本地↔全局、缓存↔非缓存 地址映射 ========== */
REGION_MAP(CPU0, ORIGIN(dsram0_local), LENGTH(dsram0_local), ORIGIN(dsram0))
REGION_MAP(CPU1, ORIGIN(dsram1_local), LENGTH(dsram1_local), ORIGIN(dsram1))
REGION_MAP(CPU2, ORIGIN(dsram2_local), LENGTH(dsram2_local), ORIGIN(dsram2))

REGION_MIRROR("pfls0", "pfls0_nc")
REGION_MIRROR("pfls1", "pfls1_nc")
REGION_MIRROR("cpu0_dlmu", "cpu0_dlmu_nc")
REGION_MIRROR("cpu1_dlmu", "cpu1_dlmu_nc")
REGION_MIRROR("cpu2_dlmu", "cpu2_dlmu_nc")

/* ========== 4. 全局固定地址段 ========== */
SECTIONS
{
    /* CPU0 复位入口段：必须放 PFLASH 起始地址 0xA0000000（非缓存） */
    .start_tc0 (0xA0000000) : FLAGS(rxl)
    {
        KEEP(*(.start))
    } > pfls0_nc

    /* CPU1/CPU2 复位入口 */
    .start_tc1 (0xA0300200) : FLAGS(rxl) { KEEP(*(.start_cpu1)) } > pfls1_nc
    .start_tc2 (0xA0300220) : FLAGS(rxl) { KEEP(*(.start_cpu2)) } > pfls1_nc

    /* 陷阱（异常）向量表 */
    .traptab_tc0 (0x80000100) : { KEEP(*(.traptab_cpu0)) } > pfls0
    .traptab_tc1 (0x80300000) : { KEEP(*(.traptab_cpu1)) } > pfls1
    .traptab_tc2 (0x80300100) : { KEEP(*(.traptab_cpu2)) } > pfls1

    /* CPU0 代码段 */
    .text :
    {
        . = ALIGN(4);
        *(.text*)
        *(.rodata*)
        _etext = .;
        . = ALIGN(4);
    } > pfls0

    /* CPU0 已初始化数据：VMA 在 DSPR0，LMA 在 PFLASH0 */
    .data :
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data*)
        _edata = .;
        . = ALIGN(4);
    } > dsram0 AT> pfls0

    /* CPU0 未初始化数据：只占 DSPR0，启动时清零 */
    .bss (NOLOAD) :
    {
        _sbss = .;
        *(.bss*)
        _ebss = .;
        . = ALIGN(4);
    } > dsram0

    /* CPU0 上下文保存区：64 字节对齐 */
    .csa (NOLOAD) :
    {
        __CSA0 = .;
        . = . + LCF_CSA0_SIZE;
        __CSA0_END = .;
    } > dsram0

    /* CPU0 用户栈与中断栈：向低地址增长 */
    .ustack (NOLOAD) :
    {
        __USTACK0_END = .;
        . = . + LCF_USTACK0_SIZE;
        __USTACK0 = .;
    } > dsram0

    .istack (NOLOAD) :
    {
        __ISTACK0_END = .;
        . = . + LCF_ISTACK0_SIZE;
        __ISTACK0 = .;
    } > dsram0

    /* ======== CPU1、CPU2 的 .data/.bss/.csa/.stack 结构（简化）========
     * 实际脚本会为每个核用 CORE_SEC 宏重复上述整套段，
     * 分别定位到 dsram1/dsram2，避免跨核访问私有 RAM。 */

    /* ======== 多核共享数据：放 LMU（非缓存区保证一致性）======== */
    .lmu_data (NOLOAD) :
    {
        *(.lmu_data*)
    } > cpu0_dlmu_nc
}
```

**几个值得注意的细节**：

- `REGION_MAP` / `REGION_MIRROR` 是 AURIX GCC 扩展命令，分别解决"本地地址 vs 全局地址"和"缓存 vs 非缓存"；
- `.start_tc0` 固定在 0xA000_0000（非缓存镜像的 PFLASH 起始），因为 BootROM 通过 BMHD 跳转时按固定地址执行；
- 每个核的 `.data`/`.bss`/`.csa`/栈必须**分开放**，绝不能共享 DSPR——官方明确反对把一个核的 DSPR 当另一个核的"扩展内存"；
- 多核共享数据放 DLMU，且推荐用**非缓存**地址（`cpu0_dlmu_nc`），避免 cache 一致性问题；
- `.bss`/`.csa`/栈用 `(NOLOAD)`，只分配空间不烧录；
- `__USTACK0` 指向栈顶（栈向低地址增长），`__USTACK0_END` 是栈底。

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
| `REGION_MAP(cpu, local_org, len, global_org)` | AURIX：本地↔全局地址映射 |
| `REGION_MIRROR(cached, noncached)` | AURIX：缓存↔非缓存镜像 |
| `FLAGS(rxl/awzl/...)` | AURIX：段属性（可加载/可分配/可零初始化） |

### 9.1 `PROVIDE` 的妙用

```ld
PROVIDE(_SMALL_DATA_ = _sdata);
```

如果目标文件中没有定义 `_SMALL_DATA_`，链接器就提供它（指向近数据段起始，供 TriCore 编译器初始化 A0 基址寄存器）；如果用户代码已定义则跳过。这给第三方库（如 RTOS、AURIX iLLD 库）提供了"缺省但不覆盖"的友好接口。

### 9.2 `ASSERT`：把错误挡在烧录之前

```ld
ASSERT(_ebss <= ORIGIN(dsram0) + LENGTH(dsram0),
       "DSPR0 溢出：.bss 超出 RAM 范围！")
```

链接时报错比运行时死机好排查一万倍，善用 `ASSERT`。AURIX 官方脚本里还常见这样的校验：

```ld
ASSERT(_ebss0 < _stack0_begin, "DSPR0 数据与栈冲突！")
```

---

## 10. 常见报错与调试技巧

### 10.1 经典报错对照表

| 报错信息 | 原因 | 解决办法 |
|---|---|---|
| `section .text will not fit in region pfls0` | 代码超出 Flash | 优化代码/开启 `-Os`，检查有无死代码，或把部分代码挪到 pfls1 |
| `region dsram0 overflowed by N bytes` | 该核 RAM 超限 | 查数组、全局变量；`.bss` 过大；把大数组挪到 LMU |
| `undefined reference to _sdata` | 启动文件与脚本符号不一致 | 对照启动文件逐一核对符号名 |
| `relocation truncated to fit: R_TRICORE_...` | 跳转/寻址超出范围 | abs18 数据超出低 512KB、32 位跳转超范围，检查段地址 |
| `_SMALL_DATA_ not defined` | 近数据段基址符号缺失 | 在脚本中 `PROVIDE(_SMALL_DATA_ = _sdata)` |
| `cannot find entry symbol _start` | `ENTRY` 写错/函数名拼写不一致 | 确认 `ENTRY` 与启动文件函数名一致 |
| `multiple definition of xxx` | 全局变量重复定义 | 检查 `-fno-common` 与头文件定义 |
| `region 'ucb' overflowed` | BMHD 引导头放错位置 | 检查 BMHD 结构体是否准确放在 0xAF400000 各槽位 |
| 上电后不进 main、程序停在 BootROM | **BMHD.STAD 与链接脚本启动段地址不一致** | 用 MemTool 重新烧录 BMHD，确保 STAD=0xA0000000（与 `.start_tc0` 对齐） |
| 三核中只有某个核不跑 | 该核栈/CSA 配置错误或 DSPR 被占用 | 检查每核独立的 `.csa`/栈符号与大小 |

### 10.2 调试工具：`nm`、`objdump`、`size`

排查链接脚本最有效的三板斧（TriCore 工具链前缀可能是 `tricore-` 或 `hightec-`）：

```bash
# 1. 查看符号最终地址（确认多核符号都落到对应 DSPR/PFLASH）
tricore-elf-nm -n firmware.elf

# 2. 反汇编，检查跳转目标是否合理
tricore-elf-objdump -d firmware.elf

# 3. 查看各段占用
tricore-elf-size -A firmware.elf

# 4. 完整段布局与 VMA/LMA（最有价值）
tricore-elf-objdump -h firmware.elf
```

`objdump -h` 会列出每个段的 **VMA 和 LMA**，一眼就能看出 `.data` 是否正确地"住在 DSPR、源于 PFLASH"，以及三个核的数据是否各归其位：

```
Sections:
Idx Name      Size    VMA       LMA       File off  Algn
  0 .start_tc0 00000080 a0000000 a0000000  00010000  2**2
  1 .text     000023c8 800000c0 800000c0  000100c0  2**2
  2 .data     00000010 70000000 80002488   ...
  3 .bss      00000230 70000010 70000010   ...
  4 .csa      00002000 70080000 70080000   ...
```

### 10.3 打印链接 Map 文件

编译时加 `-Wl,-Map=output.map` 生成 Map 文件，这是排查内存布局的"说明书"，包含所有符号地址、段归属、废弃段等完整信息：

```bash
tricore-elf-gcc ... -Wl,-Map=firmware.map
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

配合 `KEEP()` 保留必须的段（复位入口、向量表、BMHD、构造函数表），可以在不牺牲功能的情况下显著减小固件。

> ⚠️ 注意：用了 `-ffunction-sections` 后，脚本中应使用 `*(.text*)` 而非 `*(.text)`，否则段匹配不上，会得到"空段"。

---

## 12. 与其他工具链的异同

虽然本文以 AURIX GCC `ld` 为主线，但链接脚本思想是通用的：

| 工具链 | 链接脚本机制 |
|---|---|
| **AURIX GCC (HighTec)** | `.lsl` / `.ld` 文件，`ld` 语法 + `REGION_MAP`/`REGION_MIRROR` 扩展（本文内容） |
| **TASKING (英飞凌官方)** | `.lsl` 文件，使用 `LSL`（Linker Script Language），以 `memory/group/section_layout` 描述多核布局 |
| **GCC (arm-none-eabi)** | `.ld` / `.x` 文件，`ld` 语法 |
| **IAR** | `.icf` 文件，`place in` 等关键字 |
| **Keil MDK** | `.sct` 文件，`LR_` / `ER_` 加载区与执行区 |

TASKING 的 `.lsl`（TC375 官方默认工具链之一）写法完全不同，但目标一样——描述三核的内存布局：

```
// Lcf_Tasking_Tricore_Tc.lsl（TASKING 格式）
derivative tc37
{
    core tc0 { architecture = TC1V1.6.2; copytable_space = vtc:linear; }
    core tc1 { architecture = TC1V1.6.2; copytable_space = vtc:linear; }
    core tc2 { architecture = TC1V1.6.2; copytable_space = vtc:linear; }
}

memory dsram0
{
    mau = 8;
    size = 240k;
    type = ram;
    map (dest=bus:tc0:fpi_bus, dest_offset=0xd0000000, size=240k, priority=8);
    map (dest=bus:sri, dest_offset=0x70000000, size=240k);
}

memory pfls0
{
    mau = 8;
    size = 3M;
    type = rom;
    map cached     (dest=bus:sri, dest_offset=0x80000000, size=3M);
    map not_cached (dest=bus:sri, dest_offset=0xa0000000, size=3M);
}
```

IAR 的 `.icf`：

```
define memory PFLASH0 with size = 3M, start = 0x80000000;
define memory DSPR0   with size = 240K, start = 0x70000000;
place in PFLASH0 { readonly section .text };
place in DSPR0   { readwrite section .data };
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

**换了 IDE/工具链就换一套链接描述语言，但"VMA / LMA / 段 / 位置计数器 / 多核私有与共享内存"这些概念完全相通**——理解 GCC 脚本后，切换到 TASKING/IAR/Keil 只是换语法而已。

---

## 13. 总结

链接脚本是嵌入式工程师从"会写 C"走向"懂底层"的分水岭。以 TC375 为例回顾本文核心要点：

1. **链接器**负责符号解析与重定位，裸机程序必须用链接脚本定义内存布局；
2. **MEMORY** 定义芯片可用的内存区域与属性——TC375 上要分清 PFLASH0/1（各 3MB）、三组 DSPR/PSPR、LMU/DLMU（各 64KB）、DAM（32KB）、DFLASH0/1、UCB、BROM；
3. **SECTIONS** 定义输出段如何由输入段组成、放在哪块区域；
4. **VMA 与 LMA** 是理解 `.data` 拷贝、启动代码的钥匙；
5. **AURIX 多核特性**：`REGION_MAP`（本地/全局地址）、`REGION_MIRROR`（缓存/非缓存）、16 个 256MB 段的划分、每核独立的 `.csa`/栈/`.data`/`.bss`；
6. **`.`（位置计数器）、`ALIGN`、`KEEP`、`NOLOAD`** 是日常最常用的控制手段；
7. **启动链三件套**：BootROM → BMHD（含 BMI/BMHDID/STAD/CRC）→ `.start_tc0`，其中 **STAD 必须与链接脚本启动段地址一致**；
8. **启动文件与链接脚本是一体两面**，符号必须严格对应，多核项目更是如此；
9. 善用 `ASSERT`、`Map 文件`、`objdump -h` 把内存问题扼杀在链接阶段。

理解链接脚本之后，你会发现自己对"程序到底怎么跑起来的"有了更清晰的画面：BootROM 读 BMHD → CPU0 从 0xA0000000 的 `_start` 启动 → 初始化栈与 CSA → 搬数据 → 清 BSS → 调 core0_main → 再释放 CPU1/CPU2 让它们各自从自己的入口跑起来……而这一切的幕后导演，正是这份被无数人"复制粘贴"却鲜有人真正读懂的 **Linker Script**。

---

*参考文档：GNU ld 手册（`info ld`）、Infineon **AURIX TC3xx 用户手册 Part 1**（V2.0.0, 2021-02，第2章 MEMMAP / 第3章 SSW 启动 / Table 45 BMHD 结构）、**AURIX TC37x 用户手册 Appendix**（V2.0.0，第2章 MEMMAP 设备级内存映射）、TC37x 数据手册（V1.1）、英飞凌官方 AURIX Development Studio 例程 `Lcf_Gnuc_Tricore_Tc.lsl` / `Lcf_Tasking_Tricore_Tc.lsl`。*
