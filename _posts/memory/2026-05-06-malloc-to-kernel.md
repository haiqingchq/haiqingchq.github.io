---
categories: [memory]
layout: single
title: "内存原理（一）：从 malloc 到内核，一次内存申请发生了什么"
published: true
series: 内存原理专栏
series_id: memory
series_order: 1
tags: [memory, linux, kernel, malloc]
---

# 内存原理（一）：从 malloc 到内核，一次内存申请发生了什么

一行 `malloc(100)` 之后，内存到底从哪里来？什么时候真正占用物理内存？为什么 `top` 里进程显示用了 1GB，`free` 命令却显示空闲内存还有很多？

这些问题的答案，全都藏在「应用 ⇄ 标准库 ⇄ 内核 ⇄ 硬件」这条链路里。本篇从一次最普通的 `malloc(100)` 出发，把整条链路串清楚；后续每篇会拆其中的一环深入讲。

## 1. 先搞清楚四个角色

申请内存这件事，本质上是**多层中介**在协作。用一个餐厅的比喻：

| 比喻 | 对应概念 | 职责 |
|---|---|---|
| 顾客 | 你的 C/C++ 程序 | 提需求："我要 100 字节内存"（`malloc`） |
| 服务员 | glibc | 顾客唯一能直接接触到的接口 |
| 大堂经理 | ptmalloc2（分配器） | glibc 里的专门负责人，手里有一堆现成桌椅（内存块）。顾客要的少时直接安排现成的；要包场（超大内存）时才去找房东 |
| 房东 | Linux 内核 | 拥有真正的土地（物理内存）。只有经理搞不定时，才通过系统调用（`mmap` / `brk`）批地 |

为什么这样设计？**因为系统调用很贵**。每次陷入内核都要切换上下文、刷 TLB，频繁系统调用会显著拖慢程序。所以 glibc 在用户态里加了一层"内存池"——能复用就绝不打扰内核。

## 2. 完整链路：从 `malloc` 到物理内存

| 阶段 | 动作 | 发生位置 | 关键机制 | 物理内存变化 |
|---|---|---|---|---|
| 1. 申请 | `malloc` / `new` | 用户态（C 库） | 内存池分配 / 系统调用（`mmap` / `brk`） | 无变化 |
| 2. 映射 | 内核响应请求 | 内核态（OS） | 创建 VMA，更新页表（标记"不存在"） | 无变化（仅分配虚拟地址） |
| 3. 访问 | `*ptr = 10` | CPU（硬件） | MMU 查页表失败 | 无变化 |
| 4. 中断 | 触发异常 | 硬件 → 内核 | 缺页中断（Page Fault） | 无变化 |
| 5. 分配 | 内核处理中断 | 内核态（OS） | 分配物理页框，更新页表映射 | **分配物理内存** |
| 6. 完成 | 重新执行指令 | CPU | 写入数据 | 数据写入物理内存 |

三个反直觉的点先记住：

1. **`malloc` 返回成功 ≠ 拿到物理内存**——要等到第 5 步首次写入时才真正占用 RAM。
2. **第 1 步不一定进内核**——如果 ptmalloc2 的内存池里有合适块，直接复用就返回，根本不调 `mmap` / `brk`。
3. **第 4 步的"中断"是硬件触发的**——CPU 访问到没建立映射的虚拟地址，MMU 直接抛异常给内核，程序无法绕过。

下面逐阶段拆。

## 3. 每个阶段在做什么

### 3.1 申请：`malloc(100)` 在用户态做了什么

`malloc` 不是系统调用，是 glibc 提供的**用户态函数**，内部走 ptmalloc2 分配器。

ptmalloc2 维护多个 **bin（按大小分级的空闲块链表）**：

1. 先看 bin 里有没有 ≥ 100 字节的空闲块 → 有就分一块出去，**不进内核**
2. 没有就向内核要新的"地"：
   - 小请求（默认 < 128KB，由 `M_MMAP_THRESHOLD` 控制）：用 `brk` 扩展堆顶
   - 大请求（≥ 128KB）：用 `mmap` 单独申请一段匿名映射

观察方法：

```bash
strace -e trace=brk,mmap,munmap ./your_program
```

