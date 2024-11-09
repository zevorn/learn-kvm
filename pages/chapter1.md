---
layout: center
---
# 第 1 章  CPU 虚拟化

1. 介绍 CPU 虚拟化的基本概念，探讨 x86 架构在虚拟化时面临的障碍以及采取的措施

2. 重点讨论虚拟机 CPU 如何在 Host 模式和 Guest 模式之间切换

3. 讲解虚拟 CPU 在 Guest 模式下运行时由于运行敏感指令而触发虚拟机退出的典型情况

4. 通过一个具体的 KVM 用户空间的实例，直观地展示 CPU 虚拟化的概念

---
layout: cover
---

# 1.1 x86 架构 CPU 虚拟化

### 虚拟化的三个条件：

- 等价性：即 VMM 需要在宿主机上为虚拟机模拟出一个本质上与物理机一致的环境

- 高效性：即虚拟机执行时，相比在物理机上运行没有明显性能损耗

- 资源控制：即 VMM 可以完全控制系统资源。


---
layout: default
---

## 1.1.1 陷入和模拟模型

处理器分为两种运行模式：系统模式和用户模式。相应地，CPU的指令也分为特权指令和非特权指令。

在陷入和模拟模型下，虚拟机的完整模型（用户程序 + 内核），全部运行在用户模式，

这种方式称为**特权级压缩**（Ring Compression）。示意图如下：

```
       User Mode                          System Mode      
+-------------------+                  +------------------+
|                   |                  |                  |
|     Guest App     |                  |                  |
|                   |                  |       VMM        |
+-------------------+                  |                  |
|                   |                  |                  |
|      Guset OS     |                  |                  |
|      +------------+                  +------------+     |
|      |            |                  |            |     |
|      |  priv insn |<---------------->|  emulator  |     |
|      |            |                  |            |     |
+------+------------+                  +------------+-----+
```

---
layout: default
---

## 1.1.2 x86 架构虚拟化的障碍

- 敏感指令：修改系统资源的，或者在不同模式下行为有不同表现的

- 特权指令：指那些只能在最高特权级别（即环0，Ring 0）上执行的指令

> 注意：x86 架构不是所有敏感指令，都是特权指令

```asm
section .text
global _start

_start:
    pushf              ; 保存当前的标志寄存器内容到栈中

    mov bx, sp         ; 获取栈顶地址

    ; 栈中的标志寄存器值现在位于 bx 中指向的位置
    ; IF 位在标志寄存器的第 9 位 (从 0 开始计数)，因此要清除 IF，我们需要确保这一位置为 0
    ; 使用 and 操作来清除 IF 位
    mov ax, [bx]       ; 把栈顶的数据读入 AX 寄存器
    and ax, 0xFDFFh    ; 清除 IF 位 (置位为 0)

    mov [bx], ax       ; 将修改后的标志寄存器值写回到栈中

    popf               ; 从栈中恢复标志寄存器
```

---
layout: default
---

### 半虚拟化解决方案

即修改 Guest 的代码，但这不符合虚拟化的透明标准。

### 二进制翻译方案

- 静态翻译：运行前扫描整个文件，对敏感指令进行翻译；

- 动态翻译：运行时以代码块为单元，动态地修改二进制代码。

> 动态翻译在很多 VMM 中得到应用，而且优化的效果非常不错。

---

## 1.1.3 VMX
<br>
虽然大家从软件层面采用了多种方案来解决 x86 架构在虚拟化时遇到的问题，

但是这些解决方案除了引入了额外的开销外，还给 VMM 的实现带来了巨大的复杂性。

<br>
于是，Intel 尝试从硬件层面解决这个问题。

Intel 并没有将那些非特权的敏感指令修改为特权指令，因为并不是所有的特权指令都需要拦截处理。

```
         Disabled EPT                                     Enabled EPT                           
+------------+      +--------------+            +------------+      +--------------+           
| Process1   |  |   | VMM          |            | Process1   |  |   | VMM          |           
|            |  |   |              |            |            |  |   |              |           
+-----+------+  |   | +-----------+|            +-----+------+  |   |              |           
 +----v-----+   |   | |  Update   ||             +----v-----+   |   |              |           
 |switch CR3|---+---+>|Shadow page||             |switch CR3|   |   |              |           
 |Other ops |<--+---+-+   table   ||             |Other ops |   |   |              |           
 +----+-----+   |   | +-----------+|             +----+-----+   |   |              |           
+-----v------+  |   |              |            +-----v------+  |   |              |           
|            |      |              |            |            |      |              |           
| Process2   |      |              |            | Process2   |      |              |           
+------------+      +--------------+            +------------+      +--------------+           
```

