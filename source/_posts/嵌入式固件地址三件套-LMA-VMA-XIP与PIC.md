---
title: 嵌入式固件地址三件套：加载地址、运行时地址、XIP 与 PIC
date: 2026-08-02 10:30:00
tags:
  - 嵌入式
  - LMA
  - VMA
  - XIP
  - PIC
  - 链接脚本
  - Linker Script
  - RT-Thread
  - 英飞凌
  - TC375
  - AURIX
  - TriCore
  - SOTA
---

# 嵌入式固件地址三件套：加载地址、运行时地址、XIP 与 PIC

> 嵌入式开发中最基础也最容易混淆的一组概念，几乎所有底层问题（启动流程、重定位、内存布局、OTA 升级）都跟它们有关。本文用一个 RT-Thread 真实工程和英飞凌 TC375 SOTA 方案把四件事讲透。

## 一、基本概念：加载地址与运行时地址

| 概念 | 英文 | 含义 |
|---|---|---|
| **加载地址** | **LMA**（Load Memory Address） | 程序（代码/数据）**存放**在非易失存储介质（Flash、ROM）中的物理位置，即"**烧录进去时放在哪里**" |
| **运行时地址** | **VMA**（Virtual Memory Address） | 程序**执行**时，CPU 实际**访问/寻址**的内存地址，即"**跑起来时在哪里**" |

一句话总结：

> **加载地址 = 睡觉的地方（存放），运行时地址 = 干活的地方（执行）。**

两者**不一定相等**，这是整个话题的核心。

## 二、为什么会有两套地址？

| 原因 | 说明 |
|---|---|
| **存储介质运行太慢** | Flash 读取速度远慢于 RAM，大程序直接在 Flash 执行会拖慢系统 |
| **存储介质不能随机访问** | NAND Flash 按块/页访问，**根本无法直接取指执行**，必须先把代码搬到 RAM |
| **指令/数据必须可写** | `.data`（有初值的全局变量）运行时**要被修改**，而 Flash 不可随机写 |
| **外设初始化依赖 RAM** | SDRAM 初始化需要代码先运行，而代码本身又在 SDRAM 里 → 启动早期必须在片内 SRAM 里"借宿" |

经典方案：

```
+------------------ Flash (LMA) ------------------+
|  .text (代码)  |  .rodata (常量) |  .data 初值  |
+------------------------------------------------+
        | 上电                    | 拷贝搬运
        v                        v
+------------------ RAM (VMA) --------------------+
|        .text 执行区(可选)      |  .data | .bss |
+-------------------------------------------------+
```

### 程序段与 LMA/VMA 的关系

| 段 | 内容 | 初始值存在哪 | 运行时在哪 | LMA==VMA? |
|---|---|---|---|---|
| `.text` | 可执行代码 | Flash | Flash（XIP）或 RAM（拷贝执行） | 看配置 |
| `.rodata` | 只读常量 | Flash | Flash | 通常相等 |
| `.data` | **已初始化**的全局/静态变量 | **初值存 Flash** | **RAM** | **不相等** |
| `.bss` | **未初始化**的全局/静态变量 | 无 | RAM（上电清零） | 不相等 |
| `.stack`/`.heap` | 栈/堆 | 无 | RAM | 不相等 |

**关键点**：`.data` 段的 LMA 在 Flash、VMA 在 RAM，Flash 里存的是"初始值的备份"，启动时由代码拷到 RAM；`.bss` 段甚至不占 Flash 空间，只需清零。

## 三、链接脚本中的 LMA / VMA（GCC 语法）

```ld
SECTIONS
{
    /* VMA 指向 Flash：代码直接在 Flash 执行 (XIP) */
    .text : {
        *(.text)  *(.rodata*)
    } > FLASH                /* 只写一个区段 -> LMA==VMA */

    /* VMA 指向 RAM，LMA 指向 Flash（AT> 关键字） */
    .data : {
        *(.data)
    } > RAM AT> FLASH        /* VMA 在 RAM，LMA 在 Flash */

    /* BSS：只有 VMA，没有 LMA */
    .bss : { *(.bss) } > RAM
}
```

