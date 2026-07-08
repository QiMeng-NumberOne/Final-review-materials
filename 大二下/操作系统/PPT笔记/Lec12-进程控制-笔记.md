# Lec12：进程控制

> **与 Lec11 的衔接**：Lec11 讲了进程/线程的概念、状态模型、PCB。Lec12 讲**怎么操作进程**——创建(fork)、加载(exec)、等待(wait)、升退出(exit)、切换(switch)。

---

## 📑 目录

1. [进程创建 — fork()](#1-进程创建--fork)
2. [进程加载 — exec()](#2-进程加载--exec)
3. [进程等待 — wait()](#3-进程等待--wait)
4. [进程退出 — exit()](#4-进程退出--exit)
5. [进程切换 — Context Switch](#5-进程切换--context-switch)
6. [ucore 中的进程控制实现](#6-ucore-中的进程控制实现)
7. [其他进程控制系统调用](#7-其他进程控制系统调用)
8. [FAQ](#8-faq)
9. [知识脉络图](#9-知识脉络图)

---

## 1. 进程创建 — fork()

### 1.1 fork() 原理

> `fork()` 把一个进程**复制**成两个：父进程（old PID）和子进程（new PID）。

```
fork() 的返回值（⭐ 核心考点）：

  父进程中 fork() → 返回子进程的 PID（> 0）
  子进程中 fork() → 返回 0
  出错            → 返回 -1

父子进程通过返回值来区分身份，执行不同的代码分支。
```

#### 基本使用模式

```c
main() {
    int childPID;
    s1;                                // ① 只有父进程执行
    childPID = fork();                 // ② 这里一分为二！
    
    if (childPID == 0)
        <子进程执行代码>                // ③ 只有子进程进入
    else {
        <父进程执行代码>                // ④ 只有父进程进入
        wait();
    }
    s2;
}
```

### 1.2 地址空间复制

> 对于子进程而言，`fork()` 是在调用时间对父进程地址空间的**一次复制**。

```
fork() 之前：                       fork() 之后：
                                    
父进程                              父进程              子进程
┌──────────┐                       ┌──────────┐        ┌──────────┐
│ 代码      │                       │ 代码      │        │ 代码      │ ← 复制
│ 数据      │          →            │ 数据      │        │ 数据      │ ← 复制
│ 堆        │                       │ 堆        │        │ 堆        │ ← 复制
│ 栈        │                       │ 栈        │        │ 栈        │ ← 复制
│ 寄存器    │                       │ 寄存器    │        │ 寄存器    │ ← 复制(除eax)
│ 打开文件   │                       │ 打开文件   │        │ 打开文件   │ ← 复制(共享fd)
│ PCB       │                       │ PCB       │        │ PCB       │ ← 新建
└──────────┘                       └──────────┘        └──────────┘
                                   childPID=子PID      childPID=0
```

**复制的内容**：
- 代码段、数据段、堆、栈 → 全部复制
- CPU 寄存器 → 全部复制（**除了 eax**，eax 存的是返回值）
- 打开文件列表 → 共享（父子指向同一文件描述符表项）

### 1.3 fork() 的开销问题

```
fork() 的开销：
  ● 为子进程分配新的物理内存
  ● 逐页复制父进程的内存内容
  ● 复制 CPU 寄存器
  → 开销昂贵！！

问题：99% 情况下，fork() 之后马上调用 exec()
  → fork() 中复制内存是 白费力气
  → 子进程马上用 exec() 加载新程序，所有内存全部被替换
  → 复制了半天，全扔了！
```

**解决方案演进**：

```
① vfork()（早期方案）：
  创建子进程时 不再复制内存映像
  子进程直接共享父进程的地址空间
  子进程应立即调用 exec()
  → 轻量级 fork()

② Copy on Write, COW（现代方案）：
  fork() 时 不复制 物理内存
  父子进程 共享 同一份物理页帧
  页表项标记为 只读
  只有当某一方 写入 时才复制 那一页！

  COW 如何工作（联系 Lec6/8 页表知识）：
    ① fork()：父子页表指向同一物理帧，标记只读（U/S=1, W=0）
    ② 进程尝试写 → 触发页保护异常 → 陷入内核
    ③ OS 分配新帧 → 复制原帧内容 → 修改写者页表指向新帧
    ④ 返回用户态 → 写入成功
    → 没写到的页永远不复制！省时又省内存
```

### 1.4 fork() 的进程树示例（PPT）

```c
int main() {
    pid_t pid;
    int i;
    for (i = 0; i < LOOP; i++) {
        pid = fork();
        if (pid < 0) {
            fprintf(stderr, "Fork Failed");
            exit(-1);
        } else if (pid == 0) {
            // 子进程：打印自己的信息
            fprintf(stdout, "i=%d, pid=%d, parent pid=%d\n",
                    i, getpid(), getppid());
        }
    }
    wait(NULL);
    exit(0);
}
```

```
执行结果（进程树，以 LOOP=3 为例）：
                        
        i=0, pid=1167, parent=1166          ← 第1次 fork 产生
           ┌────────────┼────────────────┐
        i=1           i=1              i=2            ← 第2次 fork
    pid=1168       pid=1170         pid=1169
    parent=1166    parent=1167      parent=1166
        │              │
    i=2,pid=1172   i=2,pid=1173    ← 第3次 fork
    parent=1168    parent=1170

关键：子进程会继续执行循环 → 指数级增长！
```

---

## 2. 进程加载 — exec()

### 2.1 exec() 原理

> `exec()` 用**新程序重写当前进程**，**PID 不变**。

```
exec() 调用成功时：
  ● 相同的进程（PID 不变）
  ● 代码段 → 完全重写（新程序的指令）
  ● 数据段 → 完全重写（新程序的全局变量）
  ● 堆和栈 → 全部清空重建
  ● 从新程序的 _start（main）开始执行
  ● 允许指定启动参数（argc, argv）

exec() 一旦成功 → 原程序剩下的代码 永远不会被执行！
```

### 2.2 fork + exec 经典模式

```c
main() {
    int pid = fork();           // ① 创建子进程
    if (pid == 0) {             // ② 子进程进入
        exec_status = exec("calc", argc, argv0, argv1, ...);
        printf("Why would I execute?");  // ← 永远不会执行！
        // exec 成功后，当前进程已被 calc 完全替换
    } else {                    // ③ 父进程进入
        printf("Whose your daddy?");
        child_status = wait(pid);
    }
}
```

### 2.3 Shell 执行命令的完整流程（PPT 例图）

```
用户在 shell 中输入 /bin/calc 回车：

时间线 →
                                                
内核态                   用户态                   
                                                  
① shell 进程 (PID=127) 运行中                    
   PCB: name="sh", pid=127                        
                                                  
② fork() 调用 ─────────────────────────→ 创建子进程
   PCB: pid=128, name="sh"(暂)                    
                                                  
③ 子进程 (PID=128) 是 shell 的复制                
   fork 返回 0 → 进入 if(pid==0) 分支              
                                                  
④ exec("/bin/calc") 调用               
   替换进程内容：                                  
   代码段 → calc 的代码                             
   数据段 → calc 的数据                             
   堆/栈  → 清空重建                                
   打开文件 → 可以重新指定                           
   但 PID=128 不变！                              
                                                  
⑤ 现在 PID=128 运行 calc 程序                      
   PCB: name="calc", pid=128                      

关键：同一个 PID，同一个 PCB，但里面跑的程序完全变了！
     这就是 fork+exec 的精髓。
```

### 2.4 Windows vs Unix 进程创建

| | Unix/Linux | Windows |
|------|------|------|
| 创建进程 | `fork()` + `exec()` 两步 | `CreateProcess(filename)` 一步 |
| 特点 | 先复制再替换 | 直接创建并加载新程序 |
| 灵活性 | fork 后可修改子进程环境再 exec | 参数一次性指定 |
| 类似效果 | `CreateProcess(f, CLOSE_FD, new_envp)` | |

---

## 3. 进程等待 — wait()

### 3.1 wait() 的功能

> 父进程通过 `wait()` **等待子进程结束**并获取其返回值。

```
wait() 系统调用的行为：

  ① 有子进程存活 → 父进程进入 等待状态(Blocked)，等待子进程结束
  ② 有僵尸子进程等待回收 → wait() 立即返回其中一个子进程的返回值
  ③ 无子进程存活 → wait() 立刻返回（-1）

当某子进程调用 exit() 时：
  → 唤醒父进程
  → 将 exit() 的返回值 作为父进程中 wait() 的返回值
```

### 3.2 父子同步流程

```
父进程                          子进程
  │                               │
  │ fork() ────────────────────→ 创建
  │                               │
  │ 继续运行...                     │ 运行中...
  │                               │
  │ wait(&status)                  │
  │ 进入等待状态 ⏸                  │
  │   │                           │ exit(42)
  │   │   唤醒 ←───────────────────┘
  │   │                           ✗ 结束
  │ 被唤醒                         
  │ status = 42   ← 得到了子进程的返回值
  │
  ▼ 继续运行
```

---

## 4. 进程退出 — exit()

### 4.1 exit() 的功能

> `exit()` = 进程终止时的**最终垃圾收集**，回收进程占用的所有资源。

```
exit() 系统调用做的事（按顺序）：

  ① 将调用参数作为进程的"结果"（会传给父进程的 wait()）
  ② 关闭所有打开的文件等占用资源
  ③ 释放内存（代码段、数据段、堆、栈占用的所有物理帧）
  ④ 释放大部分进程相关的内核数据结构
  ⑤ 清理所有等待的僵尸进程（该进程的子进程们）
  ⑥ 检查父进程是否存活：
     ● 父进程存活 → 保留返回值，进入 僵尸状态(zombie/defunct)
     ● 父进程已死 → 直接释放所有数据结构，结果丢弃
```

### 4.2 僵尸进程（Zombie）

```
僵尸进程 ≠ 还在运行的进程

僵尸进程：子进程已经 exit() 了，但父进程还没有调用 wait()
         → PCB 和返回值 被内核保留
         → 进程名还在列表里，状态显示为 <defunct>
         → 这个"空壳 PCB"就是僵尸

生命周期：
  子进程 exit() → 变成僵尸 → 父进程 wait() → PCB 回收 → 僵尸消失
  
  父进程在子进程之前死了 → 僵尸变孤儿 → init(PID=1) 收养
  → init 定期调用 wait() → 清理所有孤儿僵尸
```

### 4.3 进程状态与系统调用的对应

```
          fork()              wait()
            │                   │
            ▼                   ▼
创建 ──→ 就绪 ⇄ 运行 ──→ 等待 ──→ 结束 ──→ 僵尸 ──→ wait回收
            ▲                   │
            │  时间片完           │ exit()
            └───────────────────┘

fork()  → 创建态 → 就绪态
exec()  → 不改变状态（只是替换内容）
wait()  → 运行态 → 等待态（等待子进程）
exit()  → 运行态 → 结束态 → 僵尸态
wait()  → 回收僵尸 → PCB 释放
```

---

## 5. 进程切换 — Context Switch

### 5.1 什么是上下文切换

> 暂停当前运行进程，恢复另一个进程，让它们**无缝接力**。

```
进程上下文（Context）= 进程生命周期的全部执行状态：
  ● 寄存器：EIP(PC)、ESP(SP)、EBP、EAX、EBX、ECX、EDX、ESI、EDI
  ● CPU 状态寄存器：EFLAGS
  ● 内存地址空间：CR3（页表基址）← 联系 Lec6 PTBR

切换三步走：
  ① 保存当前进程的上下文 → PCB 的 context 字段
  ② 选择下一个就绪进程 → schedule()
  ③ 从新进程的 PCB 恢复上下文 → 仿佛从未被打断
```

### 5.2 上下文切换图解（PPT）

```
时间 →
进程P0: ████████████░░░░░░░░░░░░░░░░░░░░████████████
        运行中       │   保存到PCB0      │  从PCB0恢复   │  运行中
                    ▼                   ▲
内核:           ┌──────────┐       ┌──────────┐
               │schedule()│  ...  │schedule()│
               │选P1运行   │       │选P0运行   │
               └──────────┘       └──────────┘
                    │                   ▲
进程P1: ░░░░░░░░░░░░████████████████████░░░░░░░░░░░░
                  从PCB1恢复  运行中        保存到PCB1

空闲(idle): ░░░░░░░░              ░░░░░░░░░░
          没有进程可运行时跑空闲进程
```

### 5.3 switch_to 汇编实现（PPT 原码逐行解读）

```asm
switch_to:                      # switch_to(from, to)
    # ═══ 第一步：保存 from 进程的寄存器到它的 PCB ═══
    movl 4(%esp), %eax          # eax = &from->context（PCB中存上下文的地址）
    popl 0(%eax)                # 弹出返回地址EIP → 存入 from->context.eip
    movl %esp, 4(%eax)          # 保存 ESP → from->context.esp
    movl %ebx, 8(%eax)          # 保存 EBX → from->context.ebx
    movl %ecx, 12(%eax)         # 保存 ECX → from->context.ecx
    movl %edx, 16(%eax)         # 保存 EDX → from->context.edx
    movl %esi, 20(%eax)         # 保存 ESI → from->context.esi
    movl %edi, 24(%eax)         # 保存 EDI → from->context.edi
    movl %ebp, 28(%eax)         # 保存 EBP → from->context.ebp

    # ═══ 第二步：恢复 to 进程的寄存器 ═══
    movl 4(%esp), %eax          # eax = &to->context
    movl 28(%eax), %ebp         # 恢复 EBP
    movl 24(%eax), %edi         # 恢复 EDI
    movl 20(%eax), %esi         # 恢复 ESI
    movl 16(%eax), %edx         # 恢复 EDX
    movl 12(%eax), %ecx         # 恢复 ECX
    movl 8(%eax), %ebx          # 恢复 EBX
    movl 4(%eax), %esp          # 恢复 ESP

    # ═══ 第三步：跳转到 to 进程 ═══
    pushl 0(%eax)               # 把 to->context.eip 压栈
    ret                         # 弹出 EIP → 跳转到新进程继续执行！

注意：EAX 不保存/恢复，因为它是返回值寄存器
     switch_to 本身通过 EAX 传递 from/to 指针
```

### 5.4 进程上下文结构体（ucore）

```c
struct context {
    uint32_t eip;    // 指令指针（PC）
    uint32_t esp;    // 栈指针
    uint32_t ebx;    // 通用寄存器
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;    // 帧指针
};
```

---

## 6. ucore 中的进程控制实现

### 6.1 proc_struct（PCB 完整结构体）

```c
struct proc_struct {
    // === 进程标识 ===
    char name[PROC_NAME_LEN + 1];   // 进程名
    int pid;                         // 进程ID
    uint32_t flags;                 // 进程标志

    // === 进程状态与调度 ===
    enum proc_state state;           // 就绪/运行/等待/僵尸
    volatile bool need_resched;      // 是否需要重新调度
    int runs;                        // 运行次数

    // === 内存管理（联系 Lec6 段页式！）===
    struct mm_struct *mm;           // 内存管理结构（代码/数据/堆/栈的VMA链表）
    uintptr_t cr3;                  // 页表基址 ← Lec6 的 PTBR！

    // === 上下文保存 ===
    struct context context;          // switch_to 保存/恢复的寄存器
    struct trapframe *tf;           // 中断/异常/系统调用时的完整现场

    // === 栈 ===
    uintptr_t kstack;               // 内核栈基址

    // === 进程关系 ===
    struct proc_struct *parent;     // 父进程指针

    // === 队列链接 ===
    list_entry_t hash_link;         // 哈希表链接（按PID快速查找）
    list_entry_t list_link;         // 进程链表链接
};
```

### 6.2 mm_struct（内存管理结构）

```c
struct mm_struct {
    list_entry_t mmap_list;         // VMA 链表（代码段+数据段+堆+栈+共享库...）
    struct vma_struct *mmap_cache;  // 当前访问的 VMA 缓存（加速查找）
    pde_t *pgdir;                   // 页目录基址 = CR3
    int map_count;                  // VMA 区域数量
    void *sm_priv;                  // 交换管理器私有数据
};
```

**mm_struct 与前章知识的对应**：

```
mm_struct 里的 VMA 链表：
  VMA[0] → 代码段：起始0x400000, 大小100KB, 只读, 可执行
  VMA[1] → 数据段：起始0x402000, 大小50KB, 读写
  VMA[2] → 堆：    起始0x405000, 大小可变, 读写
  VMA[3] → 栈：    起始0xBFFF0000, 大小可变, 读写
  ...

每个 VMA 描述了一段连续的 逻辑地址空间
底层由 页表（pgdir = CR3）映射到物理帧 ← Lec6 段页式！
```

### 6.3 空闲进程（idleproc）的创建

```
系统启动 → proc_init()：

  ① alloc_proc() → kmalloc() 分配 PCB 内存
  ② 分配 idleproc 需要的资源
  ③ 初始化 idleproc 的 PCB 字段
  ④ idleproc->pid = 0（系统第一个进程）
  ⑤ idleproc 永远处于就绪态，优先级最低
     → 没有其他进程可运行时就跑 idleproc
```

### 6.4 第一个内核线程（initproc）的创建

```
proc_init() 继续 → 创建 initproc（PID=1）：

  ① alloc_proc() → 分配新 PCB
  ② setup_stack() → 初始化 initproc 的内核堆栈
  ③ copy_stack() → 设置栈内容
  ④ copy_thread() → 设置线程上下文（tf、context）
  ⑤ do_fork() → 以 idleproc 为父进程创建子进程
  ⑥ 初始化 trapframe（系统调用/中断时的现场保存区）
  ⑦ 把 initproc 加入就绪队列
  ⑧ 唤醒 initproc → kernel_thread() → 开始执行
```

### 6.5 调度流程

```
schedule() 的完整流程：

  ① need_resched 标志被设置（时间片到 / 进程主动让出）
  ② 进入 schedule()
  ③ 如果当前进程状态为 RUNNING → 改回 READY → 放回就绪队列
  ④ sched_class_pick_next() → 从就绪队列选下一个进程
  ⑤ sched_class_dequeue() → 从就绪队列摘下
  ⑥ sched_class_enqueue() → 如果需要，另一个队列
  ⑦ proc_run() → 修改 CR3，switch_to() → 完成切换
```

---

## 7. 其他进程控制系统调用

| 系统调用 | 功能 |
|------|------|
| `sleep(n)` | 让进程在定时器等待队列中等待 n 秒 |
| `nice(prio)` | 指定进程的初始优先级（Unix 中优先级随时间衰减） |
| `ptrace(cmd, pid)` | 允许一个进程控制另一个进程的执行（调试器用：设断点、读寄存器） |

---

## 8. FAQ

### Q1：fork() 的返回值为什么父进程和子进程不同？

父子进程通过 fork() 的**不同返回值**来区分身份，进入不同的代码分支。这是内核故意设计的——子进程的 eax 寄存器被设为 0，父进程的 eax 被设为子进程 PID。

### Q2：为什么要有 COW（Copy on Write）？

fork() 后 99% 的情况是马上 exec()。fork 中复制全部内存 → 费力不讨好。COW 让父子**先共享**物理页，**写时才复制** → 省时间又省内存。

### Q3：exec() 之后 PID 变了吗？

**不变。** PID 不变，但进程的代码/数据/堆/栈被新程序**完全替换**。同一个进程，换了一个程序在跑。

### Q4：僵尸进程是什么？怎么清理？

子进程已 exit()，但父进程还没 wait() → PCB 残留 = 僵尸。父进程调用 wait() 或父进程死亡（init 收养清理）即可清除。

### Q5：switch_to 为什么不保存 eax？

eax 是返回值寄存器，switch_to 本身用它传递 `from`/`to` 指针。调用约定中 eax 是调用者负责保存的临时寄存器。

### Q6：进程切换和线程切换的区别？

- 进程切换 → 切换 CR3（页表）→ TLB 刷新 → **慢**
- 线程切换（同进程内）→ CR3 不变 → **快**

联系 Lec12 的 `switch_to` 和 Lec11 的线程概念。

---

## 9. 知识脉络图

```
进程控制（Lec12）
│
├── 创建 fork()
│   ├── 复制父进程地址空间 → 父子各自运行
│   ├── 返回值：父得子PID，子得0
│   ├── 开销 → COW 解决（fork时不复制，写时才复制页）
│   └── 经典模式：fork + exec
│
├── 加载 exec()
│   ├── 用新程序替换当前进程（PID 不变）
│   ├── 代码/数据/堆栈全部重写
│   └── Shell 执行命令的完整链路
│
├── 等待 wait()
│   ├── 父进程等待子进程结束
│   └── 获取 exit() 返回值
│
├── 退出 exit()
│   ├── 释放所有资源（关文件、释内存、清PCB）
│   └── 保留返回值 → 进入僵尸状态 → 等父进程 wait 回收
│
├── 切换 switch_to
│   ├── 保存 from 的寄存器到 PCB.context
│   ├── 恢复 to 的寄存器 ← PCB.context
│   └── 汇编实现：push/pop/mov 全套寄存器
│
├── ucore 实现
│   ├── proc_struct：完整 PCB（含 cr3/mm/context/tf）
│   ├── 创建链：alloc_proc → setup_stack → copy_thread → do_fork
│   ├── idleproc(PID=0) → initproc(PID=1)
│   └── schedule → pick_next → proc_run → switch_to
│
└── 与前面章节的串联
    ├── fork() COW 技术 → Lec8 页保护 + 缺页处理
    ├── cr3/mm_struct → Lec6 段页式 + PTBR
    ├── 进程挂起 → Lec11 七状态模型
    └── 进程地址空间 → Lec6 分段/分页/Lec8 虚拟存储
```