---

Intel 开发了 VT 技术以支持虚拟化，为 CPU 增加了 Virtual-Machine Extensions，简称 VMX。

两种运行模式：

- VMX root operation（host 模式）
- VMX non-root operation（guest 模式）

每个运行模式都支持 ring 0 ~ ring 3。

```
       VMX Root Mode                 VMX non-Root Mode 
   +------------------+            +------------------+ 
   |ring3             |            |ring3             | 
   |  +-------------+ |            | +--------------+ | 
   |  |  Host App   | |  VM Entry  | |   Guest App  | | 
   |  +-------------+ | ---------> | +--------------+ | 
   +------------------+            +------------------+ 
   |  +-------------+ |            | +--------------+ | 
   |  | Host Kernel | |  VM Exit   | | Guest Kernel | | 
   |  +-------------+ | <--------- | +--------------+ | 
   |ring0             |            |ring0             | 
   +------------------+            +------------------+ 
```

---

相比将 Guest 的内核也运行在用户模式（ring 1 ~ ring 3）的方式，

支持 VMX 的 CPU 有以下 3 点 不同：

- Guest 系统调用不再陷入 Host 内核空间

- Guest 接收的外部中断，由 Host 内核处理

- 只有 Guest 遇到敏感指令，才会发生 VM exit

---

在虚拟化场景下，同一个物理 CPU “一人分饰多角”。

为了方便调度和保存上下文，VMX设计了一个保存上下文的数据结构：VMCS

```
                                                                                                   
+-------------------+                                                      +-------------------+   
|                   |                                                      |                   |   
| +---------------+ |                                                      | +---------------+ |   
| | Guest App     | |              +------------------------+              | | Guest App     | |   
| |               | |  +--------+  |                        |  +--------+  | |               | |   
| +---------------+ +--+--------+->|        Host App        +--+--------+->| +---------------+ |   
| ----------------- |  | VMCS-1 |  |  --------------------  |  | VMCS-2 |  | ----------------- |   
|                   |<-+--------+--+       Host Kernel      |<-+--------+--+                   |   
| +---------------+ |  +--------+  |                        |  +--------+  | +---------------+ |   
| | Guest Kernel  | |              +------------------------+              | | Guest Kernel  | |   
| |               | |                                                      | |               | |   
| +---------------+ |                                                      | +---------------+ |   
+-------------------+                                                      +-------------------+   
                                                                                                   
```

---

VMCS 中主要保存着两大类数据：

<br>

#### Host 的状态和 Guest 的状态：

1. Guest-state area，保存虚拟机状态的区域

2. Host-state area，保存宿主机状态的区域

<br>

#### 控制 Guest 运行时的行为：

1. VM-exit information fields

2. VM-execution control fields

---

在创建 VCPU 时，KVM 模块将为每个 VCPU 申请一个 VMCS ，

每次 CPU 准备切入 Guest 模式时，将设置其 VMCS 指针指向即将切入的 Guest 对应的 VMCS 实例：

```c
commit 6aa8b732ca01c3d7a54e93f4d701b8aabbe60fb7 
[PATCH] kvm: userspace interface 
linux.git/drivers/kvm/vmx.c 
static struct kvm_vcpu *vmx_vcpu_load(struct kvm_vcpu *vcpu) // 加载一个 VCPU 的状态
{
    u64 phys_addr = __pa(vcpu->vmcs); // 获取 VCPU 控制结构（VMCS）的物理地址
    int cpu;
    cpu = get_cpu(); // 获取当前 CPU 的编号
    …
    // 检查当前 CPU 的 VMCS 是否与要加载的 VMCS 不同。如果不同，执行大括号内的代码
    if (per_cpu(current_vmcs, cpu) != vcpu->vmcs) {
        …
        per_cpu(current_vmcs, cpu) = vcpu->vmcs; // 将当前 CPU 的 VMCS 设置为要加载的 VMCS
        // 执行 VMX 的 VMPTRLD 指令，该指令加载 VMCS 的物理地址
        asm volatile (ASM_VMX_VMPTRLD_RAX "; setna %0"
                  : "=g"(error) : "a"(&phys_addr), "m"(phys_addr)
                  : "cc");
        …
    }
    …
}
```