- `> FLASH` 后面定义的是 **VMA**（运行时地址）；
- `AT> FLASH` 专门指定 **LMA**（加载地址），没写 `AT>` 时 LMA 默认等于 VMA；
- 链接器生成 `ADDR(section)`（VMA）与 `LOADADDR(section)`（LMA）符号供启动代码使用。

### 启动代码负责"搬运"

```c
extern char _sdata, _edata;   /* .data 的 RAM 范围 (VMA) */
extern char _sidata;          /* .data 在 Flash 中的备份 (LMA) */
extern char _sbss, _ebss;     /* .bss 范围 */

void Reset_Handler(void)
{
    char *src = &_sidata;             /* 加载地址 */
    char *dst = &_sdata;              /* 运行时地址 */
    while (dst < &_edata) *dst++ = *src++;   /* 拷贝 .data */
    for (char *p = &_sbss; p < &_ebss; p++) *p = 0;   /* 清零 .bss */
    main();
}
```

## 四、XIP：就地执行

> **XIP（Execute In Place）= 代码"躺哪就干哪"，CPU 直接从 Flash/ROM 取指执行**，此时 `.text` 段 **LMA == VMA**。

### XIP 的前提条件

| 条件 | 说明 | 反例 |
|---|---|---|
| **可随机访问** | CPU 能按字节/字随机寻址取指 | **NAND Flash** 不行 |
| **可映射到统一地址空间** | Flash 挂在系统总线上 | SD 卡、U 盘不行 |
| **读速度可接受** | 取指延迟不能拖垮 CPU | 慢速 SPI Flash 勉强 |

主流可 XIP 介质：**NOR Flash**（经典）、ROM、PSRAM/SRAM、DDR SDRAM，以及部分 QSPI 的 Memory-Mapped 模式。

### XIP 的优缺点

| 优点 | 缺点 |
|---|---|
| 省 RAM（代码不占宝贵 RAM） | 执行慢（Flash 读速度远低于 RAM） |
| 启动快（无需搬运大段代码） | 写操作不可 XIP（变量、堆栈仍要放 RAM） |
| 结构简单，无需重定位 | 代码段不支持自修改 |
| OTA/升级方便 | 频繁访问 Flash 增加功耗磨损 |

### 工程取舍

```
代码量小 + RAM紧张       → 全 XIP（多数 MCU 默认，如 STM32）
代码量小 + 性能关键       → 关键函数拷贝到 RAM（RAM-Function），其余 XIP
代码量大 + 性能要求高     → 全量拷贝到 RAM 执行
```

**RAM-Function 经典用途**：Flash 擦写/升级过程中，代码如果还在 Flash 里跑会"自食"（一边跑一边擦自己）→ 擦写函数必须在 RAM 里执行。这就是后面 SOTA 里常见的 `copyFunctionsToPSPR()`。

### XIP 的经典坑

1. **代码自修改**：运行在 Flash 的代码写代码段 → 崩溃；
2. **Flash 擦写期间 XIP 代码"自杀"**：擦写函数的代码在被擦的扇区里，擦到一半取指失败 → 解法是擦写函数放 RAM；
3. **缓存一致性**：Flash 被 DMA/外设改写时需维护 I-Cache 失效；
4. **取指等待周期**：Flash 读速度 < CPU 主频，需配置 Wait State + 指令缓存。

## 五、PIC：位置无关代码

> **PIC（Position Independent Code）= 代码里不写死绝对地址，全部用相对寻址，搬到哪都能执行。**

### 问题的提出

```
编译链接时：代码按 VMA = 0x30000000 编址（跳转、取变量都写绝对地址）
实际运行前：代码在 SRAM 0x00000000 里（前 4KB 借宿阶段）
问题：代码里写死 0x30000000 的跳转，在 0x00000000 执行 = 取不到指令 = 崩溃
```

两种解法：**重定位（Relocation）**——运行时按实际位置改写指令中的地址（需可写介质）；**PIC**——编译时根本不生成绝对地址，全部用 PC 相对寻址（无需改代码，牺牲少量性能/体积）。

