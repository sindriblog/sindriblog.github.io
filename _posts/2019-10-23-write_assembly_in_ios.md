---
title: iOS中的内嵌汇编
date: 2019-10-23 08:00
tags:
- 开发笔记
---

写一篇在`iOS`上使用汇编的文章的想法在脑袋里面停留了很久了，但是迟迟没有动手。虽然早前在做启动耗时优化的工作中，也做过通过拦截`objc_msgSend`并插入汇编指令来统计方法调用耗时的工作，但也只仅此而已。刚好最近的时间项目在做安全加固，需要写更多的汇编来提高安全性（**文章内汇编使用指令集为ARM64**），也就有了本文

## 内嵌汇编格式
    __asm__ [关键词]( 
        指令
        : [输出操作数列表]
        : [输入操作数列表]
        : [被污染的寄存器列表]
    );
    
比如函数中存在`a、b、c`三个变量，要实现`a = b + c`这句代码，汇编代码如下：

    __asm__ volatile(
        "mov x0, %[b]\n"
        "mov x1, %[c]\n"
        "add x2, x0, x1\n"
        "mov %[a], x2\n"
        : [a]"=r"(a)
        : [b]"r"(b), [c]"r"(c)
    );
    
### volatile
`volatile`关键字表示禁止编译器对汇编代码进行再优化，但基本上有没有声明编译后指令都没区别

### 操作数
操作数格式为`"[limits]constraint"`，分为权限和限定符两部分。比如`"=r"`表示参数是只写并存放在通用寄存器上

- `limits`
    
    | 关键字 | 表意 |
    | --- | --- |
    | = | 只写，通用用于输出操作数 |
    | + | 读写，只能用于输出操作数 |
    | & | 声明寄存器只能用于输出 |

- `constraint`

    | 关键字 | 表意 |
    | --- | --- |
    | f | 浮点寄存器f0~f7 |
    | G/H | 浮点常量立即数 |
    | I/L/K | 数据处理用到的立即数 |
    | J | 值为-4095~4095的索引 |
    | l/r | 寄存器r0~r15 |
    | M | 0~32/2的幂次方的常量 |
    | m | 内存地址 |
    | w | 向量寄存器s0~s31 |
    | X | 任何类型的操作数 |

### 指令
由于`ARM64`的指令过多，可通过文末的扩展阅读查阅指令，这里只讲解指令中的一些关键字：

- `%0~%N` / `%[param]`

    在使用`C`代码和汇编混编的情况下，`%`起头用来关联参数，通过`%[param]`可以声明参数名称，也可以使用匿名参数格式`%N`的方式顺序对应参数（`abc`参数会按照`012`的顺序匹配）：
    
        __asm__ volatile(
            "mov x0, %1\n"
            "mov x1, %2\n"
            "add x2, x0, x1\n"
            "mov %0, x2\n"
            : "=r"(a)
            : "r"(b), "r"(c)
        );
        
    在实操过程中，设备不一定支持`%N`的匿名参数格式，建议使用`%[param]`使可读性更强
        
- `[reg]`

    程序运行的多数情况下，寄存器内存储的是存放数据的地址，使用`[]`包裹住寄存器，表示将寄存器的存储值作为地址访问数据。下面的指令分别是取出地址`0x10086`存储的数据存放在`x1`寄存器上，然后存放到地址`0x100086`的内存中：
    
        "mov x0, #0x10086\n"
        "mov x1, [x0]\n"
        "mov x2, #0x100086\n"
        "str x1, [x2]\n"
        
- `#1` / `#0x1`

    使用`#`起头表示立即数（常数），建议使用`16进制`书写

### 调用规范
`ARM64`调用约定采用`AAPCS64`，参数从左到右存放到`x0~x7`寄存器中，参数超出`8`个时，多余的从右往左入栈，根据返回值大小不同存放在`x0/x8`返回。寄存器规则如下：


