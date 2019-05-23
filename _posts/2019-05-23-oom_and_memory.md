---
title: OOM与内存
date: 2019-05-23 08:00
tags:
- 开发笔记
---

微视的`crash log`会夹带应用剩余内存信息上报给服务器，希望借此协助诊断崩溃是否由`OOM`引发。多数情况下，`OOM`不直接导致`crash`，而是以`SIGSEGV`的信号错误，其操作逻辑更像是内存无法分配，却依然访问这个无效地址：

    int *ptr = malloc(sizeof(int *));
    *ptr = 0x100;   // crash by access invalid memory
    
下面是一个只保留了主线程调用栈的`crash log`（屏蔽部分敏感信息后）：

    Handler: Signal Handler

    Hardware Model: iPhone8,2
    Process: microvision [2581]
    Path: /var/containers/Bundle/Application/C25EDAC2-4686-4033-A481-88FB80D8CA2F/microvision.app
    Identifier: com.tencent.microvision
    Version: 5.4.3(645)
    Code Type: ARM-64 (Native)
    Parent Process:  [1]
    
    Date/Time: 2019-05-18 09:13:42.534 +0800
    OS Version: iPhone OS 12.1.4 (16D57)
    Report Version: 104
    
    SDK start time: 2019-05-18 08:28:51
    SDK server time delta: 0 s
    last record time delta : 2691534 ms
    RDM SDK Version: XXXX
    RDM Registed Uin : XXXX
    RDM DeviceId: XXXX
    RDM CI UUID: XXXX
    RDM APP KEY: XXXX
    
    Exception Type: SIGSEGV
    Exception Codes: SEGV_ACCERR at 0x0000000000000008
    Crashed Thread: 0
    
    Thread 0 Crashed: 
    0  QuartzCore                     0x000000018d09e224 CA::AttrList::set(unsigned int, _CAValueType, void const*) +  176
    1  QuartzCore                     0x000000018d042674 CA::Layer::setter(unsigned int, _CAValueType, void const*) +  372
    2  QuartzCore                     0x000000018d03edfc -[CALayer setOpacity:] +  64
    3  microvision                    0x000000010288ce94 -[XXXX createBlurViewWithFrame:] (XXXX.m:175)
    4  microvision                    0x000000010288d114 -[XXXX createButtonWithImageName:interactionType:index:] (XXXX.m:199)
    5  microvision                    0x000000010288bda0 -[XXXX init] (XXXX.m:78)
    6  microvision                    0x000000010270ceb4 -[XXXX setupView] (XXXX.m:162)
    7  microvision                    0x000000010270cae4 -[XXXX initWithFrame:] (XXXX.m:140)
    8  microvision                    0x0000000102677524 -[XXXX commonInit] (XXXX.m:202)
    9  microvision                    0x0000000102676bf4 -[XXXX initWithFrame:] (XXXX.m:159)
    10 microvision                    0x0000000102bc5534 -[XXXX initDetailCell] (XXXX.m:21)
    11 microvision                    0x0000000102bc5630 -[XXXX initWithFrame:] (XXXX.m:33)
    12 UIKitCore                      0x00000001b54bfe6c -[UICollectionView _dequeueReusableViewOfKind:withIdentifier:forIndexPath:viewCategory:] +  2172
    13 UIKitCore                      0x00000001b54c01b0 -[UICollectionView dequeueReusableCellWithReuseIdentifier:forIndexPath:] +  180
    14 microvision                    0x0000000102cb18f8 _$SSo16UICollectionViewC11microvisionE22lk_dequeueReusableCell9indexPathx10Foundation05IndexI0V_tlFSo012WSChannelSetG0C_Tg5Tm +  340
     +  340
    15 microvision                    0x000000010315cf58 collectionView (WSVideoListBaseController.swift:0)
    16 microvision                    0x0000000102dc0710 collectionView (WSFeedDetailListViewController.swift:462)
    17 microvision                    0x0000000102dc5ffc collectionView (<compiler-generated>:0)
    18 UIKitCore                      0x00000001b54ac39c -[UICollectionView _createPreparedCellForItemAtIndexPath:withLayoutAttributes:applyAttributes:isFocused:notify:] +  356
    19 UIKitCore                      0x00000001b54b06dc -[UICollectionView _updateVisibleCellsNow:] +  4036
    20 UIKitCore                      0x00000001b54b577c -[UICollectionView layoutSubviews] +  324
    21 UIKitCore                      0x00000001b608977c -[UIView(CALayerDelegate) layoutSublayersOfLayer:] +  1380
    22 QuartzCore                     0x000000018d03db7c -[CALayer layoutSublayers] +  184
    23 QuartzCore                     0x000000018d042b34 CA::Layer::layout_if_needed(CA::Transaction*) +  324
    24 QuartzCore                     0x000000018cfa1598 CA::Context::commit_transaction(CA::Transaction*) +  340
    25 QuartzCore                     0x000000018cfcfec8 CA::Transaction::commit() +  608
    26 QuartzCore                     0x000000018cfd0d30 CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*) +  92
    27 CoreFoundation                 0x00000001889d16bc ___CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ +  32
     +  32
    28 CoreFoundation                 0x00000001889cc350 ___CFRunLoopDoObservers +  412
    29 CoreFoundation                 0x00000001889cc8f0 ___CFRunLoopRun +  1264
    30 CoreFoundation                 0x00000001889cc0e0 CFRunLoopRunSpecific + 436
    31 GraphicsServices               0x000000018ac45584 GSEventRunModal + 96
    32 UIKitCore                      0x00000001b5be0c00 UIApplicationMain + 212
    33 microvision                    0x0000000102986ea4 main (main.m:40)
    34 libdyld.dylib                  0x000000018848abb4 _start +  4