---

## 1.1.4 VCPU 生命周期

对于每个虚拟处理器（VCPU），VMM 使用一个线程来代表 VCPU 这个实体。

在 Guest 运转过程中，每个 VCPU 基本都在下方所示的状态中不断地转换：

```
                    VMX Root Mode                         VMX non-Root      
    |                                              |         Mode          |  PS:
    |      userspace       |      kernelspace      |                       |  1. 如果是首次运行 Guest, 则使用 VMLaunch ，
    |   +--------------+   |  1)                   |                       |     否则使用 VMResume ；
    |   |iotcl(KVM_RUN)+---+-----------+           |                       |
    |   +------^-------+   |    +------v------+    |   2)                  |
    |          |           |    |  switch to  +----+------------+          |
    |          |           |    |    guest    |    |            |          |
    |          |           |    +------^------+    |            |          |
    |          |           |           |           |    +-------v------+   |
    |    loop  |           |           |           |    | native guest |   |
    |          |           |       4)  |           |    |  execution   |   |
    |          |           |           | Y         |    +-------+------+   |
    |          |           |    +------+------+    |            |          |
    |          |           |    |   emulate   <----+------------+          |
    |          |           |    |(kernelspace)|    |     3)                |
    |   +------+------+    |    +------+------+    |                       |
    |   |   emulate   <----+-----------+ N         |                       |
    |   | (userspace) |    |       5)              |                       |
    |   +-------------+    |                       |                       |
```

---

下面是 KVM 切入、切出 Guest 的代码：

```c
commit 6aa8b732ca01c3d7a54e93f4d701b8aabbe60fb7 
[PATCH] kvm: userspace interface 
linux.git/drivers/kvm/vmx.c 
static int vmx_vcpu_run(struct kvm_vcpu *vcpu, …) 
{ 
    ...
again:
    … 
        /* Enter guest mode */ 
        "jne launched \n\t" 
        ASM_VMX_VMLAUNCH "\n\t" 
        "jmp kvm_vmx_return \n\t" 
        "launched: " ASM_VMX_VMRESUME "\n\t" 
        ".globl kvm_vmx_return \n\t" 
        "kvm_vmx_return: " 
        /* Save guest registers, load host registers, keep flags */ 
    … 
        if (kvm_handle_exit(kvm_run, vcpu)) { 
            …
          goto again; 
        }
    ...
    return 0; 
}
```

---

如果内核空间成功处理了虚拟机的退出，则函数 kvm_handle_exit 返回 1 ，在上述代码中即直接跳转到标签 again 处，然后程序流程会再次切入 Guest 。

如果函数 kvm_handle_exit 返回0，则函数 vmx_vcpu_run 结束执行，CPU 从内核空间返回到用户空间。

以kvmtool 为例，其相关代码片段如下：

```c
commit 8d20223edc81c6b199842b36fcd5b0aa1b8d3456 
Dump KVM_EXIT_IO details 
kvmtool.git/kvm.c 
int main(int argc, char *argv[]) 
{ 
    …
    for (;;) {
        kvm__run(kvm);
        switch (kvm->kvm_run->exit_reason) {
        case KVM_EXIT_IO:
        …
    }
    …
} 
```
---
layout: cover
---

# 1.2 虚拟机切入和退出

<br>
在这一节，我们讨论内核中的 KVM 模块如何切入虚拟机，

以及围绕着虚拟机切入和退出进行的上下文保存。

---

## 1.2.1 GCC 内联汇编

KVM 模块中切入 Guest 模式的代码使用 GCC 的内联汇编编写，

为了理解这段代码，我们需要简要地介绍一下这段内联汇编涉及的语法，其基本语法模板如下：

```c
    asm volatile (assembler template
        : output operands             /* optional */
        : input operands              /* optional */
        : list of clobbered registers /* optional */
    );
```

<br>

- 关键字 asm 和 volatile

`asm` 为 GCC 关键字，表示接下来要嵌入汇编代码，