### PIC 怎么做到"处处能跑"

1. **跳转** → 用 PC 相对分支指令（`B`/`BL`），天然位置无关；
2. **取立即数/地址** → PC 相对字面量池（Literal Pool）：`LDR r0, [PC, #0x1C]`；
3. **访问全局变量/函数指针** → **GOT（全局偏移表）**：

```
                   PIC 代码                       GOT 表（放 .data 区）
   LDR  r3, [PC, #offset]   --------->   GOT[0] = &global_var
                                          GOT[1] = &func
                  相对 PC 取表项地址，       表里存运行时真实地址
                  再间接解引用
```

4. x86-64 原生支持 `RIP-relative` 寻址，编译 PIC 几乎零额外开销；ARM64 同理（`adrp`+`ldr`）。

### 编译器开关

| 选项 | 含义 |
|---|---|
| `-fPIC` / `-fpic` | 位置无关代码（共享库用） |
| `-fPIE` / `-fpie` | 位置无关可执行文件（配合 ASLR 防漏洞） |
| `-shared` | 配合 `-fPIC` 生成共享库 |

### 三种方案横向对比

| | 绝对编址（默认） | 重定位（Relocation） | PIC |
|---|---|---|---|
| 指令中地址 | 写死 VMA | 写死，但附补丁表 | 只写偏移量 |
| 换位置运行 | ❌ 崩溃 | ✅ 搬运后打补丁 | ✅ 直接跑 |
| 代码是否可写 | 无所谓 | **必须 RAM 可写** | 可只读 |
| 性能/体积 | 最优 | 略逊 | 最差（间接寻址） |
| 典型用途 | 链接到固定地址的 App | Bootloader 搬运后 | 共享库、启动早期、双 bank |

**一条主线总结**：
```
LMA≠VMA 时怎么跑？
   ├─ 介质可写且地址确定 -> 重定位打补丁
   ├─ 介质不可写/位置不确定 -> PIC
   └─ 代码就地执行 -> XIP（LMA==VMA）
```

## 六、实战①：RT-Thread RA2A1-EK 工程解析

> 以 Renesas RA2A1（Cortex-M23）上运行 RT-Thread 的真实工程为例，用 `.map` 文件实测数据说话。

### 内存区域

```
0x00000000    0x400      FLASH_GAP（向量表 + 复位启动代码 + 链接器表）
0x00000400    0x40       选项设置寄存器区 (OFS0/OFS1/SECMPU)
0x00000440    0x3fbc0    FLASH 主区（真正的 .text / .rodata）
0x20000000    0x8000     RAM（.data / .bss / 栈 / 堆）
```

### 链接脚本关键段（fsp_gen.ld）

```ld
/* ① 复位向量表 —— VMA=LMA=0x00000000（XIP） */
__flash_vectors$$ : { KEEP(*(.fixed_vectors)) ... } > FLASH_GAP

/* ② 启动早期代码（Reset_Handler/SystemInit）—— XIP */
__flash_readonly_gap$$ : { ... } > FLASH_GAP

/* ④ ★ 数据段：VMA 在 RAM，LMA 在 Flash —— 必须搬运 ★ */
__ram_from_flash$$ : { *(.data*) } > RAM AT > FLASH

/* ⑤ 零初始化段：NOLOAD 不占 Flash */
__ram_zero$$ (NOLOAD) : { *(.bss*) } > RAM

/* ⑦ 全部应用代码 —— VMA=LMA，XIP */
__flash_readonly$$ : { *(.text*) *(.rodata*) } > FLASH
```

**规则**：段只写 `> FLASH` → LMA==VMA（XIP）；写 `> RAM AT> FLASH` → VMA 在 RAM、LMA 在 Flash（搬运）；写 `NOLOAD` → 只有 VMA。

### .map 文件实测地址

