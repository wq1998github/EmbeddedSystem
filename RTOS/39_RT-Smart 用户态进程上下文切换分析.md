# 用户态进程上下文切换分析

想要深入理解用户态进程，了解上下文切换的过程是十分重要的，这一部分和底层架构紧密相关，是操作系统移植的重要部分。在 RTOS 版本中，由于应用线程与内核运行在同一个异常级别下，因此在上下文切换的时候，不用考虑线程在用户态的运行环境。

所谓的上下文切换并不神秘，想要对这一块驾轻就熟需要对当前运行硬件的体系架构非常熟悉，因为上下文切换，在硬件级别就是一组寄存器的值以及当时的 CPU 运行状态。下面会先介绍在 ARMv8 64 位下的寄存器组，然后结合相应代码讲解不同情况下的上下文切换。

## ARMv8 64 位寄存器组介绍

在 ARMv8 的 AArch64 执行状态下，EL0 - EL3 异常级别共用 31 个 64 位通用寄存器，每一个寄存器有 64 位宽，从 X0-X30，通用寄存器组如下所示：

![image-20210423152018377](figures/image-20210423152018377.png)

除了上述的通用寄存器，还有一些特殊功能寄存器组，他们分别是：

![image-20210423152111287](figures/image-20210423152111287.png)

- **Zero register**
- **Stack pointer** 
- **Program Counter**
- **Exception Link Register (ELR)**
- **Saved Process Status Register**

在上面列出的这些特殊功能寄存器中，SP、ELR、SPSR 寄存器的值在进行上下文切换时，都需要被合理的保存和恢复。

## 线程栈初始化分析

要想了解一个线程的上下文包含哪些内容，最直接的方法就是看在创建线程时，初始化线程时伪造的现场是什么样的，同样的如果需要切换到其他线程，也需要得到其他线程的这些信息，下面给出栈初始化函数：