如果 asm 与程序中其他命名冲突，可以使用`__asm__`。

`volatile` 为可选关键字，表示不需要 GCC 对下面的汇编代码做任何优化，

类似的，GCC 也支持 `__volatile__`。

---

- 汇编指令（assembler template）

这部分即要嵌入的汇编指令，由于是在 C 语言中内联汇编代码，因此须用双引号将命令括起来。

如果内嵌多行汇编指令:

1. 每行指令使用双引号括起来，以后缀 `;` 结尾
2. 每行指令使用双引号括起来，以后缀 `\n\t` 结尾

<br>

> PS: 由于 GCC 将每条指令以字符串的形式传递给汇编器 AS ，所以要使用分隔符来分隔每一条指令。

示例代码如下：

```c
__asm__ ("movl %eax, %ebx \n\t"
         "movl $56, %esi \n\t");
``` 

or

```c
__asm__ ("movl %ecx, $label(%edx,%ebx,$4);"
         "movb %ah, (%ebx);");
```

---

当使用扩展模式，即包含 output、input 和 clobber list 部分时，

汇编指令中需要使用两个 `%` 来引用寄存器，比如 `%%rax`；

使用一个 `%` 来引用输入、输出操作数，比如 `%1`，以便帮助 GCC 区分寄存器和由 C 语言提供的操作数。

```c
int val = 100, sum = 0;

asm (
    "movl %1, %%rax;"
    "movl %c[addend], %%rbx;"
    "addl %%rbx, %%rax;"
    "movl %%rax, %0;"
    : "="(sum)
    : (c)(val), [addend]"i"(400)
    : "rbx"
);
```

- **output**: 输出操作数的约束部分必须以`=`或者`+`作为前缀，`=`表示只写，`+`表示读写
- **intput**: 输入操作数来自 C 代码中的变量或者表达式，没有约束前缀
- **clobber list**： 标记可能会影响的寄存器或者状态

---

## 1.2.2 虚拟机切入和退出及相关的上下文保存

了解了内联汇编的语法后，接下来我们开始探讨虚拟机切入和退出部分的内联汇编指令：

```c
commit 1c696d0e1b7c10e1e8b34cb6c797329e3c33f262
KVM: VMX: Simplify saving guest rcx in vmx_vcpu_run
linux.git/arch/x86/kvm/vmx.c
static void vmx_vcpu_run(struct kvm_vcpu *vcpu)
{
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    ...
    asm(
    /* Store host registers */
    "push %%"R"dx; push %%"R"bp;" // 将宿主机的通用寄存器保存到栈中
    "push %%"R"cx \n\t"
    ...
    :: "c"(vmx), "d"((unsigned long) HOST_RSP),
    ...
    : "cc", "memory"
    , R"ax", R"bx", R"di", R"si" // 所有可能被内联汇编影响的寄存器 
#ifdef CONFIG_X86_64
    , "r8", "r9", "r10", "r11", "r12", "r13","r14", "r15"
#endif
...
```

---

在 Guest 退出时，CPU 会自动将 VMCS 中 Host 的 rsp/esp 寄存器恢复到物理 CPU 的 rsp/esp 寄存器中，
所以此时可以访问 VCPU 线程在 Host 态下的栈。

```c
        ...
        asm(
        /* Save guest registers, load host registers,keep …*/
        "xchg %0, (%%"R"sp) \n\t"
        "mov %%"R"ax, %c[rax](%0) \n\t"
        "mov %%"R"bx, %c[rbx](%0) \n\t"
        "pop"Q" %c[rcx](%0) \n\t" // 从栈顶弹出 Guest 的 rcx/ecx 的值到保存 Guest 状态的内存中 rcx/ecx 相应的位置
        "mov %%"R"dx, %c[rdx](%0) \n\t"
        ...
        "mov %%cr2, %%"R"ax \n\t"
        "mov %%"R"ax, %c[cr2](%0) \n\t"
        ...
        :: "c"(vmx), "d"((unsigned long) HOST_RSP),
```
<br>

> PS: 在 Guest 退出的第一时间，除了专用寄存器，这些通用寄存器中保存的都是 Guest 的状态

---

并不是每次 Guest 退出到切入，Host 的栈都会发生变化，因此 Host 的 rsp/esp 也无须每次都更新。