```
__flash_vectors$$          VMA 0x00000000   LMA 0x00000000   size 0x50   ← XIP
__flash_readonly_gap$$     VMA 0x00000050   LMA 0x00000050   size 0x364  ← XIP
   .text.Reset_Handler               0x8c
   .text.SystemRuntimeInit           0x9c
__ram_from_flash$$ (.data) VMA 0x20000000   LMA 0x000004ec   size 0x334  ← 不等！要拷
__ram_noinit$$ (栈/堆)     VMA 0x20000338   NOLOAD        size 0x414
__ram_zero$$ (.bss)        VMA 0x20000750   NOLOAD        size 0xba4
__flash_readonly$$ (主代码) VMA 0x00000820   LMA 0x00000820   size 0x119a0 ← XIP
```

**本工程关键事实**：`.text/.rodata` 全部 XIP 就地执行（LMA==VMA==0x820），只有 `.data`（0x334 字节）从 Flash 0x4ec 拷到 RAM 0x20000000，`.bss` 清零。

### 启动流程

```c
void Reset_Handler(void)
{
    SystemInit();      /* 时钟/硬件初始化 + C 运行时初始化 */
    entry();           /* crt0 → main() → hal_entry() */
}
```

真正的搬运在 `SystemRuntimeInit()`，它读的是链接器生成的**搬运表**（bsp_linker_info.h 用 `__ram_from_flash$$Base/Limit/Load` 填充，即 VMA/LMA 具体数值）——"照着表搬"，无需硬编码地址。

### 反汇编佐证：为什么这里用不到 PIC

```asm
0000008c <Reset_Handler>:
  90:  f000 f89a     bl     1c8 <SystemInit>     ← PC 相对跳转（天然位置无关）
  94:  f009 fc16     bl     98c4 <entry>

0000009c <SystemRuntimeInit>:
  aa:  f240 037c     movw   r3, #0x7c           ← ★ 直接加载"绝对地址"0x7c
  ae:  f2c0 0300     movt   r3, #0              （g_init_info 的 VMA）
```

- 跳转用 **PC 相对**（天然位置无关）；
- 访问全局数据却用 **绝对地址**（movw/movt 写死 VMA）。

**为什么可以放心用绝对地址？** 因为这是固定链接固件，代码就在 Flash 里原地跑（XIP），LMA==VMA，**运行位置永远不变** → 绝对寻址完全可靠。这就是**裸机/RTOS 程序默认不用 PIC 的原因**。

## 七、搬运与交接：谁把代码搬到 RAM，搬完怎么跑起来

> 前面讲了"搬什么、搬去哪"，这里回答两个更深入的问题：**谁在哪个阶段搬**、**搬完后怎么把执行流交到 RAM 里**。

### 7.1 谁在哪个阶段把程序搬到 RAM

搬运动作发生在"复位后、main() 之前"的启动阶段，由**用户程序的启动代码**执行。以 TC375 为例，一条完整的搬运链分三个阶段：

```
┌─ 阶段0 ─────────────────────────────────────────────┐
│ 上电/复位                                            │
│   CPU0 从 BootROM(BROM) 开始执行 —— ROM 里的固件，    │
│   不可修改，不属于你的程序                            │
│   · 读 BMI/BMHD（UCB 配置块）                        │
│   · CRC 校验 Boot Header                            │
│   · 通过 → 跳到用户程序起始地址 STAD                  │
│   · 失败 → 进入 Bootstrap Loader（CAN/ASC 加载）     │
│   ★ BootROM 只负责"跳"，不搬你的代码                 │
└─────────────────────────────────────────────────────┘
              │ 跳到 App 入口 __START (0x80000000, XIP)
              ▼
┌─ 阶段1 ─────────────────────────────────────────────┐
│ App 的启动代码（cstart / __START / _c_init）        │
│   ★ 它跑在 PFLASH 里（XIP），但干的是"搬"的活：     │
│   ① 设 SP / CSA                                    │
│   ② 处理链接器生成的 copy table：                   │
│      .data 段             LMA(Flash) → VMA(RAM)     │
│      需要 RAM 运行的代码段  LMA(Flash) → VMA(PSPR)  │
│   ③ 处理 clear table：.bss 清零                     │
│   ④ 设 BIV/BTV 向量表                              │
│   ★ 这是"把早期存在 PFLASH 的程序搬到 RAM"的主力     │
└─────────────────────────────────────────────────────┘
              │ main() / core0_main()
              ▼
┌─ 阶段2 ─────────────────────────────────────────────┐
│ 运行期间（例如 SOTA 下载时）                        │
│   copyFunctionsToPSPR() —— 由运行中的 App 调用      │
│   把 Flash 擦写函数从 PFLASH 拷到 PSPR，切换函数指针 │
│   ★ 这是"运行期主动搬运"：目的：不在被擦的 bank 上取指│
└─────────────────────────────────────────────────────┘
```

