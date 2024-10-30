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