所以 KVM 在 VCPU 中记录了 host_rsp 的值，用来比较 rsp/esp 是否发生了变化，代码如下：

```c
enum vmcs_field { /* VMCS Encodings */
    ...
    HOST_RSP = 0x00006c14,
    ...
};
static void vmx_vcpu_run(struct kvm_vcpu *vcpu)
{
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    ...
    asm(
        ...
        "cmp %%"R"sp, %c[host_rsp](%0) \n\t"
        "je 1f \n\t"
        "mov %%"R"sp, %c[host_rsp](%0) \n\t"
        __ex(ASM_VMX_VMWRITE_RSP_RDX) "\n\t" // 将
        "1: \n\t"
    ...
    :: "c"(vmx), "d"((unsigned long) HOST_RSP),
    [host_rsp]"i"(offsetof(struct vcpu_vmx,host_rsp)),
```

只有 rsp/esp 变化了，才需要更新 VMCS 中Host的 rsp/esp 字段，以减少不必要的写 VMCS 操作。

---

VMX 没有定义 CPU 自动保存 cr2 寄存器，但是事实上，Host 可能更改 cr2 的值，以下面这段代码为例：

```c
commit 1c696d0e1b7c10e1e8b34cb6c797329e3c33f262
KVM: VMX: Simplify saving guest rcx in vmx_vcpu_run
linux.git/arch/x86/kvm/x86.c
void kvm_inject_page_fault(struct kvm_vcpu *vcpu, …)
{
    ++vcpu->stat.pf_guest;
    vcpu->arch.cr2 = fault->address;
    kvm_queue_exception_e(vcpu, PF_VECTOR, fault->error_code);
}
```

<br>

1. 在切入 Guest 前，KVM 检测物理 CPU 的 cr2 寄存器与 VCPU 中保存的 Guest 的 cr2 寄存器是否相同
2. 如果不同，则需要使用 Guest 的 cr2 寄存器更新物理 CPU 的 cr2 寄存器

代码如下：

```c
    /* Reload cr2 if changed */
    "mov %c[cr2](%0), %%"R"ax \n\t"
    "mov %%cr2, %%"R"dx \n\t"
    "cmp %%"R"ax, %%"R"dx \n\t"
    "je 2f \n\t"
    "mov %%"R"ax, %%cr2 \n\t"
```
---
layout: cover
---

# 1.3 陷入和模拟

<br>
从 Host 的角度来说，VM 就是 Host 的一个进程。一个 Host 上的多个 VM 与 Host 共享系统的资源。

因此，当访问系统资源时，就需要退出到 Host 模式，由 Host 作为统一的管理者代为完成资源访问。

<br>
需要陷入 Host 和模拟的情况：

 - Guest 主动陷入 Host：比如虚拟机访问 I/O

 - Guest 被动陷入 Host: 比如遇到特殊指令，或者遇到外部中断


---

## 1.3.1 访问外设

<br>

常用的访问外设方式包括 PIO（programmed I/O）和 MMIO（memory-mapped I/O）。

在这一节，我们重点探讨 MMIO ，然后简单地介绍一下 PIO ，

更多内容将在“设备虚拟化”一章中讨论。

---

### 1. MMIO

MMIO 是 PCI 规范的一部分，I/O 设备被映射到内存地址空间而不是 I/O 空间。

以 MMIO 方式访问外设时不使用专用的访问外设的指令（out、outs、in、ins），是一种隐式的 I/O 访
问。

> PS: 由于这些映射的地址空间是留给外设的，因此 CPU 将产生页面异常，从而触发虚拟机退出，陷入 VMM 中。

以 LAPIC 为例：

```c
linux-1.3.31/arch/i386/kernel/smp.c
void smp_boot_cpus(void)
{
    …
    apic_reg = vremap(0xFEE00000,4096);
    …
}
linux-1.3.31/include/asm-i386/i82489.h
#define APIC_ICR 0x300
linux-1.3.31/include/asm-i386/smp.h
extern __inline void apic_write(unsigned long reg,unsigned long v)
{
    *((unsigned long *)(apic_reg+reg))=v;
}
```

---

此时访问 icr 寄存器就像访问普通内存一样了，写 icr 寄存器的代码如下所示：

