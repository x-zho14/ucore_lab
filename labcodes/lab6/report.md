# LAB6 实验报告

## 练习0: 填写已有实验

需要进行改动的地方:

- proc初始化时初始化LAB6的相关变量, 包括`rq`, `run_link`和`time_slice`.
- 时钟中断的处理: 每次时钟中断都要调用`sched_proc_tick`函数(调度器的接口), 若每个`TICK_NUM`才调用一次则会因为时间片过大导致练习2的结果不准确. `sched_proc_tick`接口需要自己在`sched.c`和`sched.h`内添加以绕开`static`关键字的限制.

## 练习1: 使用Round Robin调度算法

### `sched_class`中各个函数指针的用法

- `init`: 初始化调度队列;
- `enqueue`: 将进程加入调度队列;
- `dequeue`: 将进程从调度队列中去除;
- `pick_next`: 从队列中选择下一个可以执行的进程;
- `proc_tick`: 时钟中断时使用, 用于在当前时间片结束后执行.

### Round Robin调度算法下ucore的调度执行过程

ucore通过`schedule`函数执行调度, 该函数首先调用`sched_class_enqueue`将当前进程加入调度队列, 然后调用`sched_class_pick_next`选择下一个要执行的进程, 若存在则选择, 否则下一个执行`idleproc`, 最后调用`proc_run`进行进程切换. 在时钟中断时, `trap_dispatch`执行`sched_proc_tick`(调度器在时钟中断时的操作函数), 之后执行`schedule`.

Round Robin实现了时间片轮转调度算法, 每个进程加入队列后被分配大小相同的时间片, 每次时钟中断会导致当前进程可用时间片-1, 当前进程可用时间片归零后或者当前进程结束后调度到队列中第一个可用的进程.

### 如何实现"多级反馈队列调度算法"

- 设置多个不同优先级的队列.
    优先级越高的队列, 其中的每个进程分配的时间片越小.
    如进程在当前时间片内没有完成则下降到优先级更低的队列.
- 队列内部使用时间片轮转的方法.
- 进程初始被加入优先级最高的队列, 一段时间之后CPU密集型的进程优先级会逐渐降低, IO密集型的进程因可以在较短的时间片内完成故可以保持较高的优先级.

## 练习2: 实现Stride Scheduling调度算法

### Stride Scheduling调度算法原理

该算法为调度队列中的每个进程维护一个`stride`量, 每次调度时选择`stride`最小者执行, 并增加该进程的`stride`量, 增加量与进程优先级数值成反比(优先级越高者优先级数值越大, `stride`增加量越小, 后续越有可能被更多地执行).

### 设计实现过程

- 选择`skew_heap.h`中的斜堆作为调度队列的数据结构, 可以实现$O(\log n)$的维护代价与$O(1)$的选取代价.
- 比较`stride`值时避免溢出的影响.
    `proc_struct`结构使用`uint32_t`维护`lab6_stride`, 在比较函数`proc_stride_comp_f`中将2个进程`lab6_stride`相减的结果转为有符号`int32_t`, 根据其符号判断`lab6_stride`相对大小. 这里`BIG_STRIDE`的选择应当是`int32_t`表示的最大正数. 理由如下:
    假设2个进程在$[0, +\infty)$空间中的stride值分别为$X$, $Y$, 映射到$M$位二进制数能表示的空间$[0, 2^M)$后对应的值分别为$x$, $y$. 显然`BIG_STRIDE < 2^M`, 否则会出现$x=y$但$X<Y$的情况, 不满足判断函数要求. 在此前提下, 记$d=|X-Y|$, 有$d<BIG_STRIDE$.
    以$X>Y$的情况为例, $y=unsigned(x-d)$, $x-y=d$($d\leqslant x$) or $x-y=x+2^M-(2^M-(d-x))=d$(d>x). 要同时保证$signed(x-y)=signed(d)>0$, 则要求$d\leqslant 2^{M-1}-1$. 同理可以证明$X<Y$的情况.

## 我的实现与参考答案的区别

参考答案对时钟中断的处理有误, 不会调用调度器的处理函数且时钟粒度过大, 解决方法是自行添加调用调度器处理函数的非static接口, 且在每次时钟中断时均执行该处理函数.

## 本实验中重要的知识点, 以及与OS原理中对应的知识点

- 时间片轮转调度算法(Round Robin).

## OS原理中很重要但实验没有对应的知识点

- 如何避免死锁.(实际上stride可以避免但是可能需等待很长时间)
- 多级反馈队列调度算法.