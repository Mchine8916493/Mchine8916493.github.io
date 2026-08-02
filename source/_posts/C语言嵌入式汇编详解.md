---
title: C语言嵌入式汇编详解
date: 2026-08-02 16:35:16
tags:
  - C语言
  - 嵌入式汇编
  - 内联汇编
  - GCC
  - ARM
  - 汇编语言
  - 底层原理
---

在 C 语言中直接嵌入汇编指令，叫**嵌入式汇编**（Inline Assembly）或内联汇编。它让你在 C 代码中直接编写汇编指令，既能获得汇编级的控制力，又能保留 C 的便利性。这在嵌入式驱动开发、操作系统底层、性能关键代码中极为常用。本文以 **GCC 编译器**为主讲解（同时覆盖 x86 与 ARM），并补充 Keil/IAR 等其他常见编译器写法。

# 一、为什么需要嵌入式汇编

## 1.1 使用场景

| 场景 | 原因 |
|---|---|
| 访问特殊寄存器 | 如 CP15、状态寄存器等 C 无法直接操作 |
| 原子操作 | 如 `ldrex/strex`、`cmpxchg`、`fetch_and_add` |
| 特权指令 | 开/关中断、进入休眠、切换模式 |
| 性能热点 | 编译器生成不了最优代码的极小代码段 |
| 内建指令替代 | 部分场景可用 `__builtin_*`，但仍有兜底需求 |
| 启动代码 | 设置栈指针、跳转、初始化 MMU/Cache |

## 1.2 嵌入式汇编 vs 独立汇编文件

| | 嵌入式汇编（inline） | 独立 `.s` 汇编文件 |
|---|---|---|
| 与 C 变量交互 | 直接绑定，无需手动传参 | 需按 ABI 手工传参 |
| 优化 | 编译器可整体优化 | 固定不变 |
| 可移植性 | 受编译器方言影响 | 纯汇编 |
| 适用 | 短小、与 C 数据交互多 | 长段、固定不变的例程 |

---

# 二、GCC 内联汇编基础语法

## 2.1 基本语法格式

```c
__asm__ [volatile] (
    "汇编指令序列"
    : 输出操作数
    : 输入操作数
    : 破坏列表(clobber)
);
```

四个部分用冒号分隔：

```c
__asm__ (
    "addl %1, %0"        // 汇编指令，%0 %1 是操作数占位符
    : "=r" (result)      // 输出操作数
    : "r" (a), "r" (b)   // 输入操作数
    : "cc"               // 破坏列表
);
```

## 2.2 操作数约束（Constraints）

约束告诉编译器操作数放在哪里（寄存器、内存、立即数）。

### 常用寄存器约束

| 约束 | 含义 |
|---|---|
| `r` | 任意通用寄存器 |
| `a` | x86 的 EAX/RAX |
| `b` | EBX/RBX |
| `c` | ECX/RCX |
| `d` | EDX/RDX |
| `D` | EDI/RDI |
| `S` | ESI/RSI |
| `q` | x86 的 a/b/c/d 之一 |
| `I` | 0~31 的立即数（ARM 移位立即数） |
| `J` | -4095~4095（ARM） |
| `M` | 0~0xFFFFFFFF（ARM） |

### 修饰符（放在约束前）

| 修饰符 | 含义 |
|---|---|
| `=` | 输出操作数（只写） |
| `+` | 读写操作数 |
| `&` | earlyclobber：该输出在指令全部输入用完前就可能被写，须分配独立寄存器 |

## 2.3 最简单的示例

```c
#include <stdio.h>

int main(void) {
    int a = 10, b = 20, sum;

    __asm__ (
        "addl %1, %0\n\t"
        : "=r"(sum)
        : "r"(a), "0"(b)
        : "cc"
    );
    printf("%d\n", sum);   // 输出 30

    return 0;
}
```

> `%0` 对应第一个操作数（输出 sum），`%1` 对应输入 a，`%2` 对应输入 b。`"0"` 表示与第 0 个操作数共用寄存器（即输入 b 复用 sum 的寄存器）。

---

# 三、内联汇编各部分的深入

## 3.1 输出操作数（Output）

```c
int out;
__asm__ ("movl %1, %0"
         : "=r"(out)     // 输出必须带 "=" 或 "+"
         : "r"(in));
```

- 输出约束必须用 `=`（只写）或 `+`（读写）
- 多个输出用逗号分隔，分别对应 `%0`、`%1`…
- 若汇编指令修改了某个输入，但没有对应输出，编译器不知道——必须用 `+` 或加入破坏列表