**结论：阶段 1 的 App 启动代码是主力搬运者**；阶段 2 的 `copyFunctionsToPSPR()` 是特殊场景下的运行期搬运；BootROM 全程不参与搬运。

### 7.2 搬运表是谁生成的？——链接器

启动代码能搬，是因为**链接器**在链接时（根据 `.lsl` 脚本）生成了搬运表：

```lsl
section_layout :vtc:linear
{
    /* 代码段：运行地址在 PSPR(RAM)，加载地址在 PFLASH */
    group TC0_FUNCTIONS ( ordered, run_addr = mem:mpe:pspr0, copy )
    {
        select ".text.file_1.init_func";   /* 生成 ROM copy 段 [.text.xxx] */
    }
}
```

链接器生成的 map 里可以看到两个地址：

```
mpe:pflash0 | [.text.file_1.init_func]   LMA = 0x800000ac   ← 代码原始存放处
mpe:pspr0   |  .text.file_1.init_func    VMA = 0x70100000   ← 代码最终运行处
```

同时链接器生成一张 **copy table**，每条记录三个量：**源地址(LMA)、目标地址(VMA)、长度**。启动代码就是"照着这张表逐条 memcpy"。与前面 RA2A1 工程的 `__ram_from_flash$$Base/Limit/Load` 是同一套机制——只是 ARM 用链接器符号，TriCore 用 copy table，本质都是"LMA→VMA 清单"。

### 7.3 为什么搬运者能安心跑在 Flash 里？

因为**源在 Flash、目标在 RAM，二者不重叠**——搬运过程中 Flash 里的代码不会被自己改，所以"从 Flash 拷到 RAM"这个动作在 Flash 里执行完全安全。

**那什么情况会"自杀"？** 当你要**擦掉/改写 Flash 源区域**时（SOTA 下载、OTA 升级），如果还从 Flash 取指就会崩。所以两步走：

```
第 1 步：还在 Flash 跑 → 把擦写函数拷到 PSPR（copyFunctionsToPSPR）
第 2 步：把函数指针切到 PSPR 地址
第 3 步：此时擦写 Flash 才安全（取指在 PSPR，擦的是 Flash）
```

### 7.4 搬运结束后：怎么把执行流交到 RAM

核心结论：**通过"跳转到 RAM 里的 VMA 地址"完成交接**。能直接跳的原因是——**代码的链接地址就是 RAM 地址（VMA），内部所有引用在链接时都已按 VMA 解析好**，只要拷贝时"精确落到 VMA"，跳过去就能跑。

| 代码内部的引用 | 为什么换地方跑还能用 |
|---|---|
| **跳转指令**（B/CALL/BL） | PC 相对寻址，**天然位置无关**，偏移量不变 |
| **数据访问**（全局变量、常量池） | 绝对寻址，指向**数据的链接地址**——数据在它该在的地方就正常 |

三种交接方式：

**方式 A：函数指针（最常见，SOTA 擦写例程标准做法）**

```c
typedef void (*pFlashFunc_t)(void);
static pFlashFunc_t eraseFunc;    /* 函数指针，将指向 PSPR 里的副本 */

void copyFunctionsToPSPR(void)
{
    /* ① 还在 Flash 里跑：把擦写函数拷到 PSPR */
    memcpy((void*)0x70100000, (const void*)0x800000ac, size);
    /* ② 交接：把函数指针指向 RAM 里的副本地址（VMA） */
    eraseFunc = (pFlashFunc_t)0x70100000;
}

void doErase(void)
{
    eraseFunc();   /* ③ 调用 —— 实际跳进 PSPR 里的代码执行 */
}
```