| 寄存器 | 特殊名称 | 规则 |
| --- | --- | --- |
| r31 | SP | 存放栈顶地址 |
| r30 | LR | 存放函数返回地址 |
| r29 | FP | 存放函数使用栈帧地址 |
| r19~r28 |  | 被调用方需要保护的寄存器 |
| r18 |  | 平台寄存器，不建议当做临时寄存器使用 |
| r17 | IP1 | 进程内使用寄存器，不建议当做临时寄存器使用 |
| r16 | IP0 | 同r17，同时作为软中断`svc`中的系统调用参数 |
| r9~r15 |  | 临时寄存器（汇编指令中嵌入函数地址参数时，会用于保存函数地址） |
| r8 |  | 返回值寄存器（其他时候同r9~r15） |
| r0~r7 |  | 传递存储调用参数，r0可作为返回值寄存器 |
| NZCV |  | 状态寄存器 |

## 实战

### 调试检测
在`iOS`应用安全加固中，通过`sysctl + kinfo_proc`的方案可以检测应用是否被调试：

    __attribute__((__always_inline)) bool checkTracing() {
        size_t size = sizeof(struct kinfo_proc);
        struct kinfo_proc proc;
        memset(&proc, 0, size);
        
        int name[4];
        name[0] = CTL_KERN;
        name[1] = KERN_PROC;
        name[2] = KERN_PROC_PID;
        name[3] = getpid();
        
        sysctl(name, 4, &proc, &size, NULL, 0);
        return proc.kp_proc.p_flag & P_TRACED;
    }
    
但由于`fishhook`这种直接修改懒符号地址的方案存在，直接使用`sysctl`是不安全的，因此多数开发者会将这一调用替换成内嵌汇编的方案执行：

    size_t size = sizeof(struct kinfo_proc);
    struct kinfo_proc proc;
    memset(&proc, 0, size);
    
    int name[4];
    name[0] = CTL_KERN;
    name[1] = KERN_PROC;
    name[2] = KERN_PROC_PID;
    name[3] = getpid();
    
    __asm__(
        "mov x0, %[name_ptr]\n"
        "mov x1, #4\n"
        "mov x2, %[proc_ptr]\n"
        "mov x3, %[size_ptr]\n"
        "mov x4, #0x0\n"
        "mov x5, #0x0\n"
        "mov w16, #202\n"
        "svc #0x80\n"
        :
        :[name_ptr]"r"(&name), [proc_ptr]"r"(&proc), [size_ptr]"r"(&size)
    );
    
    return proc.kp_proc.p_flag & P_TRACED;
    
### 踩坑
使用`C`代码内嵌汇编开发的时候，有个致命的问题是函数入口会将临时变量入栈，并且将这些变量存放到寄存器中。上面的混编代码实际运行时，会出现下面的情况：

    // 函数入口生成的临时变量代码
    add x0, sp, #0x24       // x0存放name
    add x1, sp, #0x34       // x1存放proc
    add x2, sp, #020        // x2存放size
    
    ......
    
    // 内嵌汇编
    mov x0, x0              // name正常赋值
    mov x1, #4              // proc数据被破坏
    mov x2, x1              // size数据被破坏
    mov x3, x2
    mov x4, #0x0
    mov x5, #0x0
    mov x12, #0xca
    svc #0x80
    
编译后的代码由于临时变量顺序问题，导致了`svc`中断调用`sysctl`无法传入正确参数，最终卡死应用

## 修复
### 插入临时变量
通过编译后的指令得到一张对应表：


| 变量 | 寄存器 | 入参寄存器 |
| --- | --- | --- |
| name | x0 | x0 |
| proc | x1 | x2 |
| size | x2 | X3 |

如果能够让存储临时变量的寄存器和`svc`中断时的入参寄存器保持一致，就不会遭到破坏

> `ARM64`调用约定，参数从右往左入栈

因为检测函数无入参，所以临时参数入参后依次存放到了`x0~x2`寄存器中，顺序为`name、proc、size`，因此需要只需要在`name`和`proc`中插入一个无用的临时变量，就能让参数对应起来：

    size_t size = sizeof(struct kinfo_proc);
    struct kinfo_proc proc;
    memset(&proc, 0, size);
    
    int placeholder;
    int name[4];
    name[0] = CTL_KERN;
    name[1] = KERN_PROC;
    name[2] = KERN_PROC_PID;
    name[3] = getpid();
    