## 3.2 输入操作数（Input）

```c
int add(int a, int b) {
    int result;
    __asm__ (
        "addl %2, %1\n\t"
        "movl %1, %0"
        : "=r"(result)
        : "r"(a), "r"(b)
    );
    return result;
}
```

## 3.3 破坏列表（Clobber）

告诉编译器"我的汇编代码会破坏这些资源，请自行保存/恢复"：

```c
__asm__ (
    "movl %0, %%eax\n\t"   // 在 AT&T 语法中寄存器前要加 %%
    : : "r"(val)
    : "eax"                // 告诉编译器 eax 被改写了
);
```

常用破坏项：
- 寄存器名：`"eax"`, `"r1"`, `"r2"`…
- `"cc"`：修改了条件标志（状态寄存器）
- `"memory"`：修改了内存（编译器需保守，禁止跨内联汇编重新排布内存访问）

```c
// 内存屏障示例
#define barrier() __asm__ volatile("" : : : "memory")
```

## 3.4 volatile 关键字

```c
__asm__ volatile ("..." : : : "memory");
```

- `volatile` 告诉编译器：**不要移动、不要删除、不要优化**这段汇编
- 对"有副作用"的指令（写寄存器、I/O、开关中断）必须加
- 纯计算且结果只由输出承载的汇编，不加 volatile 反而能让编译器调度优化

---

# 四、AT&T 与 Intel 语法切换

## 4.1 GCC 默认用 AT&T 语法（x86）

| 差异 | AT&T（GCC 默认） | Intel |
|---|---|---|
| 操作数顺序 | `源, 目标` | `目标, 源` |
| 寄存器前缀 | `%%eax` | `eax` |
| 立即数前缀 | `$10` | `10` |
| 内存寻址 | `(%ebx)` | `[ebx]` |
| 后缀宽度 | `movl`（l/w/b/q） | `mov`（按操作数推断） |

## 4.2 开启 Intel 语法

```c
__asm__ (".intel_syntax noprefix\n\t"
         "mov eax, %0\n\t"
         "add eax, 1\n\t"
         ".att_syntax prefix"
         : "=r"(result)
         : "r"(input));
```

> 结尾记得切回 AT&T，避免污染后续代码。

## 4.3 ARM 内联汇编

ARM 汇编本身没有 AT&T/Intel 之分，操作数顺序为 `目标, 源1, 源2`：

```c
int add_arm(int a, int b) {
    int result;
    __asm__ (
        "add %0, %1, %2\n\t"
        : "=r"(result)
        : "r"(a), "r"(b)
    );
    return result;
}
```

---

# 五、内联汇编的常见用法示例

## 5.1 开关中断

```c
// ARM Cortex-M：开/关中断
static inline void irq_enable(void) {
    __asm__ volatile("cpsie i" : : : "memory");
}

static inline void irq_disable(void) {
    __asm__ volatile("cpsid i" : : : "memory");
}

// x86：关中断
static inline void cli(void) {
    __asm__ volatile("cli" : : : "memory");
}
static inline void sti(void) {
    __asm__ volatile("sti" : : : "memory");
}
```

## 5.2 原子操作（x86 lock 前缀）

```c
static inline int atomic_inc(int *ptr) {
    int tmp;
    __asm__ volatile (
        "lock incl %0"
        : "+m"(*ptr)
        : : "memory", "cc"
    );
    return tmp;  // 简化示例
}
```

## 5.3 读取 CPU 时间戳（x86）

```c
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    __asm__ volatile (
        "rdtsc"
        : "=a"(lo), "=d"(hi)
        : : "memory"
    );
    return ((uint64_t)hi << 32) | lo;
}
```

## 5.4 ARM 独占访问（ldrex/strex，自旋锁核心）

```c
// 原子加：ARM 上实现 *ptr += val
int atomic_add(int *ptr, int val) {
    int old, result;
    __asm__ volatile (
        "1:\n\t"
        "ldrex %0, [%2]\n\t"      // 独占读
        "add %1, %0, %3\n\t"      // 计算新值
        "strex %0, %1, [%2]\n\t"  // 独占写，返回 0=成功
        "teq %0, #0\n\t"          // 测试是否成功
        "bne 1b"                  // 失败则重试
        : "=&r"(result), "=&r"(old)
        : "r"(ptr), "r"(val)
        : "cc", "memory"
    );
    return old;
}
```

## 5.5 触发软中断（系统调用）

