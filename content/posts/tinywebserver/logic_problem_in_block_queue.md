---
title: "修复 TinyWebServer 阻塞队列的核心并发漏洞"
date: 2026-03-13T18:00:00+08:00
draft: false
tags: ["C++", "并发编程", "开源"]
categories: ["后端开发"]
math: true
---
## 引言：什么是 TinyWebServer？

TinyWebServer 是 GitHub 上极具人气的 C++ 轻量级 Web 服务器项目。在最近的源码阅读和压力测试中，我发现其异步日志系统的底层核心 —— **`block_queue`（阻塞队列）** 在极端并发环境下存在几处严重的逻辑漏洞。

我在 Arch Linux 环境下复现了这些 Bug，并向原作者提交了我的 PR。

---

## 漏洞剖析与修复策略

### 1. 数组下标越界

在原代码中，`front()` 函数试图直接通过 `m_front` 获取队首元素。然而在初始化时 `m_front = -1`，若此时队列刚好有数据存入但尚未被 `pop` 移动索引，访问将直接指向 `m_array[-1]`。

**代码对比：**

```cpp
// Fix before: 潜在的非法内存访问
value = m_array[m_front]; 

// Fix after: 遵循循环数组索引逻辑
int index = (m_front + 1) % m_max_size;
value = m_array[index];

```

### 2. 时间精度故障

在带有超时机制的 `pop(timeout)` 中，底层调用了 `pthread_cond_timedwait`。原实现存在两个致命错误：一是将毫秒误算为微秒（差了三个数量级）；二是忽略了当前时间中的微秒进位，导致定时几乎处于“随机”状态。

**数学推导与修复：**
我们需要将当前时间 `now` 加上指定的毫秒超时 `ms`，换算为绝对纳秒：

$$
total\_ns = (now.tv\_usec \times 10^3) + (ms \pmod{1000}) \times 10^6
$$

$$
t.tv\_sec = now.tv\_sec + \lfloor ms / 1000 \rfloor + \lfloor total\_ns / 10^9 \rfloor
$$

$$
t.tv\_nsec = total\_ns \pmod{10^9}
$$

### 3. 停机 与 Segfault 竞争

这是并发编程中最难处理的边界情况：如果主线程调用了析构函数释放了 `m_array`，但此时仍有消费者线程阻塞在 `wait()` 内部（此时锁是解开的）。一旦消费者被唤醒，它会立即尝试访问已经被 `delete` 的指针，造成 **Use-After-Free** 崩溃。

**修复方案：引入 `m_close` 状态机**

1. **析构时**：加锁 -> 设 `m_close = true` -> 执行 `broadcast()` 强制唤醒所有线程 -> 清理内存。
2. **逻辑内**：所有 `pop` 线程醒来后的第一件事就是检查 `m_close`，若为真则立即安全退出。

### 4. 虚假唤醒

原代码在 `pop` 中使用 `if` 判断队列是否为空。根据 POSIX 标准，条件变量可能在没有信号的情况下被意外唤醒。

**规范化修改：**

```cpp
// 修改前
if (m_size <= 0) m_cond.wait(...);

// 修改后：严谨的循环检查
while (m_size <= 0) {
    if (m_close) return false; // 配合析构安全检查
    if (!m_cond.wait(m_mutex.get())) return false;
}

```

---

## 压力测试：

在 Arch Linux 上，我构建了 Webbench。

**测试环境**：Arch Linux (Kernel 6.19.6) / GCC 15.2.1
**测试指令**：`./webbench -c 1000 -t 10 http://127.0.0.1:9006/`

**战报数据**：

> Speed=325074 pages/min, 632049 bytes/sec.
> Requests: 54179 succeed, **0 failed**.

<img width="100%" alt="压力测试结果" src="https://github.com/user-attachments/assets/8cb06513-dd82-412b-9e49-b0f8ba2e0b37" />

在 1000 并发的高压下，服务器不仅保持了 **0 失败率**，且日志系统成功记录了超过 **135 万行** 完整数据，证明了重构后的阻塞队列在性能与稳定性之间达到了平衡。

---

**相关链接：**

* 我的 Pull Request: [GitHub Link](https://github.com/qinguoyi/TinyWebServer/pull/321)
* 项目地址: [TinyWebServer](https://github.com/qinguoyi/TinyWebServer)