编译后指令变为：


    // 函数入口生成的临时变量代码
    add x0, sp, #0x24       // x0存放name
    add x1, sp, #0x34       // x1存放placeholder
    add x2, sp, 0x38        // x2存放proc
    add x3, sp, #020        // x3存放size
    
    ......
    
    // 内嵌汇编
    mov x0, x0           
    mov x1, #4           
    mov x2, x2             
    mov x3, x3
    mov x4, #0x0
    mov x5, #0x0
    mov x12, #0xca
    svc #0x80
    
### 修改指令顺序
设置入参的指令会破坏寄存器上已有的值，那么保证设置入参之前，寄存器没被破坏就可以了：

    __asm__(
        "mov x0, %[name_ptr]\n"
        "mov x3, %[size_ptr]\n"
        "mov x2, %[proc_ptr]\n"
        "mov x1, #4\n"
        "mov x4, #0x0\n"
        "mov x5, #0x0\n"
        "mov w16, #202\n"
        "svc #0x80\n"
        :
        :[name_ptr]"r"(&name), [proc_ptr]"r"(&proc), [size_ptr]"r"(&size)
    );
    
编译后指令如下：
    
    // 内嵌汇编
    mov x0, x0              // x0保存name
    mov x3, x2              // x3保存size
    mov x2, x1              // x2保存proc
    mov x1, #4
    mov x4, #0x0
    mov x5, #0x0
    mov x12, #0xca
    svc #0x80

### 全汇编实现
在和`C`代码混编的情况下，无法保证哪些寄存器会被破坏，那么直接使用汇编实现整个逻辑是一个不错的选择，需要注意`2`个问题：

1. 保证函数调用前后不会生成出入口指令，使用`__attribute__((naked))`来处理
2. 所有变量存储在栈上，需要把控制好栈的使用
3. 使用安全的寄存器（`r19~r28`）

首先先判断需要多长的栈空间，根据函数`sysctl(name, 4, &proc, &size, NULL, 0)`判断

- 参数`name`总共占用 `4 * int`空间，记为`0x10`
- 参数`proc`在`arm64`下，`sizof()`计算长度为`0x288`
- 参数`&size`指针长度为`0x8`
- 共计`0x2a0`

函数入口时，需要对`FP/LR`寄存器进行入栈，保证函数能正确退出。另外`r19~r28`共计`10`个寄存器需要进行入栈保护，最终得出函数运行时的栈空间图：

    ---------- 
    |   FP   |
    ----------  sp + 0x2f8
    |   LR   |
    ----------  sp + 0x2f0
    |   r20  |
    ----------  sp + 0x2e8
    |   r19  |
    ----------  sp + 0x2e0
    |   r22  |
    ----------  sp + 0x2d8
    |   r21  |
    ----------  sp + 0x2d0
    |   r24  |
    ----------  sp + 0x2c8
    |   r23  |
    ----------  sp + 0x2c0
    |   r26  |
    ----------  sp + 0x2b8
    |   r25  |
    ----------  sp + 0x2b0
    |   r28  |
    ----------  sp + 0x2a8
    |   r27  |
    ----------  sp + 0x2a0
    | p_size |
    ----------  sp + 0x298
    |  proc  |
    ----------  sp + 0x10
    |  name  |  
    ----------  sp
    
在保存`r19~r28`寄存器入栈后，使用其中五个寄存器来保存一些参数：

    ------------------
    |   参数  | 寄存器 |
    ------------------  
    |  name  |  r19  |
    ------------------   
    |  proc  |  r20  |
    ------------------  
    | p_size |  r21  |
    ------------------  
    |  size  |  r22  |
    ------------------  
    |   sp   |  r23  |
    ------------------  
    |  temp  |  r24  |
    ------------------ 
    
确认好栈上空间的使用后，可以开始分步骤实现：

