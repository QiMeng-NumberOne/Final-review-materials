# Lec18：信号量和管程

> **与 Lec17 的衔接**：Lec17 讲了锁和原子操作等底层同步原语。Lec18 在此基础上构建**更高层的同步抽象**——信号量和管程，并用它们解决经典并发问题。

---

## 📑 目录

1. [同步机制层次回顾](#1-同步机制层次回顾)
2. [信号量（Semaphore）](#2-信号量semaphore)
   - 2.1 [P/V 操作内部逻辑](#21-pv-操作内部逻辑详解)
   - 2.2 [信号量两种用途](#22-信号量的两种用途)
3. [信号量解决经典同步问题](#3-信号量解决经典同步问题)
   - 3.1 [生产者-消费者问题](#31-生产者-消费者问题)
   - 3.2 [哲学家就餐问题（三种方案）](#32-哲学家就餐问题)
   - 3.3 [读者-写者问题](#33-读者-写者问题)
4. [管程（Monitor）](#4-管程monitor)
   - 4.1 [Lock 和 Condition 内部逻辑详解](#41-lock-和-condition-内部逻辑详解)
   - 4.2 [Hansen 管程 vs Hoare 管程](#42-hansen-管程-vs-hoare-管程)
5. [经典同步问题——管程版](#5-经典同步问题管程版)
   - 5.1 [生产者-消费者（管程版）](#51-生产者-消费者管程版)
   - 5.2 [读者-写者（管程版——写者优先）](#52-读者-写者管程版--写者优先)
6. [信号量 vs 管程对比](#6-信号量-vs-管程对比)
7. [实战例题](#7-实战例题)
8. [解题模板总结](#8-解题模板总结)
9. [知识脉络图](#9-知识脉络图)

---

## 1. 同步机制层次回顾

```
          高层抽象
    ┌──────────────┐
    │   管程        │ ← OO封装，锁+条件变量
    │  (Monitor)   │
    ├──────────────┤
    │   信号量      │ ← P()/V() 原子操作
    │ (Semaphore)  │
    ├──────────────┤
    │    锁        │ ← Lec17 自旋锁/阻塞锁
    │   (Lock)     │
    ├──────────────┤
    │  软件/硬件    │ ← Peterson、禁用中断、TS指令
    └──────────────┘
```

---

## 2. 信号量（Semaphore）

### 2.1 P/V 操作内部逻辑详解

> 信号量 = 一个整数 `sem` + 一个等待队列 `q` + 两个**原子**操作 P 和 V。

```c
class Semaphore {
    int sem;           // 计数器
    WaitQueue q;       // 等待队列
};
```

#### P 操作（申请资源）— 内部逻辑

```
P() {
    sem--;                         // ① 先减
    if (sem < 0) {                 // ② 减完 < 0 → 说明原来就是0（没资源了）
        Add this thread to q;      // ③ 自己进入等待队列
        block(p);                  // ④ CPU让出，进程阻塞
    }
    // sem ≥ 0 → 不阻塞，继续执行
}
```

**sem 值的含义**：

```
sem ≥ 0 → 还有 sem 个可用资源
sem < 0 → 有 |sem| 个线程在排队等待
```

**实例追踪**：

```
empty 初值 = 5

第1个线程 P(empty)：sem 5→4 ≥0 → 不阻塞 ✅ （还剩4个资源）
第2个线程 P(empty)：sem 4→3 ≥0 → 不阻塞 ✅
第3个线程 P(empty)：sem 3→2 ≥0 → 不阻塞 ✅
第4个线程 P(empty)：sem 2→1 ≥0 → 不阻塞 ✅
第5个线程 P(empty)：sem 1→0 ≥0 → 不阻塞 ✅ （刚好用完）
第6个线程 P(empty)：sem 0→-1 <0 → 阻塞！❌（没资源了，排队）
第7个线程 P(empty)：sem -1→-2 <0 → 阻塞！❌（第二个排队的）
```

#### V 操作（释放资源）— 内部逻辑

```
V() {
    sem++;                         // ① 先加
    if (sem ≤ 0) {                 // ② 加完 ≤ 0 → 说明变加之前是负数（有人排队）
        Remove a thread t from q;  // ③ 从等待队列取一个
        wakeup(t);                 // ④ 唤醒它（放回就绪队列）
    }
    // sem > 0 → 唤醒谁
}
```

**接上例**：

```
此时 sem = -2（有2个线程在排队）

线程执行 V(empty)：sem -2→-1 ≤0 → 从q取一个线程 → wakeup → 该线程回到就绪队列
线程执行 V(empty)：sem -1→0  ≤0 → 从q取一个线程 → wakeup
线程执行 V(empty)：sem 0→1   >0  → 不唤醒（已经没有等待者了）
```

#### PV 操作在线程阻塞/唤醒时的完整流程

```
图书馆场景中 empty 初值=5，mutex 初值=1

管理员A：
  P(empty) → sem 5→4 → 不阻塞 → P(mutex) → 0→-1→阻塞？NO!
  → 0 ≥ 0，不阻塞，拿到锁
  → 放书 → V(mutex) → sem -1→0 ≤0 → 唤醒！（说明有人在等锁）

但如果是：
  P(empty) → sem 5→4 → P(mutex) → sem 1→0 → 拿到锁
  此时管理员B也来了：
  P(empty) → sem 4→3 → P(mutex) → sem 0→-1 <0 → B被阻塞在mutex的等待队列！

  管理员A做完放书 V(mutex)：
  sem -1→0 ≤0 → 有人等！→ 从mutex等待队列取出B → wakeup(B)
  → B回到就绪队列 → 等CPU调度到B → B从P(mutex)返回 → 继续执行
```

---

### 2.2 信号量的两种用途

| 用途 | 初值 | 模式 | 示例 |
|------|:--:|------|------|
| **互斥访问** | 1 | P(mutex) ... V(mutex) | 保护临界区 |
| **条件同步** | 0 | P(cond) 等事件 / V(cond) 通知事件 | 等缓冲区非空 |

---

## 3. 信号量解决经典同步问题

### 3.1 生产者-消费者问题

```c
semaphore mutex = 1;          // 互斥
semaphore empty = N;          // 空闲槽位数（初值=容量N）
semaphore full = 0;           // 已填充槽位数（初值=0）

// 生产者
Deposit(c) {
    P(empty);                  // ① 先等空位（没空位就阻塞）
    P(mutex);                  // ② 再拿互斥锁
    Add c to buffer;
    V(mutex);                  // ③ 先释放互斥锁
    V(full);                   // ④ 满槽+1，唤醒消费者
}

// 消费者
Remove(c) {
    P(full);                   // ① 先等有数据（没数据就阻塞）
    P(mutex);                  // ② 再拿互斥锁
    Remove c from buffer;
    V(mutex);                  // ③ 先释放互斥锁
    V(empty);                  // ④ 空槽+1，唤醒生产者
}
```

> ⭐ **PV 顺序铁律**：永远**先 P 条件，再 P 互斥**；**先 V 互斥，再 V 条件**。否则死锁！

### 3.2 哲学家就餐问题

```
5个哲学家围圆桌，5支叉子在每两人之间。需要 同时拿左右两支叉子 才能进餐。
```

**方案一：朴素方案 → 死锁**

```c
semaphore fork[5] = {1,1,1,1,1};

philosopher(i) {
    think();
    P(fork[i]);  P(fork[(i+1)%5]);  // 先左后右
    eat();
    V(fork[i]);  V(fork[(i+1)%5]);
}
// ❌ 5人同时拿左边 → 全等着拿右边 → 死锁！
```

**方案二：全局互斥 → 正确但低效**

```c
semaphore mutex = 1;
philosopher(i) {
    think();
    P(mutex);                    // 整桌互斥！
    P(fork[i]); P(fork[(i+1)%5]);
    eat();
    V(fork[i]); V(fork[(i+1)%5]);
    V(mutex);
}
// ✅ 无死锁，❌ 每次只有1人能吃
```

**方案三：奇偶编号不同顺序 → 正确+高效**

```c
philosopher(i) {
    think();
    if (i % 2 == 0) {          // 偶数：先左后右
        P(fork[i]);  P(fork[(i+1)%5]);
    } else {                   // 奇数：先右后左
        P(fork[(i+1)%5]);  P(fork[i]);
    }
    eat();
    V(fork[i]);  V(fork[(i+1)%5]);
}
// ✅ 无死锁（打破循环等待）✅ 可多人同时吃
```

### 3.3 读者-写者问题

```
约束：读-读允许，读-写互斥，写-写互斥
```

```c
// 读者优先版
semaphore WriteMutex = 1;     // 读写互斥
semaphore CountMutex = 1;     // 保护 Rcount
int Rcount = 0;

// 读者
Reader() {
    P(CountMutex);
    if (Rcount == 0) P(WriteMutex);   // 第一个读者锁门
    Rcount++;
    V(CountMutex);
    
    read();
    
    P(CountMutex);
    Rcount--;
    if (Rcount == 0) V(WriteMutex);   // 最后一个读者开门
    V(CountMutex);
}

// 写者
Writer() {
    P(WriteMutex);                    // 等所有读者离开
    write();
    V(WriteMutex);
}
```

---

## 4. 管程（Monitor）

### 4.1 Lock 和 Condition 内部逻辑详解

> 管程的组成：1个 Lock + 共享数据 + N个 Condition + 成员函数。

#### Lock::Acquire / Release 内部逻辑

```
Lock::Acquire() {
    while (锁被持有) {
        自己加入锁的等待队列;
        block();            // 阻塞
    }
    标记锁为"当前线程持有";
}

Lock::Release() {
    标记锁为"空闲";
    if (锁的等待队列非空) {
        取出一个线程;
        wakeup(它);
    }
}
```

#### Condition::Wait 内部逻辑

```
Condition::Wait(lock) {
    numWaiting++;                     // ① 等待计数+1
    自己加入这个条件的等待队列;         // ② "我在等这个条件"
    lock->Release();                  // ③ 释放锁！别人可以进来了！
    schedule();                       // ④ 让出CPU
    lock->Acquire();                  // ⑤ 被唤醒后重新拿锁
}

关键：Wait 会先释放锁再阻塞 —— 
     这样别人才能进入管程改变条件，然后唤醒你。
```

#### Condition::Signal / Broadcast

```
Signal() {
    if (条件的等待队列非空) {
        取一个线程 → wakeup → 移到就绪队列;
    }
}

Broadcast() {
    while (条件的等待队列非空) {
        取一个线程 → wakeup;
    }
}
```

### 4.2 Hansen 管程 vs Hoare 管程

```
Hoare管程（教材常用）：Signal后，被唤醒者立即接管管程 → if 足够
  Deposit() {
      lock->acquire();
      if (count == n)            // ← if！
          notFull.Wait(&lock);
      // ... 
  }

Hansen管程（真实OS/Java）：Signal后，自己继续执行 → 必须 while
  Deposit() {
      lock->acquire();
      while (count == n)         // ← while！重要！
          notFull.Wait(&lock);
      // ... 
  }

原因：Hansen中Signal只是通知，被唤醒者不立即执行。
     在被唤醒者"醒来"之前，可能有别人进来，条件可能又变了。
```

---

## 5. 经典同步问题——管程版

### 5.1 生产者-消费者（管程版）

```c
class BoundedBuffer {
    Lock lock;
    int count = 0;
    Condition notFull, notEmpty;
};

Deposit(c) {
    lock->Acquire();
    while (count == n)              // 满 → 等
        notFull.Wait(&lock);
    Add c;  count++;
    notEmpty.Signal();              // 通知：不空了
    lock->Release();
}

Remove(c) {
    lock->Acquire();
    while (count == 0)              // 空 → 等
        notEmpty.Wait(&lock);
    Remove c;  count--;
    notFull.Signal();               // 通知：不满了
    lock->Release();
}
```

### 5.2 读者-写者（管程版 —— 写者优先）

```c
// 状态变量
AR=0; AW=0;  // 活跃读者/写者数
WR=0; WW=0;  // 等待读者/写者数
Lock lock;
Condition okToRead, okToWrite;

// 读者
Private StartRead() {
    lock.Acquire();
    while ((AW + WW) > 0) {   // 有活跃写者 或 等待写者 → 读者等
        WR++;  okToRead.Wait(&lock);  WR--;
    }
    AR++;
    lock.Release();
}
Private DoneRead() {
    lock.Acquire();
    AR--;
    if (AR == 0 && WW > 0)
        okToWrite.Signal();    // 最后一个读者离开，唤醒写者
    lock.Release();
}
Public Read() {
    StartRead();
    read database;
    DoneRead();
}

// 写者
Private StartWrite() {
    lock.Acquire();
    while ((AW + AR) > 0) {   // 有活跃读者或写者 → 写者等
        WW++;  okToWrite.Wait(&lock);  WW--;
    }
    AW++;
    lock.Release();
}
Private DoneWrite() {
    lock.Acquire();
    AW--;
    if (WW > 0)
        okToWrite.Signal();         // 优先唤醒写者
    else if (WR > 0)
        okToRead.Broadcast();       // 然后唤醒所有读者
    lock.Release();
}
Public Write() {
    StartWrite();
    write database;
    DoneWrite();
}
```

---

## 6. 信号量 vs 管程对比

| | 信号量 | 管程 |
|------|------|------|
| **编程模型** | 过程式，P/V 分散 | 面向对象，方法封装 |
| **易用性** | 易出错（顺序/配对） | 较安全（管程保证互斥） |
| **共享数据** | 程序员自行保护 | 管程自动保护 |
| **等待机制** | P操作阻塞在信号量等待队列 | Wait释放锁后阻塞在条件等待队列 |
| **死锁处理** | 无法直接处理 | 无法直接处理 |
| **典型场景** | 早期OS、教学 | 现代OS、Java |

---

## 7. 实战例题

### 7.1 图书馆阅览台（信号量，中等）

```
场景：阅览台最多放5本书。
管理员A放文学书，管理员B放理工书。
读者甲取理工书，读者乙取文学书。
每次仅1人操作阅览台。

分析：
  4个约束 → 4个信号量
  ① 有空位才能放 → empty=5
  ② 有理工书才能取 → science=0
  ③ 有文学书才能取 → literature=0
  ④ 互斥 → mutex=1
```

```c
semaphore mutex=1, empty=5, science=0, literature=0;

AdminA() {  // 放文学书
    P(empty); P(mutex);
    放文学书;
    V(mutex); V(literature);
}
AdminB() {  // 放理工书
    P(empty); P(mutex);
    放理工书;
    V(mutex); V(science);
}
ReaderA() { // 取理工书
    P(science); P(mutex);
    取理工书;
    V(mutex); V(empty);
}
ReaderB() { // 取文学书
    P(literature); P(mutex);
    取文学书;
    V(mutex); V(empty);
}
```

### 7.2 十字路口（管程，较难）

```c
class Intersection {
    Lock lock;
    int onBridge=0, currentDir=-1;
    Condition okToCross;
    
    void Arrive(int dir) {
        lock->Acquire();
        while (onBridge>0 && currentDir!=dir)
            okToCross.Wait(&lock);
        onBridge++; currentDir=dir;
        lock->Release();
    }
    void Leave(int dir) {
        lock->Acquire();
        onBridge--;
        if (onBridge==0) {
            currentDir=-1;
            okToCross.Broadcast();
        }
        lock->Release();
    }
};
```

---

## 8. 解题模板总结

```
┌─────────────────────────────────────────────────────────┐
│               信号量/管程 解题通用模板                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  第一步：确定共享资源                                         │
│    每个共享资源 → 信号量/条件变量                             │
│    互斥 → mutex=1   计数 → sem=N   同步 → sem=0            │
│                                                         │
│  第二步：明确"操作前条件" 和 "操作后结果"                       │
│    放书：P(empty) → 放 → V(literature)                    │
│    取书：P(literature) → 取 → V(empty)                    │
│                                                         │
│  第三步：PV 顺序铁律                                        │
│    P(条件) → P(互斥) → 操作 → V(互斥) → V(条件)              │
│    永远先P条件后P互斥，先V互斥后V条件                          │
│                                                         │
│  第四步：死锁检查                                           │
│    是否持有锁在等待？   while中是否释放了锁？                    │
│                                                         │
│  第五步（管程版）：                                          │
│    Lock ×1，Condition ×N（每个等待原因一个）                   │
│    while(条件不满足) cond.Wait(&lock)                       │
│    操作后 cond.Signal/Broadcast                            │
│    Hansen用 while，Hoare 用 if                             │
│    唤醒所有等待者 → Broadcast                                │
└─────────────────────────────────────────────────────────┘
```

---

## 9. 知识脉络图

```
Lec17 底层同步
  ├── 禁用中断、原子操作(TS)
  ├── 软件方法(Peterson...)
  └── 锁(自旋锁/阻塞锁)
        │
        ▼
Lec18 高层抽象
  │
  ├── 信号量(Semaphore)
  │   ├── P() = sem--; if(sem<0) block
  │   ├── V() = sem++; if(sem≤0) wakeup
  │   ├── 互斥(mutex=1) + 条件同步(cond=0)
  │   ├── PV顺序：P条件→P互斥→操作→V互斥→V条件
  │   └── 经典问题：生产-消费/哲学家/读-写
  │
  └── 管程(Monitor)
      ├── Lock(互斥) + Condition(同步)
      ├── Wait(lock) = 释放锁+阻塞+醒后重拿锁
      ├── Signal = 唤醒一个 / Broadcast = 唤醒全部
      ├── Hansen(while) vs Hoare(if)
      └── 经典问题：生产者/读者-写者(写者优先)
```