```c
/**
 * This function will initialize thread stack.
 *
 * @param tentry the entry of thread
 * @param parameter the parameter of entry
 * @param stack_addr the beginning stack address
 * @param texit the function will be called when thread exit
 *
 * @return stack address
 */
rt_uint8_t *rt_hw_stack_init(void *tentry, void *parameter,
                             rt_uint8_t *stack_addr, void *texit)
{
    rt_ubase_t *stk;

    stk      = (rt_ubase_t *)stack_addr;

    *(--stk) = (rt_ubase_t) 0;               /* Q0 */
    *(--stk) = (rt_ubase_t) 0;               /* Q0 */
    *(--stk) = (rt_ubase_t) 0;               /* Q1 */
    *(--stk) = (rt_ubase_t) 0;               /* Q1 */
    *(--stk) = (rt_ubase_t) 0;               /* Q2 */
    *(--stk) = (rt_ubase_t) 0;               /* Q2 */
    *(--stk) = (rt_ubase_t) 0;               /* Q3 */
    *(--stk) = (rt_ubase_t) 0;               /* Q3 */
    *(--stk) = (rt_ubase_t) 0;               /* Q4 */
    *(--stk) = (rt_ubase_t) 0;               /* Q4 */
    *(--stk) = (rt_ubase_t) 0;               /* Q5 */
    *(--stk) = (rt_ubase_t) 0;               /* Q5 */
    *(--stk) = (rt_ubase_t) 0;               /* Q6 */
    *(--stk) = (rt_ubase_t) 0;               /* Q6 */
    *(--stk) = (rt_ubase_t) 0;               /* Q7 */
    *(--stk) = (rt_ubase_t) 0;               /* Q7 */
    *(--stk) = (rt_ubase_t) 0;               /* Q8 */
    *(--stk) = (rt_ubase_t) 0;               /* Q8 */
    *(--stk) = (rt_ubase_t) 0;               /* Q9 */
    *(--stk) = (rt_ubase_t) 0;               /* Q9 */
    *(--stk) = (rt_ubase_t) 0;               /* Q10 */
    *(--stk) = (rt_ubase_t) 0;               /* Q10 */
    *(--stk) = (rt_ubase_t) 0;               /* Q11 */
    *(--stk) = (rt_ubase_t) 0;               /* Q11 */
    *(--stk) = (rt_ubase_t) 0;               /* Q12 */
    *(--stk) = (rt_ubase_t) 0;               /* Q12 */
    *(--stk) = (rt_ubase_t) 0;               /* Q13 */
    *(--stk) = (rt_ubase_t) 0;               /* Q13 */
    *(--stk) = (rt_ubase_t) 0;               /* Q14 */
    *(--stk) = (rt_ubase_t) 0;               /* Q14 */
    *(--stk) = (rt_ubase_t) 0;               /* Q15 */
    *(--stk) = (rt_ubase_t) 0;               /* Q15 */

    *(--stk) = (rt_ubase_t) 1;               /* X1 */
    *(--stk) = (rt_ubase_t) parameter;       /* X0 */
    *(--stk) = (rt_ubase_t) 3;               /* X3 */
    *(--stk) = (rt_ubase_t) 2;               /* X2 */
    *(--stk) = (rt_ubase_t) 5;               /* X5 */
    *(--stk) = (rt_ubase_t) 4;               /* X4 */
    *(--stk) = (rt_ubase_t) 7;               /* X7 */
    *(--stk) = (rt_ubase_t) 6;               /* X6 */
    *(--stk) = (rt_ubase_t) 9;               /* X9 */
    *(--stk) = (rt_ubase_t) 8;               /* X8 */
    *(--stk) = (rt_ubase_t) 11;              /* X11 */
    *(--stk) = (rt_ubase_t) 10;              /* X10 */
    *(--stk) = (rt_ubase_t) 13;              /* X13 */
    *(--stk) = (rt_ubase_t) 12;              /* X12 */
    *(--stk) = (rt_ubase_t) 15;              /* X15 */
    *(--stk) = (rt_ubase_t) 14;              /* X14 */
    *(--stk) = (rt_ubase_t) 17;              /* X17 */
    *(--stk) = (rt_ubase_t) 16;              /* X16 */
    *(--stk) = (rt_ubase_t) 19;              /* X19 */
    *(--stk) = (rt_ubase_t) 18;              /* X18 */
    *(--stk) = (rt_ubase_t) 21;              /* X21 */
    *(--stk) = (rt_ubase_t) 20;              /* X20 */
    *(--stk) = (rt_ubase_t) 23;              /* X23 */
    *(--stk) = (rt_ubase_t) 22;              /* X22 */
    *(--stk) = (rt_ubase_t) 25;              /* X25 */
    *(--stk) = (rt_ubase_t) 24;              /* X24 */
    *(--stk) = (rt_ubase_t) 27;              /* X27 */
    *(--stk) = (rt_ubase_t) 26;              /* X26 */
    *(--stk) = (rt_ubase_t) 29;              /* X29 */
    *(--stk) = (rt_ubase_t) 28;              /* X28 */

    *(--stk) = (rt_ubase_t) 0;               /* FPSR 浮点状态寄存器 */
    *(--stk) = (rt_ubase_t) 0;               /* FPCR 浮点控制寄存器 */

    *(--stk) = (rt_ubase_t) texit;           /* X30 LR 寄存器 */
    *(--stk) = (rt_ubase_t) 0;               /* sp_el0 */

    *(--stk) = INITIAL_SPSR_EL1;             /* SPSR on EL1 */
    *(--stk) = (rt_ubase_t) tentry;          /* 异常返回地址 */

    return (rt_uint8_t *)stk;  /* return task's current stack address */
}
```

## 上下文切换场景

在 SMART 中可能有如下上下文切换场景：

- Syscall 过程中的上下文切换 
- 切换到系统的第一个线程
- 线程到线程切换
- 从中断切换到新的线程

下面分别分析上述四种情况下的上下文切换实现情况。

### Syscall 过程中的上下文切换

用户态应用程序调用 SVC（Supervisor Call）之后，会触发异常，进入异常处理函数，下面的代码处理了当 SVC 发生时的线程上下文保存和恢复，：