```c
// ARM 32 位 Linux 系统调用：sys_write
static inline long sys_write(int fd, const char *buf, size_t count) {
    register long r7 __asm__("r7") = 4;       // sys_write 编号
    register long r0 __asm__("r0") = fd;
    register const char *r1 __asm__("r1") = buf;
    register size_t r2 __asm__("r2") = count;
    __asm__ volatile ("svc #0"
                      : "+r"(r0)
                      : "r"(r1), "r"(r2), "r"(r7)
                      : "memory");
    return r0;
}
```

## 5.6 读特殊寄存器（CP15，ARM）

```c
static inline uint32_t read_cp15_sctlr(void) {
    uint32_t val;
    __asm__ volatile (
        "mrc p15, 0, %0, c1, c0, 0"   // 读系统控制寄存器
        : "=r"(val)
    );
    return val;
}
```

## 5.7 内存屏障

```c
#define dmb()   __asm__ volatile("dmb"  : : : "memory")
#define dsb()   __asm__ volatile("dsb"  : : : "memory")
#define isb()   __asm__ volatile("isb"  : : : "memory")
```

---

# 六、Linux 内核中的内联汇编实例

## 6.1 读 / 写寄存器

```c
// Linux 内核 arch/arm 通用寄存器访问
#define readl(addr) ({ \
    u32 __v; \
    __asm__ volatile("ldr %0, %1" : "=r"(__v) : "m"(*(volatile u32 *)(addr))); \
    __v; \
})
```

## 6.2 自旋锁（Linux 经典实现片段）

```c
static inline void arch_spin_lock(raw_spinlock_t *lock) {
    unsigned long tmp;
    __asm__ __volatile__(
        "1: ldrex %0, [%1]\n"
        "   teq %0, #0\n"
        "   strexeq %0, %2, [%1]\n"
        "   teqeq %0, #0\n"
        "   bne 1b"
        : "=&r" (tmp)
        : "r" (&lock->raw_lock), "r" (1)
        : "cc", "memory");
}
```

> 内联汇编大量用于 Linux 内核对锁、屏障、系统调用的封装，是嵌入式汇编的最佳教材。

---

# 七、其他编译器的内联汇编

## 7.1 MSVC（Windows，Intel 语法）

```c
// x86 32 位可用 __asm 块
__asm {
    mov eax, a
    add eax, b
    mov result, eax
}

// x64 不支持 __asm，需用内建函数或独立汇编
```

## 7.2 Keil MDK / ARM Compiler

```c
// Keil 使用 __asm 关键字（ARM Compiler 6 支持内联）
__asm int add_func(int a, int b) {
    ADD r0, r0, r1
    BX  lr
}
```

## 7.3 IAR EWARM

```c
// IAR 用 __asm 关键字，支持操作数绑定
int add(int a, int b) {
    int result;
    __asm("add %0, %1, %2" : "=r"(result) : "r"(a), "r"(b));
    return result;
}
```

## 7.4 Clang

与 GCC 语法兼容，可直接用 `__asm__`。

> 不同编译器方言差异大，**跨编译器代码尽量用 `__builtin_*` 内建函数**替代，提高可移植性。

---

# 八、用 __builtin 代替内联汇编（推荐优先）

很多场景 GCC 提供了内建函数，比手写汇编更安全、更可移植：

```c
__sync_fetch_and_add(&x, 1);       // 原子自增
__sync_lock_test_and_set(&x, 1);   // 原子交换
__atomic_add_fetch(&x, 1, __ATOMIC_SEQ_CST);  // C11 原子操作
__builtin_popcount(x);             // 位计数
__builtin_clz(x);                  // 前导零
__builtin_ctz(x);                  // 尾随零
__builtin_expect(x, 1);            // 分支预测提示
__builtin_unreachable();           // 告诉编译器不会到达
__builtin_return_address(0);       // 获取返回地址
```

**建议优先级**：
1. 优先标准库 / C11 原子
2. 其次 `__builtin_*`
3. 只有前两者覆盖不了，才手写内联汇编

---

# 九、内联汇编常见陷阱

## 9.1 忘记输出约束的 "="

```c
__asm__("movl %1, %0" : "r"(out) : "r"(in));  // ❌ 输出少了 "="
__asm__("movl %1, %0" : "=r"(out) : "r"(in)); // ✅
```

## 9.2 修改了操作数却没声明

```c
// 输入 a 被指令修改，但只声明为输入 → 编译器不感知，产生错误代码
__asm__("incl %1" : "=r"(r) : "r"(a));  // ⚠️ a 被改但未声明
__asm__("incl %1" : "=r"(r), "+r"(a));  // ✅ 用 + 声明读写
```

