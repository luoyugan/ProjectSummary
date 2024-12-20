# 1 uboot启动流程第1阶段

主要流程：部分硬件初始化->加载完整的uboot到RAM->跳转到第2阶段入口开始执行

第1阶段主要用到的文件是：

- start.S文件，位于 u-boot/cpu/arm920t/start.S
- lowlevel_init.S文件，位于 u-boot/board/smdk2410/lowlevel_init.S

## 1.1 start.S文件分析

### 1. 启动-_start

查看下面的代码：

        _start:                                 # 异常处理向量表
        b start_code
        ldr pc, _undefined_instruction          # 未定义指令异常：0x00000004
        ldr pc, _software_interrupt             # 软中断异常:0x00000008
        ldr pc, _prefetch_abort                 # 预取异常：0x0000000C
        ldr pc, _data_abort                     # 数据异常:0x00000010
        ldr pc, _not_used                       # 未使用：0x00000014
        ldr pc, _irq                            # 外部中断请求IRQ:0x00000018
        ldr pc, _fiq                            # 快束中断请求FIQ:0x0000001C

从上面的内容可以看出，除第1行代码之外，其余代码都是跳转到特定位置去执行中断服务子程序。

正常情况下，程序的执行流程是不会走到中断处理流程中去的，而是直接跳转到reset处开始执行。那我们接下来就看reset处的代码都干了什么。

### 2. reset-设置超级管理模式

设置CPU为SVC32模式，即超级管理权限模式

        start_code:
            mrs r0,cpsr             # 将程序状态寄存器读取到通用寄存器R0
            bic r0,r0,#0x1f         # 清除当前的工作模式
            orr r0,r0,#0xd3         # 设置超级管理员权限
            msr cpsr,r0             # 将结果写回到CPSR寄存器

cpsr 是ARM体系结构中的程序状态寄存器，其结构如下：