```asm
.globl  vector_exception
.type vector_exception, @function
vector_exception:
    /* 进入异常首先将当前上下文压榨，如果当前异常是 svc，那么此时就是将用户程序的上下文保存到系统栈中 */
    SAVE_CONTEXT                            /* save the context before exception */
    STP     X0, X1, [SP, #-0x10]!           /* current SP is saved in X0 */
    BL      rt_hw_trap_exception            /* handle exception */
    LDP     X0, X1, [SP], #0x10             /* load saved SP to X0 */
    MOV     SP, X0                          /* restore SP */  
    RESTORE_CONTEXT                         /* restore context */   
```

#### 保存线程上下文到系统栈

下面详细查看 `SAVE_CONTEXT` 宏是如何处理的，与线程初始化时填充栈的内容做对比，发现是完全一致的，特殊功能寄存器有：

- FPSR
- FPCR
- SPSR_EL1
- ELR_EL1

```assembly
.macro SAVE_CONTEXT
    /* Save the entire context. */
    SAVE_FPU SP                         /* 保存浮点相关寄存器 */
    STP     X0, X1, [SP, #-0x10]!
    STP     X2, X3, [SP, #-0x10]!
    STP     X4, X5, [SP, #-0x10]!
    STP     X6, X7, [SP, #-0x10]!
    STP     X8, X9, [SP, #-0x10]!
    STP     X10, X11, [SP, #-0x10]!
    STP     X12, X13, [SP, #-0x10]!
    STP     X14, X15, [SP, #-0x10]!
    STP     X16, X17, [SP, #-0x10]!
    STP     X18, X19, [SP, #-0x10]!
    STP     X20, X21, [SP, #-0x10]!
    STP     X22, X23, [SP, #-0x10]!
    STP     X24, X25, [SP, #-0x10]!
    STP     X26, X27, [SP, #-0x10]!
    STP     X28, X29, [SP, #-0x10]!
    MRS     X28, FPCR
    MRS     X29, FPSR
    STP     X28, X29, [SP, #-0x10]!
    MRS     X29, SP_EL0
    STP     X29, X30, [SP, #-0x10]!

    MRS     X3, SPSR_EL1
    MRS     X2, ELR_EL1

    STP     X2, X3, [SP, #-0x10]!

    MOV     X0, SP   /* Move SP into X0 for saving. */
.endm
```

#### 从系统栈中恢复线程上下文

上下文恢复的过程与上下文保存过程是对称的，有哪些信息被保存，哪些信息就需要被恢复。不过在上下文恢复的过程中，还是多了一些特别的内容：

- 检查当前进程是否要退出，如果该系统调用会使得进程退出，那么就调用进程退出相关函数后重新执行调度，不再执行后续上下文恢复过程
- 检查是否需要进行异常级别切换，根据 SPSR_EL1 中相关的状态位判断是否需要返回 EL1 级别，这里很容易理解，如果异常发生时系统处于 EL1，那就不用返回 EL0，如果异常发生时系统处于 EL0，那么后续就需要返回 EL0 用户态
- 根据上一步的判断结果，决定是否执行返回用户态的函数

线程上下文切换的功能实现如下所示：

```assembly
.macro RESTORE_CONTEXT
    BL      lwp_check_exit           /* doesn't return if the process needs to exit  */

    LDP     X2, X3, [SP], #0x10      /* load SPSR and ELR to x2 x3 */

    TST     X3, #0x1f                /* check if need to return to user */
    MSR     SPSR_EL1, X3             /* restore SPSR_EL1 */
    MSR     ELR_EL1, X2              /* restore ELR_EL1 */

    LDP     X29, X30, [SP], #0x10    /* restore SP_EL0 and LR register  */
    MSR     SP_EL0, X29
    LDP     X28, X29, [SP], #0x10    /* restore FPCR and FPSR register  */
    MSR     FPCR, X28
    MSR     FPSR, X29
    LDP     X28, X29, [SP], #0x10    /* restore other general-purpose registers  */
    LDP     X26, X27, [SP], #0x10
    LDP     X24, X25, [SP], #0x10
    LDP     X22, X23, [SP], #0x10
    LDP     X20, X21, [SP], #0x10
    LDP     X18, X19, [SP], #0x10
    LDP     X16, X17, [SP], #0x10
    LDP     X14, X15, [SP], #0x10
    LDP     X12, X13, [SP], #0x10
    LDP     X10, X11, [SP], #0x10
    LDP     X8, X9, [SP], #0x10
    LDP     X6, X7, [SP], #0x10
    LDP     X4, X5, [SP], #0x10
    LDP     X2, X3, [SP], #0x10
    LDP     X0, X1, [SP], #0x10
    RESTORE_FPU SP                    /* restore registers about FPU */

    BEQ     ret_to_user               /* return to user */

    ERET                              /* return but not to user */
.endm
```