## 9.3 earlyclobber 漏写 &

```c
// 输入和输出可能分配到同一寄存器，导致计算结果被破坏
__asm__("addl %1, %0"
        : "=&r"(result)     // & 强制 result 用独立寄存器
        : "r"(a), "r"(b));
```

## 9.4 忘记 clobber

```c
__asm__("movl %0, %%ecx" : : "r"(x));  // ❌ 改了 ecx 没声明
__asm__("movl %0, %%ecx" : : "r"(x) : "ecx"); // ✅
```

## 9.5 忘记 volatile 导致指令被优化掉

```c
// 没有副作用也没输出的纯汇编可能被优化删除
__asm__ volatile("nop");   // ✅ 加 volatile
```

## 9.6 汇编指令内部操作数顺序搞反

x86 AT&T：`"movl %1, %0"` 是 `%1 → %0`（源在前）。

## 9.7 寄存器被 C 编译器用于其他用途

内联汇编里**不要直接写死寄存器名**（如 `eax`、`r0`），除非声明在破坏列表中，否则可能覆盖编译器正在使用的值。

---

# 十、调试内联汇编

## 10.1 查看编译器生成的汇编

```bash
gcc -S test.c -o test.s        # 看完整汇编
gcc -O2 -S test.c -o test.s    # 优化后
objdump -d test                # 反汇编
```

## 10.2 验证生成的指令序列

```bash
gcc -O2 -S -fverbose-asm test.c -o test.s   # 带注释的汇编
```

## 10.3 常见检查

```c
- 用 printf 打印结果验证
- 用 -fsanitize 检测内存问题
- 用 GDB stepi 单步进入汇编指令
- 反汇编确认 clobber 后寄存器确实被保存
```

---

# 十一、实战：完整示例

## 11.1 用内联汇编实现 memcpy（ARM 优化示例）

```c
void asm_memcpy(void *dst, const void *src, size_t n) {
    __asm__ volatile (
        "1:\n\t"
        "ldr r3, [%1], #4\n\t"     // 读取 4 字节，源指针后移
        "str r3, [%0], #4\n\t"     // 写入 4 字节，目的指针后移
        "subs %2, %2, #4\n\t"      // n -= 4
        "bne 1b"                   // 不为 0 继续
        : "+r"(dst), "+r"(src), "+r"(n)
        : : "memory", "r3"
    );
}
```

## 11.2 内联汇编 + C 混合完成位翻转

```c
// 反转一个字节的 8 位
uint8_t reverse_bits(uint8_t b) {
    uint8_t out;
    __asm__ (
        "mov %0, %1\n\t"       // out = b
        "rbit %0, %0\n\t"      // ARM 指令：反转整个寄存器的位
        "lsr %0, %0, #24\n\t"  // 移到低 8 位
        : "=r"(out)
        : "r"(b)
    );
    return out;
}
```

---

# 十二、跨平台封装的建议

为了可移植性，建议用宏封装，各平台分别实现：

```c
#if defined(__ARM_ARCH_7A__)
#define memory_barrier() __asm__ volatile("dmb" : : : "memory")
#elif defined(__x86_64__) || defined(__i386__)
#define memory_barrier() __asm__ volatile("mfence" : : : "memory")
#elif defined(__riscv)
#define memory_barrier() __asm__ volatile("fence rw, rw" : : : "memory")
#else
#define memory_barrier() __asm__ volatile("" : : : "memory")
#endif
```

---

# 总结

C 语言嵌入式汇编的知识图谱：

> **基本语法（四段式：指令/输出/输入/破坏）→ 约束与修饰符（r、=、+、&）→ volatile 语义 → 平台差异（AT&T vs Intel、x86 vs ARM）→ 实用场景（开关中断/原子操作/系统调用/寄存器访问）→ 陷阱规避（clobber、earlyclobber、优化删除）**

**最佳实践总结**：

1. 优先用 `__builtin_*` 和 C11 原子操作，减少裸汇编
2. 汇编必须加 `volatile`（有副作用时）
3. 每个被修改的寄存器、内存、标志都要写进破坏列表
4. 输入输出用约束绑定，避免写死寄存器名
5. 使用 `&` 防止 earlyclobber 冲突
6. 写完务必反汇编核对生成的代码

> **一句话记住**：内联汇编是 C 和汇编之间的"接口协议"——通过约束和破坏列表把寄存器所有权交给编译器管理，你只负责描述"做什么"，编译器负责"别踩到我的寄存器"。