#### 函数出入口
在函数的出入口负责两件事情：`FP/LR`的出入栈、`r19~r28`的出入栈

    __asm__ volatile(
        "stp x29, x30, [sp, #-0x10]!\n"
        "stp x19, x20, [sp, #-0x10]!\n"
        "stp x21, x22, [sp, #-0x10]!\n"
        "stp x23, x24, [sp, #-0x10]!\n"
        "stp x25, x26, [sp, #-0x10]!\n"
        "stp x27, x28, [sp, #-0x10]!\n"
        
        ......
        
        "ldp x19, x20, [sp], #0x10\n"
        "ldp x21, x22, [sp], #0x10\n"
        "ldp x23, x24, [sp], #0x10\n"
        "ldp x25, x26, [sp], #0x10\n"
        "ldp x27, x28, [sp], #0x10\n"
        "ldp x29, x30, [sp], #0x10\n"
    );

#### 栈开辟空间
临时变量总共用到`0x2a0`的空间，并且需要使用`5`个寄存器保存变量

    __asm__ volatile(
        ......
        "sub sp, sp, #0x2a0\n"
        
        // 开辟栈空间，寄存器保存变量
        "mov x19, sp\n"             // x19 = name
        "add, x20, sp, #0x10\n"     // x20 = proc
        "add, x21, sp, #0x298\n"    // x21 = p_size
        "mov x22, #0x288\n"         // x22 = size
        "mov x23, sp\n"             // x23 = sp
        "str x22, [x21]\n"          // p_size = &size
        
        "add sp, sp, #0x2a0\n"
        ......
    );
    
#### kinfo_proc
确定`proc`的内存之后，需要将：

    size_t size = sizeof(struct kinfo_proc);
    struct kinfo_proc proc;
    memset(&proc, 0, size);
    
转换成对应的汇编，其中`proc`存储在`x20`，`x22`存储了`size`，`memset`一共需要三个参数，分别入参：

    __asm__ volatile(
        ......
        
        "mov x24, %[memset_ptr]\n"
        "mov x0, x20\n"
        "mov x1, #0x0\n"
        "mov x2, x12\n"
        "blr x24\n"
        
        ......
        :
        :[memset_ptr]"r"(memset)
    );
    
#### name
由于`name`是`int`数组，在明确其存储位置的情况下，需要分别将`4`个`4字节`的参数存储到对应的内存位置，其位置分布如下：

    -------------
    |  name[3]  |  
    -------------  sp + 0xc
    |  name[2]  |  
    -------------  sp + 0x8
    |  name[1]  |  
    -------------  sp + 0x4
    |  name[0]  |  
    -------------  sp
    
另外`name`需要使用到`getpid()`来配置参数，通过`svc`的中断可以获取这一参数（`svc`系统调用参数可以参考扩展阅读中的`Kernel Syscalls`）

    #define CTL_KERN        1
    #define KERN_PROC       14
    #define KERN_PROC_PID   1

    __asm__ volatile(
        ......
        
        // getpid
        "mov x0, #0\n"
        "mov w16, #20\n"
        "mov x3, x0\n"          // name[3]=getpid()
    
        // 设置参数并存储
        "mov x0, #0x1\n"
        "mov x1, #0xe\n"
        "mov x2, #0x1\n"
        "str w0, [x23, 0x0]\n"
        "str w1, [x23, 0x4]\n"
        "str w2, [x23, 0x8]\n"
        "str w3, [x23, 0xc]\n"
        
        ......
    );
    
#### sysctl
最后是调用`sysctl`，根据参数和寄存器对应关系入参调用即可：

    __asm__ volatile(
        ......
    
        "mov x0, x19\n"
        "mov x1, #0x4\n"
        "mov x2, x20\n"
        "mov x3, x21\n"
        "mov x4, #0x0\n"
        "mov x5, #0x0\n"
        "mov w16, #202\n"
        "svc #0x80\n"
                
        ......
    );
    
