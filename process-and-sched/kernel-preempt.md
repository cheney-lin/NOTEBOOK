  ### Linux下的内核抢占
https://www.cnblogs.com/ck1020/p/6497963.html
        很遗憾之前在介绍进程调度的文章中，虽然涉及到了内核抢占，但是却没有对其进行深入介绍，今天就稍微总结下内核抢占。
        内核抢占在一定程度上减少了对某种事件的响应延迟，这也是内核抢占被引入的目的。之前的内核中，除了显示调用系统调度器的某些点，内核其他地方是不允许中重新调度的，如果内核在做一些比较复杂的工作，就会造成某些急于处理的事得不到及时响应。针对内核抢占其实本质上也是对当前进程而言（不知道这么描述是否合适），因为内核是为用户程序提供服务，换言之，其本身不会主动的去执行某个动作。这里内核抢占，重点在于用户程序请求内核服务时，CPU切换到内核态在执行某个系统服务期间，被抢占。

　　当然，即使支持内核抢占，也不是什么时候都可以的，还是要考虑对临界区的保护。类似于多处理器架构，如果进程A陷入到内核模式访问某个临界资源，而在访问期间，进程B也要访问临界区，如果这种抢占被允许，那么就发生了临界区被重入。所以，在访问临界资源时需要**禁止内核抢占**，在出临界区则要开启内核抢占。

　　为了支持内核抢占，在进程结构体的thread_info结构中有个preempt_count字段，用以记录当前内核（活动）是否可以被抢占。当该值为0时，允许被抢占；否则，不允许。

　　关于内核抢占有几个函数：
``` c
#define preempt_disable() \
do { \
    inc_preempt_count(); \
    barrier(); \
} while (0)


#define preempt_enable() \
do { \
    preempt_enable_no_resched(); \
    barrier(); \
    preempt_check_resched(); \
} while (0)
```
``` c
#define inc_preempt_count() add_preempt_count(1)
#define dec_preempt_count() sub_preempt_count(1)


# define add_preempt_count(val)    do { preempt_count() += (val); } while (0)
# define sub_preempt_count(val)    do { preempt_count() -= (val); } while (0)
```

上面两个函数是禁止和启用内核抢占。其实就是一个宏定义。禁止内核抢占本质上就是把当前进程的thread_info结构中的preempt_count字段加1，而启用内核抢占就是减1.注意这里启用内核抢占之后，调用了preempt_check_resched检查当前是否需要重新调度，这也是一个宏，实现如下：
``` c
#define preempt_check_resched() \
do { \
    if (unlikely(test_thread_flag(TIF_NEED_RESCHED))) \
        preempt_schedule(); \
} while (0)
```

 就是在开启内核抢占之后，检查下此时是否有比较重要的进程等待执行，如果有，则调用preempt_schedule函数执行调度，可见在显示开启内核抢占之后，是触发内核抢占的一个时机。而调度过程本身是不允许被抢占的。preempt_schedule函数如下
``` c
asmlinkage void __sched notrace preempt_schedule(void)
{
    struct thread_info *ti = current_thread_info();

    /*
     * If there is a non-zero preempt_count or interrupts are disabled,
     * we do not want to preempt the current task. Just return..
     */
    if (likely(ti->preempt_count || irqs_disabled()))
        return;

    do {
        add_preempt_count_notrace(PREEMPT_ACTIVE);
        __schedule();
        sub_preempt_count_notrace(PREEMPT_ACTIVE);

        /*
         * Check again in case we missed a preemption opportunity
         * between schedule and now.
         */
        barrier();
    } while (need_resched());
}
```
如果抢占计数器不为0或者当前处于关闭硬件中断状态均是不可以被抢占的。

前面提到，在不支持内核抢占的内核中，内核态程序会一直执行，直到返回用户空间进程时，才会检查调度。在内核中执行期间，能打断当前执行的，只有中断，在硬件中断来了之后，处理器会根据情况着手处理硬件中断，在硬件中断处理完毕需要恢复现场时，若检查之前的状态是内核态，则不触发调度，只有之前状态是用户态时才会触发调度。而在支持内核抢占的内核中，在从硬件中断返回时，不管是返回用户态和内核态都会检查调度，若是返回内核态，检查当前线程调度标识和抢占标识，若都允许，则可进程调度。这是内核抢占下由增加的一个调度点。
  