`eraseFunc()` 编译出来是**间接调用**：ARM 是 `BLX rX`、TriCore 是 `CALLI rX`，目标在寄存器里，**不受范围限制、能跳任意地址**——这就是交接的载体。

**方式 B：直接长跳转 / 硬跳（整个 App 在 RAM 里跑时）**

```c
void (*entry)(void) = (void(*)(void))0x90000000;
entry();              /* 直接跳到 RAM 里的 App 入口 */
```

```asm
; ARM: 间接跳转
LDR   r0, =0x90000000
BX    r0              ; 注意：最后一位是 ARM/Thumb 状态标志，bit0 必须=1

; TriCore: 间接跳转
mov.a  a0, 0x90000000
JI     a0             ; Jump Indirect，跳到 RAM 入口
```

**方式 C：硬件向量表跳入（ISR 在 RAM 的场景）**

中断到来时不是"谁去调"，而是**硬件到点自动跳**：向量表某项存 ISR 的 VMA（RAM 地址），中断触发 → 硬件读向量 → 跳进 RAM 里的 ISR。交接的对象是**表的基址寄存器**：ARM 设 `VTOR`、TriCore 设 `BIV` 指向 RAM 里的表。

### 7.5 RAM 函数怎么"回头"？——返回机制与坑

```
Flash 里的调用者 (0x800000xx)
    │  CALL 到 RAM (0x70100000)    ← 返回地址被压栈/保存
    ▼
RAM 里的函数执行 (0x70100000)
    │  内部跳转、数据访问都 OK
    │  RET / BX LR                 ← 弹出返回地址
    ▼
回到 Flash 里的调用者，继续跑
```

- ARM：`BLX rX` 自动把返回地址存 LR，函数末尾 `BX LR` 回来；
- TriCore：`CALLI` 把返回地址存进 CSA（上下文保存区），`RET` 从 CSA 弹出。

**关键约束**：返回地址指向"调用者所在的 Flash"——所以**如果调用者所在的那块 Flash 正在被擦写，就回不去了**。这正是 SOTA 里擦写例程要"自包含"的根本原因：

```
✅ 正确：擦写函数拷到 PSPR 后自包含运行（内部不调用仍留在被擦 bank 的代码）
❌ 错误：PSPR 里的擦写函数调用了 active bank 里的辅助函数，然后擦同一块 bank → 调一半取指失败
```

### 7.6 注意事项速查

| 坑 | 说明 |
|---|---|
| **必须拷到精确 VMA** | 拷贝目标地址错 1 字节，绝对寻址的全局数据访问全错 |
| **调用者所在 Flash 不能被擦** | 返回地址指向调用点，擦了就回不去 → 擦写例程要自包含 |
| **拷贝者自身安全** | memcpy/搬运代码要跑在"不会被擦的存储"里 |
| **I-Cache / 缓存一致性** | **DMA** 拷贝代码进 RAM 时需 flush/invalidate 指令缓存再跳；CPU 用 memcpy 搬则天然一致 |
| **字节对齐/浮点状态** | 代码按函数边界、4 字节对齐拷贝；浮点上下文注意 CPn 使能 |

> 一句话：**"拷到哪"是 copy table 的事，"跳过去"是函数指针/跳转指令的事，"回来"是调用约定的事——三者对了，RAM 程序就跑起来了。**

## 八、实战②：TC375 SOTA 到底用不用 PIC？

> 关键事实（Infineon 官方文档确认）：**TC3xx 官方 SOTA 是硬件 bank 交换（SWAP），不需要 PIC。**

### TC375 内存格局

| 资源 | 地址 | 大小 |
|---|---|---|
| PFLASH0 (PF0) | `0x80000000` | 3MB |
| PFLASH1 (PF1) | `0x80300000` | 3MB |
| LMU | `0x90000000` | 512KB |
| CPU0 DSPR / PSPR | `0x70000000` / `0x70100000` | 96KB / 32KB |