```c
linux-1.3.31/arch/i386/kernel/smp.c
void smp_boot_cpus(void)
{
    …
    apic_write(APIC_ICR, cfg); /* Kick the second */
    …
}
```

当 Guest 执行这条指令时：

```c {*}{maxHeight:'220px'}
commit 97222cc8316328965851ed28d23f6b64b4c912d2
KVM: Emulate local APIC in kernel
linux.git/drivers/kvm/vmx.c
static int handle_exception(struct kvm_vcpu *vcpu, …)
{
    …
    if (is_page_fault(intr_info)) {
        …
        r = kvm_mmu_page_fault(vcpu, cr2, error_code);
        …
        if (!r) {
            …
            return 1;
        }
        er = emulate_instruction(vcpu, kvm_run, cr2, error_code);
        …
    }
    …
}
```

---

x86 指令格式

```
+---------------+----------+----------+-------+----------------+-------------+
|               |          |          |       |                |             |
|  insn prefix  |  opcode  |  ModR/M  |  SIB  |  displacement  |  immediate  |
|               |          |          |       |                |             |
+---------------+----------+----------+-------+----------------+-------------+
```

- insn prefix: rep, lock, repne, ...

- opcode: 1-3 bytes

- ModR/M: 1 byte

- displacement: 表示偏移

- immediate: 表示立即数

---

我们以下面的代码片段为例，看一下编译器将 MMIO 访问翻译的汇编指令：

```c
// test.c
char *icr_reg;
void write() {
    *((unsigned long *)icr_reg) = 123;
}
```

我们将上述代码片段编译为汇编指令：`gcc -S test.c`

核心汇编指令如下：

```asm
// test.s
    movq icr_reg(%rip), %rax
    movq $123, (%rax)
```

可见，这段 MMIO 访问被编译器翻译为 mov 指令，源操作数是立即数，

目的操作数 icr_reg（%rip）相当于 icr 寄存器映射到内存地址空间中的内存地址。

---

KVM 中模拟指令的入口函数是 emulate_instruction，其核心部分在函数 x86_emulate_memop 中:

```c {*}{maxHeight:'400px'}
commit 97222cc8316328965851ed28d23f6b64b4c912d2
KVM: Emulate local APIC in kernel
linux.git/drivers/kvm/x86_emulate.c
01 int x86_emulate_memop(struct x86_emulate_ctxt *ctxt, …)
02 {
03     unsigned d;
04     u8 b, sib, twobyte = 0, rex_prefix = 0;
05     …
06     for (i = 0; i < 8; i++) {
07         switch (b = insn_fetch(u8, 1, _eip)) {
08     …
09     d = opcode_table[b];
10     …
11     if (d & ModRM) {
12         modrm = insn_fetch(u8, 1, _eip);
13         modrm_mod |= (modrm & 0xc0) >> 6;
14         …
15     }
16     …
17     switch (d & SrcMask) {
18     …
19     case SrcImm:
20         src.type = OP_IMM;
21         src.ptr = (unsigned long *)_eip;
22         src.bytes = (d & ByteOp) ? 1 : op_bytes;
23         …
24         switch (src.bytes) {
25             case 1:
26             src.val = insn_fetch(s8, 1, _eip);
27             break;
28             …
29     }
30     …
31     switch (d & DstMask) {
32         …
33     case DstMem:
34         dst.type = OP_MEM;
35         dst.ptr = (unsigned long *)cr2;
36         dst.bytes = (d & ByteOp) ? 1 : op_bytes;
37         …
38     }
39     …
40     switch (b) {
41     …
42     case 0x88 ... 0x8b: /* mov */
43     case 0xc6 ... 0xc7: /* mov (sole member of Grp11) */
44         dst.val = src.val;
45         break;
46     …
47     }
48 
49     writeback:
50     if (!no_wb) {
51         switch (dst.type) {
52         …
53         case OP_MEM:
54             …
56                 rc = ops->write_emulated((unsigned long)dst.ptr,
57                              &dst.val, dst.bytes,
58                              ctxt->vcpu);
59      …
60      ctxt->vcpu->rip = _eip;
61      …
62 }
```

---

写寄存器的操作可能会伴随一些副作用，需要设备做些额外的操作：

