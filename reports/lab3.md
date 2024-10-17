# Lab 3 实验报告

## 编程作业

### 进程创建

`spawn` 的流程相当于 `fork` + `exec` 的结合，实现方法也极其相似：

首先像 `exec` 一样将 ELF 数据载入内存空间，而不需要像 `fork` 一样复制父进程的内存空间，通过这一步得到了新的进程的内存空间 (`memory_set`)，栈顶指针 (`user_sp`) 和程序入口地址 (`entry_point`)。

然后像 `fork` 一样分配新的进程 PID 和进程控制块，将新进程的状态设置为 `Ready`，并将其加入父进程的子进程列表。在这之后通过上面的信息设置新进程的陷入上下文(`TrapContext`)，最后将新进程加入进程队列。

### stride 调度算法

首先需要在 `TaskControlBlock` 中添加 `stride` 调度算法所需的字段，包括 `stride` 和 `pass`，优先级无需保存，只在设置的时候用于计算 `pass`。

为了使 `TaskControlBlock` 能够在 `alloc::collections::BinaryHeap` 中排序，需要实现 `Ord`，所以同时还需要实现 `Eq`， `PartialOrd` 和 `PartialEq`。需要注意的是 `BinaryHeap` 是一个大顶堆，所以需要将 `Ord` 的实现反过来。

在 `TaskManager` 中将原本的 `RoundRobin` 调度算法的就绪队列所使用的 `VecDeque` 替换为 `BinaryHeap`，并重写 `add` 和 `fetch` 方法。

最后需要在任务前加回就绪队列时候更新 `stride`，将其加上 `pass` 的值。

## 问答作业

### stride 算法深入

stride 算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride， p1.stride = 255, p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

- 实际情况是轮到 p1 执行吗？为什么？

A: 不是。因为 p1 的 stride 是 255 % 10 = 5，说明 p1 的 stride 有溢出的现象；而 p2 的 stride 是 250 % 10 = 0，说明 p2 的 stride 大概率没有溢出。因此 p1 的 stride 实际上比 p2 的 stride 大，p2 会继续执行。

我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明， 在不考虑溢出的情况下 , 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。

- 为什么？尝试简单说明（不要求严格证明）。

A: 因为 `pass` 的范围是 [1, BigStride / 2]，而 stride 算法每次都会选择 `stride` 小的进程执行，所以 `STRIDE_MAX - STRIDE_MIN` 的范围也是 [0, BigStride / 2]。

- 已知以上结论，考虑溢出的情况下，可以为 Stride 设计特别的比较器，让 BinaryHeap<Stride> 的 pop 方法能返回真正最小的 Stride。补全下列代码中的 partial_cmp 函数，假设两个 Stride 永远不会相等

A: 代码如下：

```rust
use core::cmp::Ordering;

const BIG_STRIDE: u64 = u64::MAX;

struct Stride(u64);

impl PartialOrd for Stride {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        // 检查两个stride是否都小于 BIG_STRIDE / 2
        if self.0 < BIG_STRIDE / 2 && other.0 > BIG_STRIDE / 2 {
            // self.0 小且 other.0 大，意味着 self 是更小的实际stride（考虑溢出），但是因为 BinaryHeap 是大顶堆，所以返回 Greater
            Some(Ordering::Greater)
        } else if self.0 > BIG_STRIDE / 2 && other.0 < BIG_STRIDE / 2 {
            Some(Ordering::Less)
        } else {
            other.0.partial_cmp(&self.0)
        }
    }
}

impl PartialEq for Stride {
    fn eq(&self, other: &Self) -> bool {
        false
    }
}
```

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与以下各位就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

2. 此外，我也参考了以下资料，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

- <https://nankai.gitbook.io/ucore-os-on-risc-v64/lab6/tiao-du-suan-fa-kuang-jia>：参考了其中对于 Stride 调度算法的解释，不直接涉及代码。

1. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

2. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