如果你的程序循环 `malloc(100)` 一万次，你会发现绝大多数次都没出现 `brk` / `mmap`——那是 ptmalloc2 在内部复用。

### 3.2 映射：拿到的不是真内存，是"借条"

不论走 `brk` 还是 `mmap`，内核做的事其实只是**记账**：

- 在进程的 `mm_struct` 里登记一个 **VMA（virtual memory area）**，描述"这段虚拟地址属于本进程"
- 页表项保持"不存在"状态（`Present` 位为 0）

可以用 `cat /proc/<pid>/maps` 查看进程当前所有 VMA。

此时 `malloc` 已经返回了一个看似可用的虚拟地址指针，但**物理内存一字节都没分配**。这就是俗称的 **lazy allocation（按需分配）**。

### 3.3 访问 → 中断：第一次写入触发缺页

当程序执行 `*ptr = 10`：

1. CPU 把虚拟地址送到 **MMU（内存管理单元）**，MMU 查页表
2. 页表项 `Present = 0` → MMU 触发 **Page Fault** 异常
3. CPU 把控制权交给内核的缺页处理入口（`do_page_fault`）

这一步是硬件强制的，应用程序无法干预。

### 3.4 分配 → 完成：内核给一页真内存

内核缺页处理大致流程：

1. 检查触发地址是否落在合法 VMA 内
   - 不在 → `SIGSEGV`（这就是"段错误"的来源）
2. 从 **伙伴系统（buddy allocator）** 拿一块物理页框（典型 4KB）
3. 把"虚拟地址 → 物理页框"的映射写进页表
4. 返回用户态，让 CPU **重新执行**触发缺页的那条指令

这次 MMU 查表成功，数据落到物理内存。一次 `malloc` + 写入的故事到此结束。

## 4. 实际怎么验证

```bash
# 看进程的虚拟内存映射（VMA）
cat /proc/$(pidof your_program)/maps

# 看"承诺占用"和"实际占用"的差距
cat /proc/$(pidof your_program)/status | grep -E "VmSize|VmRSS"
# VmSize: 虚拟地址总大小（malloc 之后就涨）
# VmRSS:  常驻物理内存（首次写入之后才涨）

# 抓系统调用，看 malloc 内部进没进内核
strace -e trace=brk,mmap,munmap ./your_program

# 抓缺页统计
ps -o min_flt,maj_flt -p $(pidof your_program)
# min_flt: 次缺页（仅分配物理页，无磁盘 IO）
# maj_flt: 主缺页（涉及磁盘 IO，比如换页、文件页加载）
```

## 5. 几个常被搞混的概念

| 容易误解 | 真相 |
|---|---|
| `malloc` 返回成功 = 分到内存 | 错。只是分到了**虚拟地址**，物理内存要等首次访问才分 |
| `top` 里 RES 大 = 内存泄漏 | 不一定。RES 包含共享库映射、Page Cache 等 |
| `free` 命令显示 free 很小 = 内存不够 | 不一定。Linux 倾向把空闲 RAM 用作 Page Cache，看 `available` 才准 |
| `mmap` 是 C 库函数 | 错。`mmap` 是系统调用，C 库的 `mmap()` 只是 wrapper |
| `free()` 就是把内存还给内核 | 不一定。多数时候只是把块还给 ptmalloc2 的 bin，并不立刻 `munmap` 给内核 |

## 6. 本专栏后续要讲什么

本篇把链路串起来，后续每篇深入一个环节：

- 第 2 篇：进程的虚拟地址空间长什么样（VMA / `mm_struct` / 进程地址布局）
- 第 3 篇：ptmalloc2 内幕（fastbin / smallbin / largebin / unsorted bin）
- 第 4 篇：tcmalloc / jemalloc 是怎么解决 ptmalloc2 痛点的
- 第 5 篇：Linux 物理内存管理（zone / 伙伴系统 / slab）
- 第 6 篇：从 Page Fault 到 OOM Killer
- 第 7 篇：Go runtime 的内存分配（mcache / mcentral / mheap / GC）

如果你希望先看哪一篇，欢迎在 GitHub Issue 留言。

{% include series-nav.html %}
