# 1 uboot启动流程第2阶段

第2阶段，uboot完成进一步的硬件初始化，并设置了uboot下的命令行、环境变量、并跳转到内核中。其主要用到的文件是：

- board.c文件，位于`u-boot/lib_arm/board.c`
- main.c文件，位于 `u-boot/common/main.c`

1.1 初始化

        void start_armboot (void)
        {
            init_fnc_t **init_fnc_ptr;
            char *s;
        #ifndef CFG_NO_FLASH
            ulong size;
        #endif
        #if defined(CONFIG_VFD) || defined(CONFIG_LCD)
            unsigned long addr;
        #endif

            /* 在上面的代码中gd的值绑定到寄存器r8中了 */
            gd = (gd_t*)(_armboot_start - CFG_MALLOC_LEN - sizeof(gd_t));
            /* 为GCC >= 3.4以上的编译进行代码优化，而插入内存barrier */
            __asm__ __volatile__("": : :"memory");

            memset ((void*)gd, 0, sizeof (gd_t));
            gd->bd = (bd_t*)((char*)gd - sizeof(bd_t));
            memset (gd->bd, 0, sizeof (bd_t));

            monitor_flash_len = _bss_start - _armboot_start;

            for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
                if ((*init_fnc_ptr)() != 0) {
                    hang ();
                }
            }
        }

首先，我们先来分析`init_fnc_t **init_fnc_ptr;`这行代码。

要分析这行代码，首先看指针数组`init_fnc_t *init_sequence[]`

        typedef int (init_fnc_t) (void);

        init_fnc_t *init_sequence[] = {
            cpu_init,               /* 与CPU相关的初始化 */
            board_init,             /* 与板子初始化相关的初始化 */
            interrupt_init,         /* 中断初始化 */
            env_init,               /* 初始化环境变量 */
            init_baudrate,          /* 初始化波特率设置 */
            serial_init,            /* serial通信相关初始化 */
            console_init_f,         /* console初始化的第一部分 */
            display_banner,         /* say that we are here */
            // ...根据配置，还有一些其它的初始化
            dram_init,              /* 配置可用的RAM块 */
            display_dram_config,
            NULL,
        };

根据这儿的分析，我们就可以知道`init_fnc_ptr`就是一个函数指针。在后面的for循环中，将函数指针数组的首地址`init_sequence`赋值给`init_fnc_ptr`，然后循环，对所有的硬件进行初始化。

而对于代码`gd = (gd_t*)(_armboot_start - CFG_MALLOC_LEN - sizeof(gd_t));`确实有些抽象。而要分析它，必须看一下下面这个宏定义：

        DECLARE_GLOBAL_DATA_PTR;        //在board.c最上面
    
而它的定义如下：

        #define DECLARE_GLOBAL_DATA_PTR     register volatile gd_t *gd asm ("r8")

这个声明，告诉编译器使用寄存器r8来存储gd_t类型的指针gd，即这个定义声明了一个指针，并且指明了它的存储位置。也就是说，我们声明了一个寄存器变量，它的初始值为_armboot_start - CFG_MALLOC_LEN - sizeof(gd_t)，也就是0x33F80000-(0x20000+2048*1024)-0x24。也就是说，gd指向了一段可用的内存区域，而这段内存区域相当于u-boot的全局变量。

那指针gd指向的数据结构到底是什么呢？为什么要设置这个数据结构呢？那么接下来让我们看一下这个数据结构吧。

        typedef struct  global_data {
            bd_t            *bd;
            unsigned long   flags;
            unsigned long   baudrate;
            unsigned long   have_console;   /* serial_init() 函数被调用 */
            unsigned long   reloc_off;      /* Relocation Offset */
            unsigned long   env_addr;       /* Address  of Environment struct */
            unsigned long   env_valid;      /* Checksum of Environment valid? */
            unsigned long   fb_base;        /* base address of frame buffer */
        #ifdef CONFIG_VFD
            unsigned char   vfd_type;       /* display type */
        #endif
            void            **jt;           /* jump table */
        } gd_t;