#### flag检测
最终需要返回`p_flag`和`P_TRACED`的与比较检测，这里需要通过获取`p_flag`在结构体中的偏移来访问数据，`struct extern_proc`的结构如下：

    struct extern_proc {
        union {
            struct {
                struct  proc *__p_forw; /* Doubly-linked run/sleep queue. */
                struct  proc *__p_back;
            } p_st1;
            struct timeval __p_starttime;   /* process start time */
        } p_un;
        
        #define p_forw p_un.p_st1.__p_forw
        #define p_back p_un.p_st1.__p_back
        #define p_starttime p_un.__p_starttime
        
        struct  vmspace *p_vmspace;     /* Address space. */
        struct  sigacts *p_sigacts;     /* Signal actions, state (PROC ONLY). */
        int     p_flag;                 /* P_* flags. */
        char    p_stat;                 /* S* process status. */
        pid_t   p_pid;                  /* Process identifier. */
        pid_t   p_oppid;         /* Save parent pid during ptrace. XXX */
        int     p_dupfd;         /* Sideways return value from fdopen. XXX */
        /* Mach related  */
        caddr_t user_stack;     /* where user stack was allocated */
        void    *exit_thread;   /* XXX Which thread is exiting? */
        int             p_debugger;             /* allow to debug */
        boolean_t       sigwait;        /* indication to suspend */
        /* scheduling */
        u_int   p_estcpu;        /* Time averaged value of p_cpticks. */
        int     p_cpticks;       /* Ticks of cpu time. */
        fixpt_t p_pctcpu;        /* %cpu for this process during p_swtime */
        void    *p_wchan;        /* Sleep address. */
        char    *p_wmesg;        /* Reason for sleep. */
        u_int   p_swtime;        /* Time swapped in or out. */
        u_int   p_slptime;       /* Time since last blocked. */
        struct  itimerval p_realtimer;  /* Alarm timer. */
        struct  timeval p_rtime;        /* Real time. */
        u_quad_t p_uticks;              /* Statclock hits in user mode. */
        u_quad_t p_sticks;              /* Statclock hits in system mode. */
        u_quad_t p_iticks;              /* Statclock hits processing intr. */
        int     p_traceflag;            /* Kernel trace points. */
        struct  vnode *p_tracep;        /* Trace to vnode. */
        int     p_siglist;              /* DEPRECATED. */
        struct  vnode *p_textvp;        /* Vnode of executable. */
        int     p_holdcnt;              /* If non-zero, don't swap. */
        sigset_t p_sigmask;     /* DEPRECATED. */
        sigset_t p_sigignore;   /* Signals being ignored. */
        sigset_t p_sigcatch;    /* Signals being caught by user. */
        u_char  p_priority;     /* Process priority. */
        u_char  p_usrpri;       /* User-priority based on p_cpu and p_nice. */
        char    p_nice;         /* Process "nice" value. */
        char    p_comm[MAXCOMLEN + 1];
        struct  pgrp *p_pgrp;   /* Pointer to process group. */
        struct  user *p_addr;   /* Kernel virtual addr of u-area (PROC ONLY). */
        u_short p_xstat;        /* Exit status for wait; also stop signal. */
        u_short p_acflag;       /* Accounting flags. */
        struct  rusage *p_ru;   /* Exit information. XXX */
    };
    
其中`union p_un`的`size`为`0x10`，以及`p_flag`前面的两个指针分别占用`0x8`，可以确认结构体的内存占用图：

    -------------------
    |      p_flag     |  
    -------------------  kinfo_proc + 0x20
    |     p_sigacts   |  
    -------------------  kinfo_proc + 0x18
    |     p_vmspace   |  
    -------------------  kinfo_proc + 0x10
    |    union p_un   |  
    -------------------  kinfo_proc
    
比对标记并且将检测结果存放到`x0`中返回：

    #define P_TRACED        0x00000800

    __asm__ volatile(
        ......
        
        "ldr, x24, [x20, #0x20]\n"      // x24 = proc.kp_proc.p_flag
        "mov x25, #0x800\n"             // x25 = P_TRACED
        "blc x0, x24, x25\n"            // x0 = x24 & x25
        
        ......
    );
    

## 扩展阅读

[Kernel_Syscalls](https://www.theiphonewiki.com/wiki/Kernel_Syscalls)

[ARM64 架构之入栈/出栈操作](https://juejin.im/post/5cadeda55188251ad87b0eed)

[深入iOS系统底层之CPU寄存器](https://juejin.im/post/5a786c555188257a6854b18c)

[Procedure Call Standard for the ARM 64-bit Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0055b/IHI0055B_aapcs64.pdf)

