# Lab 1 实验报告

## 编程作业

### 获取任务信息

首先观察系统调用的签名为
```rust
fn sys_task_info(ti: *mut TaskInfo) -> isize
```
其中的 `ti` 很明显是一个指向 `TaskInfo` 结构体的指针。 为了方便开发我将 `TaskInfo` 中所有字段更改为公开字段(`pub`)。

对于对系统调用次数的记录，遵循实验说明中的桶计数并将其加入 `TaskControlBlock` 中，并为 `TaskControlBlock` 实现 `record_syscall_times` 方法，并且在每次系统调用时先调用该方法。

而对于运行时间的记录，在首次运行时设置 `TaskControlBlock` 的 `start_time` 为当前时间，因此在每次调用 `sys_task_info` 时可以计算出运行时间。

最后为 `TaskControlBlock` 实现 `get_taskinfo` 方法，获取当前任务的信息。

与 `sys_get_time` 类似，最后将该 `TaskInfo` 结构体的内容拷贝到 `ti` 指向的内存中。

关于桶计数的问题，可以通过哈希表的方式实现，避免内存浪费。

## 简答作业

+ Q: 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 三个 bad 测例 (ch2b_bad_*.rs) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。
  A:  ch2b_bad_address.rs 试图在 U 态写入 0x0 地址，触发了页错误 (`Trap::Exception(Exception::StoreFault)`)，程序退出。 ch2b_bad_instr.rs 试图在 U 态执行 `sret` 指令，触发了非法指令错误 (`Trap::Exception(Exception::IllegalInstruction)`)，程序退出。 ch2b_bad_trap.rs 试图执行 `ebreak` 指令，触发了非法指令错误，程序退出。 `ch2b_bad_register.rs` 试图在 U 态下读取 sstatus CSR 寄存器，触发了非法指令错误，程序退出。 我使用的 RustSBI 版本为 0.2.0-alpha.6。

+ Q: 深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:
  1. L40：刚进入 __restore 时，a0 代表了什么值。请指出 __restore 的两种使用情景。
    A: 陷入(Trap)处理完毕后：a0 指向内核栈的栈指针，也就是之前 __alltraps (L37) 中保存的Trap 上下文。
    任务切换(__switch)后 (即 ret 后)：a0 指向前一个任务的任务上下文(TaskContext)。其在切换之前被传入 __switch 函数。
  2. L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。
    ```asm
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    ```
    A: 这几行汇编代码特殊处理了 sstatus, sepc 和 sscratch 寄存器，从栈中加载了这三个寄存器的值。这三个寄存器的值对于进入用户态有以下意义：sssatus 寄存器中的 `SPP` 位设置为 0，表示进入用户态；sepc 寄存器中保存了用户态程序的下一条指令地址(pc)；sscratch 寄存器中保存了用户态栈的栈顶地址(sp)。
  3. L50-L56：为何跳过了 x2 和 x4？
    ```asm
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    ```
    A: 因为 x2 是 sp 寄存器，后续需要通过它来将其他寄存器的值保存到正确的位置。x4 是 tp 寄存器，除非特殊情况手动使用，一般不会用到。（在实验指导书中有说明）
  4. L60：该指令之后，sp 和 sscratch 中的值分别有什么意义？
    ```asm
    csrrw sp, sscratch, sp
    ```
    A: 该指令将 sscratch 与 sp 互换。__restore 中，这之后 sp 指向用户栈顶，sscratch 指向内核栈顶。互换这两个寄存器的值是为了在内核态和用户态之间切换时，能够正确地使用栈。
  5. __restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？
    A: 在 `sret` 指令执行之后会进入用户态。因为 `sret` 指令会将 `sstatus` 寄存器中的 `SPP` 位设置为 0，表示进入用户态。
  6. L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？
    ```asm
    csrrw sp, sscratch, sp
    ```
    A: 该指令将 sscratch 与 sp 互换。__alltraps 中，这之后 sp 指向内核栈顶，sscratch 指向用户栈顶。与题 4 类似，互换这两个寄存器的值是为了在内核态和用户态之间切换时，能够正确地使用栈。
  7. 从 U 态进入 S 态是哪一条指令发生的？
    A: 在用户态执行 `ecall` 时发生的。

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与以下各位就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

助教-文泰龙：讨论无法启动 RustSBi 的问题，最终确定为 QEMU 版本的差异问题，与项目代码无关。

2. 此外，我也参考了以下资料，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：


3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