下面展示一下返回用户态功能是如何实现的：

```assembly
.global ret_to_user
.type ret_to_user, @function
ret_to_user:
    SAVE_FPU sp
    stp x0, x1, [sp, #-0x10]!
    stp x2, x3, [sp, #-0x10]!
    stp x4, x5, [sp, #-0x10]!
    stp x6, x7, [sp, #-0x10]!
    stp x8, x9, [sp, #-0x10]!
    mrs x0, fpcr
    mrs x1, fpsr
    stp x0, x1, [sp, #-0x10]!
    stp x29, x30, [sp, #-0x10]!

    bl lwp_signal_check
    cmp x0, xzr

    ldp x29, x30, [sp], #0x10
    ldp x0, x1, [sp], #0x10
    msr fpcr, x0
    msr fpsr, x1
    ldp x8, x9, [sp], #0x10
    ldp x6, x7, [sp], #0x10
    ldp x4, x5, [sp], #0x10
    ldp x2, x3, [sp], #0x10
    ldp x0, x1, [sp], #0x10
    RESTORE_FPU sp

    bne user_do_signal
    eret
```

可以看到在返回用户态前，执行了对 signal 的检查，如果 lwp_signal_check 的返回值不为 0，则调用 user_do_signal 函数执行相应的信号动作，执行完毕后再返回用户态，否则直接返回用户态。

### 切换到系统的第一个线程

`rt_hw_context_switch_to` 函数在系统中只被调用一次，在系统调度器启动时，调度器会查询系统中当前最高优先级的线程，然后跳转到该线程执行，该函数的实现如下所示：

```assembly
/*
 * void rt_hw_context_switch_to(rt_uint32 to, struct rt_thread *to_thread);
 * X0 --> to (thread stack)
 * X1 --> to_thread
 */
.globl rt_hw_context_switch_to
.type rt_hw_context_switch_to, @function
rt_hw_context_switch_to:
    LDR     X0, [X0]
    MOV     SP, X0
    MOV     X0, X1
    BL      rt_cpus_lock_status_restore
    BL      rt_thread_self
    BL      lwp_user_setting_restore
    B       rt_hw_context_switch_exit
```

`rt_hw_context_switch_exit` 函数的实现如下所示：

```assembly
.global rt_hw_context_switch_exit
rt_hw_context_switch_exit:
    RESTORE_CONTEXT
```

这个函数还是比较简单的，直接恢复目标线程的上下文，然后切换到该线程运行即可，RESTORE_CONTEXT 宏在上面已经有了详细介绍。

### 线程到线程上下文切换

首先查看线程到线程切换函数 `rt_hw_context_switch` ，该函数被 rt_schedule() 调用，实现如下：

```assembly
/*
 * void rt_hw_context_switch(rt_uint32 from, rt_uint32 to, struct rt_thread *to_thread);
 * X0 --> from (from_thread stack)
 * X1 --> to (to_thread stack)
 * X2 --> to_thread
 */
.globl rt_hw_context_switch
.type rt_hw_context_switch, @function
rt_hw_context_switch:
    SAVE_CONTEXT_FROM_EL1               /* save current thread context in EL1 */
    MOV    X3, SP                       /* save sp after save context */
    STR    X3, [X0]                     /* update sp to from_thread's TCB */
    LDR    X0, [X1]                     /* load to_thread task's sp to X0 */
    MOV    SP, X0                       /* set sp to to_thread's sp  */
    MOV    X0, X2                       /* set X0 to to_thread's TCB */
    BL     rt_cpus_lock_status_restore  /* restores the previous lock-holding state of the target thread */
    BL     rt_thread_self               /* gets the thread running on the current CPU */
    BL     lwp_user_setting_restore     /* restore the info about TLS in tpidr_el0 register */
    B      rt_hw_context_switch_exit    /* begin to switch to the next thread */
```