```c
commit c5ec153402b6d276fe20029da1059ba42a4b55e5
KVM: enable in-kernel APIC INIT/SIPI handling
linux.git/drivers/kvm/kvm_main.c
int emulator_write_emulated(unsigned long addr, const void *val,…)
{
    …
    return emulator_write_emulated_onepage(addr, val, …);
}

static int emulator_write_emulated_onepage(unsigned long addr,…)
{
    …
    mmio_dev = vcpu_find_mmio_dev(vcpu, gpa);
    if (mmio_dev) {
        kvm_iodevice_write(mmio_dev, gpa, bytes, val);
        return X86EMUL_CONTINUE;
    }
    …
}
```

---

对于 LAPIC 模拟设备，这个函数是 apic_mmio_write:

```c {*}{maxHeight:'100px'}
commit c5ec153402b6d276fe20029da1059ba42a4b55e5
KVM: enable in-kernel APIC INIT/SIPI handling
linux.git/drivers/kvm/lapic.c
static void apic_mmio_write(struct kvm_io_device *this, …)
{
    …
    case APIC_ICR:
    …
    apic_send_ipi(apic);
    …
}
```

鉴于 LAPIC 的寄存器的访问非常频繁，所以 Intel 从硬件层面做了很多支持:

```c
commit f78e0e2ee498e8f847500b565792c7d7634dcf54
KVM: VMX: Enable memory mapped TPR shadow (FlexPriority)
linux.git/drivers/kvm/vmx.c
static int (*kvm_vmx_exit_handlers[])(…) = {
    …
    [EXIT_REASON_APIC_ACCESS] =
    handle_apic_access,
};
static int handle_apic_access(struct kvm_vcpu *vcpu, …)
{
    …
    er = emulate_instruction(vcpu, kvm_run, 0, 0, 0);
    …
}
```

---

### 2. PIO

PIO 使用专用的 I/O 指令（out、outs、in、ins）访问外设，当 Guest 通过这些专门的 I/O 指令访问外设时，
处于 Guest 模式的 CPU 将主动发生陷入，进入 VMM 。

Intel PIO 指令支持两种模式:

- 普通的 I/O: 一次传递 1 个值，对应于 x86 架构的指令 out、in
- string I/O: 一次传递多个值，对应于 x86 架构的指令 outs、ins

<br>

因此，对于普通的 I/O ，只需要记录下 val ，而对于 string I/O ，则需要记录下 I/O 值所在的地址。

---

我们以向块设备写数据为例，对于普通的 I/O ，其使用的是 out 指令:

|指令|描述|
| --- | --- |
| OUT imm8, AL | 将 AL 寄存器的值写入 imm8 指定的端口 |
| OUT imm8, AX | 将 AX 寄存器的值写入 imm8 指定的端口 |
| OUT imm8, EAX | 将 EAX 寄存器的值写入 imm8 指定的端口 |
| OUT DX, AL | 将 AL 寄存器的值写入 DX 寄存器中记录的 I/O 端口 |
| OUT DX, AX | 将 AX 寄存器的值写入 DX 寄存器中记录的 I/O 端口 |
| OUT DX, EAX | 将 EAX 寄存器的值写入 DX 寄存器中记录的 I/O 端口 |

<br>

> 可以看到，无论哪种格式，out 指令的源操作数都是寄存器 al、ax、eax 系列。

---

因此，当陷入 KVM 模块时，KVM 模块可以从 Guest 的 rax 寄存器的值中取出 Guest 准备写给外设的值，
KVM 将这个值存储到结构体 kvm_run 中。

```c
commit 6aa8b732ca01c3d7a54e93f4d701b8aabbe60fb7
[PATCH] kvm: userspace interface
linux.git/drivers/kvm/vmx.c
static int handle_io(struct kvm_vcpu *vcpu, …)
{
    …
    if (kvm_run->io.string) {
        …
        kvm_run->io.address =
        vmcs_readl(GUEST_LINEAR_ADDRESS);
    } else
        kvm_run->io.value = vcpu->regs[VCPU_REGS_RAX]; /* rax */
    return 0;
}
```

对于 string 类型的 I/O ，需要记录的是数据所在的内存地址，

这个地址在陷入 KVM 前，CPU 会将其记录在 VMCS 的字段 GUEST_LINEAR_ADDRESS 中，

KVM 将这个值从 VMCS 中读出来，存储到结构体 kvm_run 中