### 硬件 SWAP 机制

```
PFLASH 分为 A、B 两组 bank

  Active bank（PF0）→ 映射到可执行地址 0x80000000
  Inactive bank（PF1）→ 映射到"可读写"的其它地址

  更新完成 → 交换 → 【只改变地址映射，不拷贝任何数据】
  交换后 App 依然从 0x80000000 取指执行
```

官方文档三句话是关键：

- *"only the address mapping will change. no data needs to be copied and the address ranges being executed from are always the same."*
- *"parameters are pre-configured in a User Configuration Block (UCB) and only updated by on-chip system firmware during the subsequent System Reset."*
- *"All NVM operations always use the standard address map regardless of swap settings."*

**所以 App 只需一份镜像、一个链接地址（0x80000000），永远从同一个地址执行**——硬件帮你"重定位"了，PIC 纯属多余。

### 硬件方案的代价

| 代价 | 说明 |
|---|---|
| 性能下降 | 必须禁用 CPU 到本地 PFLASH 的 fast path，保证 A/B bank 执行性能一致 |
| 部分 Flash 不可用 | 在 1MB+3MB 变体上，3MB 组的上 2MB 不能放代码 |
| 交换时机受限 | 只能在系统复位时由片上固件按 UCB 配置切换 |

### 什么情况下 TC375 SOTA 才需要 PIC？

| SOTA 方案 | App 运行地址 | 需要 PIC？ |
|---|---|---|
| 官方硬件双 bank 交换 | 永远是 0x80000000 | ❌ |
| 双构建（A/B 各编一次） | 两个固定地址 | ❌ |
| 搬移到 RAM 执行 | 固定 RAM 地址 | ❌ |
| 单镜像 + 软件跳转两个不同 Flash 地址 | 两个地址 | ✅ **必须** |

**易混点**：Flash 擦写函数拷到 PSPR 运行 **≠ PIC**，那是"拷贝到固定 RAM 地址执行"，属于重定位/拷贝执行，SOTA 下载阶段都要做。

### 对比：STM32 vs TC375 的"换 bank"

```
STM32（无硬件重映射）：
  物理 bank 地址固定不变，App 换 bank 后运行地址变了
  → 双构建、搬移、或 PIC     ← 软件自己想办法

TC375（硬件重映射）：
  bank 交换只改地址映射，App 永远从 0x80000000 执行
  → 硬件搞定，不用 PIC
```

## 九、如果真要上 PIC，要改哪些东西？

> 以 TASKING 工具链为例（AURIX 车规主力编译器）。前提：**只用在不走硬件 SWAP、单镜像多地址运行场景**；AUTOSAR + 闭源 MCAL/OS 库场景基本走不通，建议双构建。

### 1. 编译/链接选项

```bash
ctc --cpu=tc37x --pic=A12 test.c          # 编译：EABI 完整 PIC/PID 模式
ltc --pic=A12 -L <install>\ctc\lib\pic\tc1.6.2 -deabi_pic.lsl ...  # 链接
# 简化模式（RAM/ROM 数据各 ≤64KB）：
ctc --pic=A0/A1 test.c
```

**连带影响**：`--immediate-in-code` 自动开启；`--profile`/`--runtime` 不兼容被禁用；代码压缩禁用；只能用 `\lib\pic\` 下的 PIC 运行库（libc 只有 `picinit.o`）。

### 2. 链接脚本 `.lsl`

- `#include "eabi_pic.lsl"`（或 `pic.lsl`）；
- GOT/描述符表放进 RAM（或 LMU），输出基址符号（如 `__PIC_BASE`）；
- 内存布局适配 SOTA 双 bank（App 要能装进任一 bank）。

### 3. 启动代码（最关键，最容易错）

```c
void __START(void)
{
    __mtcr(CPU_A12, &__PIC_BASE);   /* ① A12 = GOT 描述符表基址（PIC 必需） */
    /* ② 设置 SP / CSA */
    /* ③ copy table（.data: Flash→RAM）+ clear table（.bss 清零） */
    /* ④ 初始化 BIV/BTV */
    /* ⑤ core0_main() */
}
```