该函数一定是在内核态执行的，其功能是先保存上下文到 from_thread 线程栈中，然后再切换到目标线程的上下文。

#### 从中断切换到新线程

先看一下中断发生后系统执行了哪些操作：

```assembly
.globl vector_irq
.type vector_irq, @function
vector_irq:
    CLREX
    SAVE_CONTEXT
    STP     X0, X1, [SP, #-0x10]!   /* X0 is thread sp */

    BL      rt_interrupt_enter
    BL      rt_hw_trap_irq
    BL      rt_interrupt_leave

    LDP     X0, X1, [SP], #0x10
    BL      rt_scheduler_do_irq_switch
    B       rt_hw_context_switch_exit
```

中断发生时，系统可能处于用户态或者内核态，无论是哪种情况，当前上下文都会被保存到线程栈中，也就是说进入 IRQ 时被打断的线程上下文就已经保存好了。

在中断处理完成后，如果发现在中断后需要执行一次调度，那么就会调用 `rt_scheduler_do_irq_switch` 函数来完成该功能，最终会调用到 `rt_hw_context_switch_interrupt`  ，该函数实现如下：

```assembly
/*
 * void rt_hw_context_switch_interrupt(context, from sp, to sp, tp tcb)
 * X0 :interrupt context
 * X1 :addr of from_thread's sp
 * X2 :addr of to_thread's sp
 * X3 :to_thread's tcb
 */
.globl rt_hw_context_switch_interrupt
.type rt_hw_context_switch_interrupt, @function
rt_hw_context_switch_interrupt:
    STP     X0, X1, [SP, #-0x10]!
    STP     X2, X3, [SP, #-0x10]!
    STP     X29, X30, [SP, #-0x10]!
    BL      rt_thread_self
    BL      lwp_user_setting_save
    LDP     X29, X30, [SP], #0x10
    LDP     X2, X3, [SP], #0x10
    LDP     X0, X1, [SP], #0x10
    STR     X0, [X1]                            /* update from_thread's sp */
    LDR     X0, [X2]                            /* get to_thread's sp */
    MOV     SP, X0
    MOV     X0, X3                              /* get to_thread's tcb */
    MOV     X19, X0
    BL      rt_cpus_lock_status_restore         /* restores the previous lock-holding state of the target thread */
    MOV     X0, X19
    BL      lwp_user_setting_restore            /* restore the info about TLS in tpidr_el0 register */
    B       rt_hw_context_switch_exit           /* begin to switch to the next thread */
```

该函数执行一些辅助操作后就切换到下一个线程运行，不需要做上下文保存操作，因为在进入中断异常时，已经完成了上下文保存。

## 用户态线程与内核态线程

在 smart 中采用了用户态线程与内核态线程一对一模型，也就是说当创建一个用户态线程时，同时在内核态有一个线程与之对应。这两个线程拥有各自独立的栈，也有各自独立的代码段。

线程切换过程都是在内核态进行的，内核线程会在合适的时候切换到用户态，执行用户态的代码段。当用户态线程执行系统调用操作返回内核时，则由与之对应的内核态线程执行相应的操作。

切换到用户态线程后，内核态线程的 text 就不再继续执行了，只有当用户态线程调用 syscall 时才会切换回内核态，利用内核态线程的栈来执行 syscall  指令，然后在合适的时候返回用户态。

## 总结

上下文切换的重点内容为 `SAVE_CONTEXT`  和 `RESTORE_CONTEXT`，这两个宏执行了架构级别的上下文保存与恢复，主要是通用寄存器组和一些特殊功能寄存器。

想要完整理解 smart 中的上下文切换，还需要理解用户态线程与内核态线程之间的关系，如何从用户态线程切换到内核态线程，以及如何从内核态线程返回到用户态线程。

对于上下文切换这一在内核移植中重要的主题，很多初学者都很感兴趣但是也充满了疑惑，我认为如果能提升对芯片体系架构的理解，那么搞定上下文切换就一点也不难了。