|M[4:0] | CPU模式 | 可访问寄存器| 说明|
|-----------|----------|---------|--------|
|0b10000 | User | pc,R14~R0,CPSR | 正常ARM程序执行状态|
|0b10001 | FIQ | PC,R14_FIQ-R8_FIQ,R7~R0,CPSR,SPSR_FIQ | 为支持数据传输或通道处理设计|
|0b10010 | IRQ | PC,R14_IRQ-R13_IRQ,R12~R0,CPSR,SPSR_IRQ | 用于一般用途的中断处理|
|0b10011 | SUPERVISOR | PC,R14_SVC-R13_SVC,R12~R0,CPSR,SPSR_SVC | 操作系统保护模式|
|0b10111 | ABORT | PC,R14_ABT-R13_ABT,R12~R0,CPSR,SPSR_ABT | 数据或指令预取中止后进入|
|0b11011 | UNDEFINED | PC,R14_UND-R8_UND,R12~R0,CPSR,SPSR_UND | 执行未定义指令时进入|
|0b11111 | SYSTEM | PC,R14-R0,CPSR(ARM V4以及更高版本） | 操作系统的特权用户模式|

> I、F、T三位如果写1即禁用，所以reset后面的4句操作的结果为设置CPU为SUPERVISOR模式且禁用中断。那为什么选择这个模式呢？
> 首先，可以排除的就是ABORT和UNDEFINED模式，看上去就不像正常模式。
> 其次，对于快速中断fiq和中断irq来说，此处uboot初始化的时候，也还没啥中断要处理和能够处理，而且即使是注册了终端服务程序后，
> 能够处理中断，那么这两种模式，也是自动切换过去的，所以，此处也不应该设置为其中任何一种模式。
> 于usr模式，由于此模式无法直接访问很多的硬件资源，而uboot初始化，就必须要去访问这类资源，所以此处可以排除，不能设置为用户usr模式。
> 而svc模式本身就属于特权模式，本身就可以访问那些受控资源，而且，比sys模式还多了些自己模式下的影子寄存器，所以，相对sys模式来说，可以访问资源的能力相同，但是拥有更多的硬件资源。

### 3. 关闭看门狗

        ldr     r0, =pWTCON         # 取得看门狗寄存器的地址
        mov     r1, #0x0            # 将R1寄存器清0
        str     r1, [r0]            # 将看门狗寄存器清0，即将看门狗禁止，包括定时器定时，溢出中断及溢出复位等

### 4. 关中断

        mov r1, #0xffffffff         # 设R1寄存器为0xFFFF FFFF
        ldr r0, =INTMSK             # 读取中断屏蔽寄存器的地址
        str r1, [r0]                # 将中断屏蔽寄存器中的位全设1,屏蔽所有中断

        ldr r1, =INTSUBMSK_val      # 设R1寄存器为0xFFFF
        ldr r0, =INTSUBMSK          # 读取辅助中断屏蔽寄存器的地址
        str r1, [r0]                # 将辅助中断屏蔽寄中的11个中断信号屏蔽掉，本人觉得INTSUBMS_val应设成7ff

### 5. 设置时钟

### 6.关闭MCU，设置ARM时许

        #ifndef CONFIG_SKIP_LOWLEVEL_INIT
        cpu_init_crit:
            // 使I/D cache失效： 协处理寄存器操作，将r0中的数据写入到协处理器p15的c7中，c7对应cp15的cache控制寄存器
            mov r0, #0
            mcr p15, 0, r0, c7, c7, 0   /* flush v3/v4 cache */
            mcr p15, 0, r0, c8, c7, 0   /* flush v4 TLB */ 使TLB操作寄存器失效：将r0数据送到cp15的c8、c7中。C8对应TLB操作寄存器

            /*
            * 禁用MMU和缓存
            */
            mrc p15, 0, r0, c1, c0, 0   // 将c1、c0的值写入到r0中
            bic r0, r0, #0x00002300 @ clear bits 13, 9:8 (--V- --RS)
            bic r0, r0, #0x00000087 @ clear bits 7, 2:0 (B--- -CAM)
            orr r0, r0, #0x00000002 @ set bit 2 (A) Align
            orr r0, r0, #0x00001000 @ set bit 12 (I) I-Cache
            mcr p15, 0, r0, c1, c0, 0   // 将设置好的r0值写入到协处理器p15的c1、c0中，关闭MMU

            /*
            * 在重加载之前，我们必须设置RAM的时序，因为内存的时序依赖于板子，
            * 在board目录下可以发现lowlevel_init.S文件
            */
            mov ip, lr                  // 将lr寄存器内容保存到ip寄存器中，用于子程序调用返回
        #if defined(CONFIG_AT91RM9200DK) || defined(CONFIG_AT91RM9200EK) || defined(CONFIG_AT91RM9200DF)
        #else
            bl  lowlevel_init           // 跳转到`lowlevel_init`地址执行
        #endif
            mov lr, ip
            mov pc, lr
        #endif /* CONFIG_SKIP_LOWLEVEL_INIT */

**1.Cache是什么呢？**

Cache是处理器内部的一个高速缓存单元，为了应对处理器和外部存储器之间的速度不匹配而设立的。其速度要比内存的读写速度快好多，接近处理器的工作速度，一般处理器从内存中读取数据到Cache中，到下次再用到数据时，会先去cache中查找，如果cache中存在的话，就不会访问内存了，用以提高系统性能。

**2.系统引导时为什么关闭Cache？**

从上面的解释中，可以看出，在系统未初始化完成时，代码还没有转移到内存中，也就是说，我们还没有用到内存，先将MMU和Cache关闭，以免发生不可预料的错误。

**3.怎样使Cache中的数据无效？**

见上面的代码。

## 1.2 lowlevel_init.S文件分析

### 1.2.1 RAM初始化

这一步主要完成RAM的初始化，也就是通过写控制RAM的寄存器，对寄存器的存取方式进行控制。主要代码位于文件`lowlevel_init.S`中。

`lowlevel_init.S`文件内容如下：

        lowlevel_init:
            /* memory control configuration */
            /* make r0 relative the current location so that it */
            /* reads SMRDATA out of FLASH rather than memory ! */
            ldr r0, =SMRDATA                    // 读取下面标号为SMRDATA处的地址到R0中
            ldr r1, _TEXT_BASE                  // 程序的加载地址 TEXT_BASE = 0x33F80000 到 R1中
            sub r0, r0, r1                      // 计算SMRDATA的相对地址保存到R0中，
                                                /* SMRDATA为虚拟地址,而TEXT_BASE为虚拟地址的起始地址
                                                * TEXT_BASE为0x33F8 0000,SMRDATA为0x33F8 06C8
                                                * 而现在程序运行在起始地址为0x0000 0000的地方
                                                * 所以需要计算以0x0000 0000为标准的相对地址 */
            ldr r1, =BWSCON                     // 取得带宽与等待状态控制寄存器地址到R1中，也就是控制内存的寄存器的首地址
            add r2, r0, #13*4                   // R2保存要操作的寄存器的个数，在这儿是13
        0:
            ldr     r3, [r0], #4                // 数据处理后R0自加4，[R0]->R3，R0+4->R0
            str     r3, [r1], #4                // 将这些数据写入到控制内存的寄存器中。
            cmp     r2, r0                      // 循环从Flash中读取13个Word大小的值到内存中
            bne     0b

            mov pc, lr                          // 返回函数lowlevel_init()的调用地方

            .ltorg
        /* the literal pools origin */

            SMRDATA:
                .word (0+(B1_BWSCON<<4)+(B2_BWSCON<<8)+(B3_BWSCON<<12)+(B4_BWSCON<<16)+(B5_BWSCON<<20)+(B6_BWSCON<<24)+(B7_BWSCON<<28))
                .word ((B0_Tacs<<13)+(B0_Tcos<<11)+(B0_Tacc<<8)+(B0_Tcoh<<6)+(B0_Tah<<4)+(B0_Tacp<<2)+(B0_PMC))
                .word ((B1_Tacs<<13)+(B1_Tcos<<11)+(B1_Tacc<<8)+(B1_Tcoh<<6)+(B1_Tah<<4)+(B1_Tacp<<2)+(B1_PMC))
                .word ((B2_Tacs<<13)+(B2_Tcos<<11)+(B2_Tacc<<8)+(B2_Tcoh<<6)+(B2_Tah<<4)+(B2_Tacp<<2)+(B2_PMC))
                .word ((B3_Tacs<<13)+(B3_Tcos<<11)+(B3_Tacc<<8)+(B3_Tcoh<<6)+(B3_Tah<<4)+(B3_Tacp<<2)+(B3_PMC))
                .word ((B4_Tacs<<13)+(B4_Tcos<<11)+(B4_Tacc<<8)+(B4_Tcoh<<6)+(B4_Tah<<4)+(B4_Tacp<<2)+(B4_PMC))
                .word ((B5_Tacs<<13)+(B5_Tcos<<11)+(B5_Tacc<<8)+(B5_Tcoh<<6)+(B5_Tah<<4)+(B5_Tacp<<2)+(B5_PMC))
                .word ((B6_MT<<15)+(B6_Trcd<<2)+(B6_SCAN))
                .word ((B7_MT<<15)+(B7_Trcd<<2)+(B7_SCAN))
                //设置REFRESH,在S3C2440中11～17位是保留的，也即(Tchr<<16)无意义
                .word ((REFEN<<23)+(TREFMD<<22)+(Trp<<20)+(Trc<<18)+(Tchr<<16)+REFCNT)
                .word 0x32      // 设置BANKSIZE,对于容量可以设置大写，多出来的空内存会被自动检测出来
                .word 0x30      // 设置MRSRB6
                .word 0x30      // 设置MRSRB7

### 1.2.2 uboot代码加载

        #ifndef CONFIG_SKIP_RELOCATE_UBOOT
            adr r0, _start          /* r0保存当前程序的位置 */
        relocate:                   /* 将uboot代码重定位到RAM中 */
            teq r0, #0              /* 测试是否从地址0开始运行 */
            bleq    may_resume      /* yes -> do low-level setup */

            adr r0, _start          /* 上面的代码有可能会破会r0中的值 */
            ldr r1, _TEXT_BASE      /* 测试从Flash还是RAM中运行程序，它们的地址是不一样的 */
            cmp r0, r1              /* 在debug期间不需要重定位，直接在Flash中运行代码 */
            beq     done_relocate

            ldr r2, _armboot_start
            ldr r3, _bss_start
            sub r2, r3, r2          /* 根据前面分析的uboot.lds文件可知，r3-r2就是uboot代码的大小，将其存入寄存器r2中 */
            add r2, r0, r2          /* r0是程序的起始地址，加上uboot代码的大小就是uboot代码的结束地址 */

        copy_loop:
            ldmia   r0!, {r3-r10}   /* 从源地址[r0]处开始拷贝 */
            stmia   r1!, {r3-r10}   /* 拷贝到目标地址[r1]处 */
            cmp r0, r2              /* 直到源代码结束地址[r2] */
            ble copy_loop

### 1.2.3 建立堆栈

设置堆栈，其中，`_TEXT_BASE=0x33F80000`，而`_TEXT_BASE=0x33F80000`,`CFG_GBL_DATA_SIZE`,`CONFIG_STACKSIZE_IRQ`,`CONFIG_STACKSIZE_FIQ`在文件uboot/include/configs/mini2440.h中定义。

        /* 建立堆栈 */
        stack_setup:
            ldr r0, _TEXT_BASE                                          /* upper 128 KiB: relocated uboot   */
            sub r0, r0, #CFG_MALLOC_LEN                                 /* malloc area */
            sub r0, r0, #CFG_GBL_DATA_SIZE                              /* bdinfo */
        #ifdef CONFIG_USE_IRQ
            sub r0, r0, #(CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ)
        #endif
            sub sp, r0, #12                                             /* leave 3 words for abort-stack    */

### 1.2.4 清除bss段

        clear_bss:
            ldr r0, _bss_start                                          /* find start of bss segment        */
            ldr r1, _bss_end                                            /* stop here                        */
            mov     r2, #0x00000000                                     /* clear                            */

        clbss_l:str r2, [r0]                                            /* clear loop...                    */
            add r0, r0, #4
            cmp r0, r1
            ble clbss_l

### 1.2.5 跳转到uboot第2阶段

            ldr pc, _start_armboot

        _start_armboot: .word start_armboot

初始化外设完成之后，程序跳转到u-boot第2阶段的入口函数 start_armboot 。 ldr pc，_start_armboot 为绝对跳转命令，pc值等于_start_armboot的连接地址，程序跳到SDRAM中执行。在此之前程序都是在flash中运行的，绝对跳转必须在初始SDRAM，执行代码重定位之后才能进行。