TASKING 提供 `picinit.o` 处理 A12 初始化；若自定义初始化（`--user-provided-initialization-code`），**A12 没设对，第一条全局变量访问就 trap**。

### 4. 中断/陷阱向量表（BIV/BTV）

TriCore 的 BIV 寄存器类似 ARM 的 VTOR，PIC 模式下必须**在 App 启动时动态设置**（不能链接写死）：

```c
__mtcr(CPU_BIV, (uint32_t)&__inttab);
```

### 5. Bootloader 跳转协议

| 交接项 | 非 PIC | PIC（EABI） |
|---|---|---|
| SP / CSA | ✅ | ✅ |
| A12（GOT 基址） | ❌ | ✅ 必须设 |
| A0/A1/A8/A9（简化模式基址） | ❌ | ✅（如用简化模式） |
| BIV/BTV | App 自己设 | App 自己设 |

现实中"跳转 App 后复位/跑飞"的 bug 多出在这里：A12 没设对立刻 trap。

### 6. 运行库与第三方库（AUTOSAR 场景痛点）

- 所有预编译库必须同为 PIC 版；
- 第三方闭源库（Vector/EB 的 MCAL、BSW、AUTOSAR OS）没出 PIC 版 → 用 `abs_addr`/`if_jump_tab` 属性通过固定绝对地址回调"静态软件"，或让供应商出 PIC 版；
- iLLD 是源码，可全量重编，工作量可控。

### 7. 应用代码约束

| 禁止事项 | 说明 |
|---|---|
| 内联汇编写死绝对地址 | 编译期不查，运行期必炸 |
| 常量指针指向绝对地址 | 需封装成跳板函数 |
| 函数指针直接存绝对地址 | 需通过 GOT/符号解析 |
| `__near`/`__a0` 限定符混用 | EABI 模式下要重审 |

### 8. 改动清单速查表

| # | 改动项 | 工作量 | 备注 |
|---|---|---|---|
| 1 | 编译/链接加 `--pic=A12` | 小 | 全工程统一，配 `eabi_pic.lsl` |
| 2 | 换 PIC 运行库 | 小 | 供应商库是拦路虎 |
| 3 | `.lsl` 加 GOT/描述符表布局 | 中 | RAM 里给 GOT 留空间 |
| 4 | 启动代码设 A12 + 重定位表 | 中 | **最容易错** |
| 5 | BIV/BTV 动态设置 | 小 | App 启动时设 |
| 6 | Bootloader 跳转协议 | 中 | 交接 SP/CSA/A12/BIV |
| 7 | 代码约束审查 | 中 | 绝对地址清零 |
| 8 | 第三方库 PIC 化 | **大** | MCAL/OS 闭源库基本卡死 |
| 9 | 安全/CRC/认证复核 | 大 | ISO 26262 影响评估 |
| 10 | 调试工具 SWAP 支持 | 小 | TRACE32/winIDEA 配置 |

### 最终选型建议

| 你的情况 | 建议 |
|---|---|
| 用官方 SOTA（UCB SWAP） | **不要 PIC**，配 UCB 就行 |
| 自研 Bootloader + 全源码（裸机/RT-Thread） | PIC 可行，按清单 1~8 推进 |
| 自研 Bootloader + AUTOSAR 闭源 MCAL/OS | **建议双构建**，比 PIC 靠谱得多 |

## 十、总结：一句话记忆

> **LMA 决定"烧到哪"，VMA 决定"跑到哪"；.text 常常就地执行（XIP，两者相等），.data 必由 Flash 搬运到 RAM（两者不等），.bss 只清不搬。**
> **链接脚本定坐标，启动代码做搬运，.map 文件来验证。**
> **PIC 是"地址不确定"场景的救星：共享库、启动早期借宿、双 bank OTA；但 TC375 官方 SOTA 靠硬件 SWAP 保证执行地址不变，天然不需要 PIC——能用硬件解决的事，别让软件背锅。**