除了调用栈外，通过`task_vm_info_data_t`计算获取的设备可用内存为`1560.81MB`，基本可以排除内存不足的可能性。另外，`log`中最后的非系统库调用代码如下：

    UIVisualEffectView *blurView = [[UIVisualEffectView alloc] initWithEffect: [UIBlurEffect effectWithStyle: UIBlurEffectStyleDark]];
    blurView.layer.cornerRadius = kButtonSize / 2;
    blurView.layer.masksToBounds = YES;
    blurView.frame = frame;
    blurView.alpha = 0.5;           /// crash发生的位置
    [self addSubview: blurView];

日志中的`Exception Codes`显示的是`SEGV_ACCERR at 0x0000000000000008`，由于`ASLR`保护技术的存在，运行在`64`位的应用会随机分配一个起始偏移地址，起始地址大于`0x100000000`，因此被访问的这个地址是无效的危险地址，从客户端代码已经无法定位问题，需要崩溃处的系统`api`做追查。

### 调用还原
利用`Xcode`的符号断点很快就获得对应的指令代码（只展示到`crash`位置指令为止）：

    QuartzCore`CA::AttrList::set:
    ->  0x21e47d6cc <+0>:   stp    x26, x25, [sp, #-0x50]!
        0x21e47d6d0 <+4>:   stp    x24, x23, [sp, #0x10]
        0x21e47d6d4 <+8>:   stp    x22, x21, [sp, #0x20]
        0x21e47d6d8 <+12>:  stp    x20, x19, [sp, #0x30]
        0x21e47d6dc <+16>:  stp    x29, x30, [sp, #0x40]
        0x21e47d6e0 <+20>:  add    x29, sp, #0x40            ; =0x40 
        0x21e47d6e4 <+24>:  mov    x20, x3
        0x21e47d6e8 <+28>:  mov    x22, x2
        0x21e47d6ec <+32>:  mov    x23, x1
        0x21e47d6f0 <+36>:  mov    x19, x0
        0x21e47d6f4 <+40>:  b      0x21e47d71c               ; <+80>
        0x21e47d6f8 <+44>:  mov    x21, x19
        0x21e47d6fc <+48>:  mov    x0, x21
        0x21e47d700 <+52>:  bl     0x21e47d498               ; CA::AttrList::copy_()
        0x21e47d704 <+56>:  mov    x19, x0
        0x21e47d708 <+60>:  sub    w8, w25, #0x1             ; =0x1 
        0x21e47d70c <+64>:  ldr    x9, [x21, #0x8]
        0x21e47d710 <+68>:  and    x9, x9, #0xfffffffffffffff8
        0x21e47d714 <+72>:  orr    x8, x9, x8
        0x21e47d718 <+76>:  str    x8, [x21, #0x8]
        0x21e47d71c <+80>:  mov    x24, x19
        0x21e47d720 <+84>:  ldr    w8, [x24, #0x8]!
        0x21e47d724 <+88>:  ands   w25, w8, #0x7
        0x21e47d728 <+92>:  b.ne   0x21e47d6f8               ; <+44>
        0x21e47d72c <+96>:  ldr    x21, [x19]
        0x21e47d730 <+100>: cbz    x21, 0x21e47d764          ; <+152>
        0x21e47d734 <+104>: mov    w8, #0x0
        0x21e47d738 <+108>: mov    x25, x19
        0x21e47d73c <+112>: ldr    w9, [x21, #0x8]
        0x21e47d740 <+116>: and    w10, w9, #0xffffff
        0x21e47d744 <+120>: cmp    w10, w23
        0x21e47d748 <+124>: b.eq   0x21e47d7dc               ; <+272>
        0x21e47d74c <+128>: cmp    w9, #0x0                  ; =0x0 
        0x21e47d750 <+132>: cset   w9, lt
        0x21e47d754 <+136>: orr    w8, w8, w9
        0x21e47d758 <+140>: mov    x25, x21
        0x21e47d75c <+144>: ldr    x21, [x21]
        0x21e47d760 <+148>: cbnz   x21, 0x21e47d73c          ; <+112>
        0x21e47d764 <+152>: orr    w0, wzr, #0x18
        0x21e47d768 <+156>: bl     0x21e37ea90               ; get_malloc_zone(unsigned long)
        0x21e47d76c <+160>: orr    w1, wzr, #0x18
        0x21e47d770 <+164>: bl     0x219a17900               ; malloc_zone_malloc
        <+168>: mov    x21, x0
        <+172>: and    w8, w23, #0xffffff
        <+176>: str    w8, [x21, #0x8]      // crash

从下往上看几个关键指令：

- `<+176>: str w8, [x21, #0x8]`
    
    崩溃行指令将`x8`寄存器的数据存储到`x21`地址偏移`0x8`的地址空间中导致了崩溃，结合日志中的`SEGV_ACCERR at 0x0000000000000008`，说明`x21`的地址值为`0`

- `<+152>: orr w0, wzr, #0x18`<br>`<+156>: bl     0x21e37ea90`<br>`<+160>: orr w1, wzr, #0x18`<br>`<+164>: bl`<br>`0x21e47d774 <+168>: mov x21, x0`

    按照`arm64`指令的调用约定，函数调用的返回值会存储在`(x/w)0`寄存器中，函数参数从左到右依次存放到`(x/w)0 ~ (x/w)7`这些寄存器中。`x21`寄存器的数据自于函数`malloc_zone_malloc`的返回值，根据函数名称可以断定这是内存分配失败后的无效地址访问错误
    
根据关键指令进行崩溃位置的函数还原伪代码：

    CA::AttrList::set(unsigned int arg1, _CAValueType arg2, void const* arg3) {
        ......
        unsigned int size = 24;
        _CAValueType *ptr = (_CAValueType *)malloc_zone_malloc(get_malloc_zone(size), size);
        _CAValueType val = arg2 & 0xffffff;
        ptr++;
        *ptr = val;     // crash
    }
    
通过对函数的汇编还原，基本可以认为这是一起`OOM`，但为什么通过`task_vm_info_data_t`获取到设备可用内存依然有这么多？

### malloc_zone
在`macOS/iOS`上，苹果使用了名为`malloc_zone`的内存分配方式，将内存块的分配按照尺寸拆成不同的`zone`来完成，一个`zone`可以看做是一系列内存分配相关`api`的组合结构，其结构表现如下：

    typedef struct _malloc_zone_t {
    
        void	*reserved1;	/* RESERVED FOR CFAllocator DO NOT USE */
        void	*reserved2;	/* RESERVED FOR CFAllocator DO NOT USE */
        size_t 	(* MALLOC_ZONE_FN_PTR(size))(struct _malloc_zone_t *zone, const void *ptr); /* returns the size of a block or 0 if not in this zone; must be fast, especially for negative answers */
        void 	*(* MALLOC_ZONE_FN_PTR(malloc))(struct _malloc_zone_t *zone, size_t size);
        void 	*(* MALLOC_ZONE_FN_PTR(calloc))(struct _malloc_zone_t *zone, size_t num_items, size_t size); /* same as malloc, but block returned is set to zero */
        void 	*(* MALLOC_ZONE_FN_PTR(valloc))(struct _malloc_zone_t *zone, size_t size); /* same as malloc, but block returned is set to zero and is guaranteed to be page aligned */
        void 	(* MALLOC_ZONE_FN_PTR(free))(struct _malloc_zone_t *zone, void *ptr);
        void 	*(* MALLOC_ZONE_FN_PTR(realloc))(struct _malloc_zone_t *zone, void *ptr, size_t size);
        void 	(* MALLOC_ZONE_FN_PTR(destroy))(struct _malloc_zone_t *zone); /* zone is destroyed and all memory reclaimed */
        const char	*zone_name;
    
        /* Optional batch callbacks; these may be NULL */
        unsigned	(* MALLOC_ZONE_FN_PTR(batch_malloc))(struct _malloc_zone_t *zone, size_t size, void **results, unsigned num_requested); /* given a size, returns pointers capable of holding that size; returns the number of pointers allocated (maybe 0 or less than num_requested) */
        void	(* MALLOC_ZONE_FN_PTR(batch_free))(struct _malloc_zone_t *zone, void **to_be_freed, unsigned num_to_be_freed); /* frees all the pointers in to_be_freed; note that to_be_freed may be overwritten during the process */
    
        struct malloc_introspection_t	* MALLOC_INTROSPECT_TBL_PTR(introspect);
        unsigned	version;
        	
        /* aligned memory allocation. The callback may be NULL. Present in version >= 5. */
        void *(* MALLOC_ZONE_FN_PTR(memalign))(struct _malloc_zone_t *zone, size_t alignment, size_t size);
        
        /* free a pointer known to be in zone and known to have the given size. The callback may be NULL. Present in version >= 6.*/
        void (* MALLOC_ZONE_FN_PTR(free_definite_size))(struct _malloc_zone_t *zone, void *ptr, size_t size);
    
        /* Empty out caches in the face of memory pressure. The callback may be NULL. Present in version >= 8. */
        size_t 	(* MALLOC_ZONE_FN_PTR(pressure_relief))(struct _malloc_zone_t *zone, size_t goal);
    
        boolean_t (* MALLOC_ZONE_FN_PTR(claimed_address))(struct _malloc_zone_t *zone, void *ptr);
    } malloc_zone_t;

部分结构在`malloc/malloc.h`头文件中可以找到，关键的分配函数开源在[libmalloc](https://opensource.apple.com/source/libmalloc/libmalloc-116.30.3/src/)，不同大小的内存块分配类型如下：

  
| 类型 | 尺寸范围 | 最小分割单位 |
| --- | --- | --- |
| nano | 16~256B | 16B |
| tiny | 16~1008B | 16B |
| small | 1~15KB(设备内存小于1GB) / 1~127KB(大于等于1GB) | 512B |
| large | 15+KB(<1GB) / 127+KB(>=1GB) | 内核页表尺寸 |

在`64`位系统上，默认情况下采用`nano`类型（<=`256B`）的内存分配方式，考虑到这点，下文只描述这种类型的内存分配

#### 结构
和`nano`相关的主要结构体存放在`nano_zone.h`当中，标明了重要参数的意义：

    /// 可分配内存信息结构
    typedef struct nano_meta_s {
        OSQueueHead			slot_LIFO MALLOC_NANO_CACHE_ALIGN;
        unsigned int		slot_madvised_log_page_count;
        volatile uintptr_t		slot_current_base_addr;     // 内存分配基址
        volatile uintptr_t		slot_limit_addr;            // 内存分配地址上限
        volatile size_t		slot_objects_mapped;     
        volatile size_t		slot_objects_skipped;     
        bitarray_t			slot_madvised_pages;            // 当前分配地址

        volatile uintptr_t		slot_bump_addr MALLOC_NANO_CACHE_ALIGN;
        volatile boolean_t		slot_exhausted;
        unsigned int		slot_bytes;                     // 分配的内存size                 
        unsigned int		slot_objects;                   // 可存储的内存块个数
    } *nano_meta_admin_t;
    
    
    /// nano类型分配
    typedef struct nanozone_s {
        malloc_zone_t		basic_zone;      // 内存分配zone
        uint8_t			    pad[PAGE_MAX_SIZE - sizeof(malloc_zone_t)];
    
        struct nano_meta_s		meta_data[NANO_MAG_SIZE][NANO_SLOT_SIZE];   // 可用内存信息二维列表
        _malloc_lock_s			band_resupply_lock[NANO_MAG_SIZE];          // 锁
        uintptr_t               band_max_mapped_baseaddr[NANO_MAG_SIZE];
        size_t			        core_mapped_size[NANO_MAG_SIZE];
    
        unsigned			debug_flags;
        unsigned			our_signature;
        unsigned			phys_ncpus;      // 物理cpu数量
        unsigned			logical_ncpus;   // 逻辑cpu数量
        unsigned			hyper_shift;     // 用于进行逻辑cpu->物理cpu的计算
    
        /* security cookie */
        uintptr_t			cookie;
    
        malloc_zone_t		*helper_zone;    // 非nano内存类型的分配zone
    } nanozone_t;

借用[阿里云栖社区](https://yq.aliyun.com/articles/3065)的图表示结构：

![](http://img1.tbcdn.cn/L1/461/1/4960742425dc6b93cf39f56546db2203a9a0a231)

`nano_zone`使用类似位图的结构体数组来存放可分配的内存地址信息，图中纵向方向表示物理`cpu`核心所能分配的内存信息数组。考虑到现代`cpu`的超线程设计，一个物理`cpu`核心上存在大于`1`个的逻辑`cpu`执行流，因此使用`hyper_shift`来协助获取真实`cpu`的信息：

    malloc_zone_t* create_nano_zone(size_t initial_size, malloc_zone_t *helper_zone, unsigned debug_flags) {
        ......
        nanozone->phys_ncpus = *(uint8_t *)(uintptr_t)_COMM_PAGE_PHYSICAL_CPUS;
        nanozone->logical_ncpus = *(uint8_t *)(uintptr_t)_COMM_PAGE_LOGICAL_CPUS;
        
        switch (nanozone->logical_ncpus/nanozone->phys_ncpus) {
    		case 1:
    			nanozone->hyper_shift = 0;
    			break;
    		case 2:
    			nanozone->hyper_shift = 1;
    			break;
    		case 4:
    			nanozone->hyper_shift = 2;
    			break;
    		default:
    			malloc_printf("*** FATAL ERROR - logical_ncpus / phys_ncpus not 1, 2, or 4.\n");
    			exit(-1);
    	}
    }

    #define NANO_MAG_BITS			5
    #define NANO_MAG_SIZE 		    (1 << NANO_MAG_BITS)
    #define NANO_MAG_INDEX(nz)		(_os_cpu_number() >> nz->hyper_shift)

由于`nano`可分配的尺寸大小为`16~256B`，因此按照`16B`最小单位进行分割成`16`份

    #define NANO_MAX_SIZE			256 /* Buckets sized {16, 32, 48, 64, 80, 96, 112, ...} */
    #define NANO_SLOT_BITS			4
    #define NANO_SLOT_SIZE 		   (1 << NANO_SLOT_BITS)

#### 分配
[malloc.c](https://opensource.apple.com/source/libmalloc/libmalloc-116.30.3/src/malloc.c.auto.html)中可以看到`malloc_zone_malloc`的源码实现：

    void *
    malloc_zone_malloc(malloc_zone_t *zone, size_t size)
    {
    	MALLOC_TRACE(TRACE_malloc | DBG_FUNC_START, (uintptr_t)zone, size, 0, 0);
    
    	void *ptr;
    	if (malloc_check_start && (malloc_check_counter++ >= malloc_check_start)) {
    		internal_check();
    	}
    	if (size > MALLOC_ABSOLUTE_MAX_SIZE) {
    		return NULL;
    	}
    
    	ptr = zone->malloc(zone, size);    // 核心实现
    
    	
    	if (malloc_logger) {
    		malloc_logger(MALLOC_LOG_TYPE_ALLOCATE | MALLOC_LOG_TYPE_HAS_ZONE, (uintptr_t)zone, (uintptr_t)size, 0, (uintptr_t)ptr, 0);
    	}
    
    	MALLOC_TRACE(TRACE_malloc | DBG_FUNC_END, (uintptr_t)zone, size, (uintptr_t)ptr, 0);
    	return ptr;
    }

整个内存分配的实现分为三步：
1. 安全条件检测
2. 通过`zone`申请内存
3. 日志记录输出

`nano_zone`与内存分配相关的初始化同样放在`create_nano_zone`中：

    void *
    malloc_zone_malloc(malloc_zone_t *zone, size_t size)
    {
        nanozone->basic_zone.version = 8;
        nanozone->basic_zone.size = (void *)nano_size;
        nanozone->basic_zone.malloc = (debug_flags & SCALABLE_MALLOC_DO_SCRIBBLE) ? (void *)nano_malloc_scribble : (void *)nano_malloc;
        nanozone->basic_zone.calloc = (void *)nano_calloc;
        nanozone->basic_zone.valloc = (void *)nano_valloc;
        nanozone->basic_zone.free = (debug_flags & SCALABLE_MALLOC_DO_SCRIBBLE) ? (void *)nano_free_scribble : (void *)nano_free;
        nanozone->basic_zone.realloc = (void *)nano_realloc;
        nanozone->basic_zone.destroy = (void *)nano_destroy;
        nanozone->basic_zone.batch_malloc = (void *)nano_batch_malloc;
        nanozone->basic_zone.batch_free = (void *)nano_batch_free;
        nanozone->basic_zone.introspect = (struct malloc_introspection_t *)&nano_introspect;
        nanozone->basic_zone.memalign = (void *)nano_memalign;
        nanozone->basic_zone.free_definite_size = (debug_flags & SCALABLE_MALLOC_DO_SCRIBBLE) ?
        (void *)nano_free_definite_size_scribble : (void *)nano_free_definite_size;
    }

默认情况下`SCALABLE_MALLOC_DO_SCRIBBLE`总是不启用的，因此申请内存最终走到了`nano_malloc`函数中（移除`DEBUG`代码）：

    static void *
    nano_malloc(nanozone_t *nanozone, size_t size)
    {
        if (size <= NANO_MAX_SIZE) {
            void *p = _nano_malloc_check_clear(nanozone, size, 0);
            if (p) {
                return p;
            } else {
                /* FALLTHROUGH to helper zone */
            }
        }
        
        malloc_zone_t *zone = (malloc_zone_t *)(nanozone->helper_zone);
        return zone->malloc(zone, size);
    }
    
    static MALLOC_INLINE size_t
    segregated_size_to_fit(nanozone_t *nanozone, size_t size, size_t *pKey)
    {
        size_t k, slot_bytes;
            
        if (0 == size) {
            size = NANO_REGIME_QUANTA_SIZE; // Historical behavior
        }
        k = (size + NANO_REGIME_QUANTA_SIZE - 1) >> SHIFT_NANO_QUANTUM; // round up and shift for number of quanta
        slot_bytes = k << SHIFT_NANO_QUANTUM;							// multiply by power of two quanta size
        *pKey = k - 1;													// Zero-based!
    
        return slot_bytes;
    }

    static void *
    _nano_malloc_check_clear(nanozone_t *nanozone, size_t size, boolean_t cleared_requested)
    {
        // 记录输出
        MALLOC_TRACE(TRACE_nano_malloc, (uintptr_t)nanozone, size, cleared_requested, 0);
    
        /// 获取cpu对应的index以及内存size对应的slot
        void *ptr;
        size_t slot_key;
        size_t slot_bytes = segregated_size_to_fit(nanozone, size, &slot_key); // Note slot_key is set here
        unsigned int mag_index = NANO_MAG_INDEX(nanozone);
        nano_meta_admin_t pMeta = &(nanozone->meta_data[mag_index][slot_key]);
    
        /// 检测最近释放的内存块是否存在可用的
        ptr = OSAtomicDequeue(&(pMeta->slot_LIFO), offsetof(struct chained_block_s, next));
        if (ptr) {
        #if NANO_FREE_DEQUEUE_DILIGENCE
            size_t gotSize;
            nano_blk_addr_t p; // the compiler holds this in a register
    
            /// 检测nano标记位
            p.addr = (uint64_t)ptr;
            if (nanozone->our_signature != p.fields.nano_signature) {
                nanozone_error(nanozone, 1, "Invalid signature for pointer dequeued from free list", ptr, NULL);
            }
    
            /// 检测cpu核心是否相同
            if (mag_index != p.fields.nano_mag_index) {
                nanozone_error(nanozone, 1, "Mismatched magazine for pointer dequeued from free list", ptr, NULL);
            }
    
            /// 校验slot信息
            gotSize = _nano_vet_and_size_of_free(nanozone, ptr);
            if (0 == gotSize) {
                nanozone_error(nanozone, 1, "Invalid pointer dequeued from free list", ptr, NULL);
            }
            if (gotSize != slot_bytes) {
                nanozone_error(nanozone, 1, "Mismatched size for pointer dequeued from free list", ptr, NULL);
            }
    
    		if ((((chained_block_t)ptr)->double_free_guard ^ nanozone->cookie) != 0xBADDC0DEDEADBEADULL) {
                nanozone_error(nanozone, 1, "Heap corruption detected, free list canary is damaged", ptr, NULL);
    		}
        #endif /* NANO_FREE_DEQUEUE_DILIGENCE */
        
    		((chained_block_t)ptr)->double_free_guard = 0;
    		((chained_block_t)ptr)->next = NULL; // clear out next pointer to protect free list
    	} else {
            /// 获取新的内存
            ptr = segregated_next_block(nanozone, pMeta, slot_bytes, mag_index);
    	}
    
        if (cleared_requested && ptr) {
            memset(ptr, 0, slot_bytes); // TODO: Needs a memory barrier after memset to ensure zeroes land first?
        }
        return ptr;
    }

由于涉及到的函数过多且篇幅问题，想进一步了解`slot`所属的内存块`band`的分配规则，建议阅读源码。最后同样放上一张同样来自云栖社区的非`nano`类型分配的结构图，配合源码阅读效果更佳：

![](http://img1.tbcdn.cn/L1/461/1/c50505c7577e0d883c3a33c21c1a0763ae408350)

### 结论
苹果系统将内存分配分类并使用不同的`zone`完成内存申请，因此在应用启动之后，每一种类型能够使用的地址范围已经被限定。这导致了即便只有单一类型的内存地址被分配完毕，设备依然有可用内存的情况下，同样会引发`OOM`问题

