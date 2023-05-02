---
title: OS实验日志2.1-cpu初始化
date: 2023-05-02 11:01:09
tags:
---

# cpu初始化

首先要进行cpu初始化

    cpu_init();

在cpu初始化中调用gdt初始化

    void cpu_init(void)
    {
        init_gdt();
    }


## GDT

在操作系统中，每个处理器对应一个全局描述符表（GDT）

为何需要GDT表？<br>
    因为对于x86系统，电脑开机时处于实模式下，只能访问20位地址总线，寻址范围仅为1M，使用16位段寄存器+16位段偏移来寻址，当进入保护模式时，可以使用32位地址总线，寻址范围扩大为4GB，而寄存器位数并没有扩大，旧的寻址模式不能访问1M以上内存，此时就产生了GDT结构。

    不改变寄存器位数的原因是为了向下兼容16位，同时又能保证64位甚至更高位的cpu能够向下兼容

### GDT结构
在上一章中我们提到GDT是描述逻辑地址的数据结构，它将代码分为不同的段来描述，每种段对应一个段描述符，cpu通过存储在之前16位的段寄存器中的段选择子来选择段描述符，GDT本身的地址存储在一个寄存器GDTR中。<br>
我们可以任意分配各个数据段的地址以及GDT本身存放的地址，但是段描述符、段选择子以及GDTR本身的结构是intel x86编程手册中所规定好的，下面是这些规定的好的结构

0. 段选择子

        RPL                        0 : 1
        TI                         2 : 2
        Index                      3 : 15

    0 : 1是指占据寄存器的0位和1位，2 ：2指占据第二位，后面的描述都会采用这种规则<br>
    RPL是权限位，表示当前查表的进程运行在何种特权级下，数值越低权限越高，我们暂且放置<br>
    TI表示段选择子用来选择哪个表，0表示GDT表、1表示LDT表，我们暂且置1<br>
    Index是表的索引号，决定了选择哪个段描述符

*由于GDT将代码分为不同的段来描述，这些段的性质各不相同，因此它们对应的段描述符页各不相同，我们在这里暂且只列出目前用到的段，以后新加入的段对应的段描述符结构会在对应章节给出*

1. CS段（64位）

        Limit:                      0  : 15
        Base:                       16 ：31
        Base|TYPE|S|DPL|P：         0 ：7 | 8 : 11 | 12: 12 | 13 : 14 | 15 : 15
        Limit|AVL|L|DB|G|Base       16 : 19 | 20 : 20 | 21 : 21 | 22 : 22 | 23 : 23 | 24 : 31
    Base：段的基地址，由一个字（32位下，32bits，4字节）组成，意味着可以是保护模式下4G内存中任意一个位置(4G=2^32, 单位由G决定)<br>
    Limit: 段界限，20bits<br>
    G：段界限的单位，1bits，为0以字节为单位，为1以4kb为单位<br>
    P：存在标志位，1bits，为0说明内存不存在，当检查到为0而后面指令又要用到这段时可以引发缺页中断，然后中断将这段加载到内存，为1存在<br>
    DPL：4个特权级，2bits,代表的数值越小特权级越高，00，01，10，11<br>
    S：描述符类型，1bits, 为0表示系统段System Segment，为1表示代码段Code Segment<br>(关于什么是系统段什么是代码段见注释)[<sup>3</sup>](#ref-3)
    DB：默认操作数大小或指针大小，1bits,D为0表示对于代码段/数据段指令的偏移地址或操作数为16位，为1则32位<br>
    TYPE：表项所描述的数据段类型为代码段\数据段\系统调用门\TSS结构\LDT\...，4bits, 并且根据S的值为0还是为1对于位的解释不同,，较为复杂，详情见手册<br>
    L：标志64位代码段<br>
    AVL：给操作系统的保留位<br>

2. DS段

###  GDT实现

知道了这些段描述符的结构，我们使用代码来为它们创建合适的数据结构

1. 对于段选择子，我们使用宏定义直接定义位的数值来描述它，我们先设置内核数据段和内核代码段的段选择子，在后面初始化GDT时会用到，intel规定GDT表的第0项保留，所以表项从下标1开始设置，TI位为0，权限位暂且不设置

    一个GDT表项为64字节，所以索引值需要左移三位，这三位留给RPL（2位）和TI（1位），所以在宏定义中需要*8，相当于左移三位，而在GDT表中需要/8，相当于右移三位取索引值作为数组下标

        #define KERNEL_SELECTOR_CS  (1 * 8)

        #define KERNEL_SELECTOR_DS  (2 * 8)


2. 对于段描述符，我们使用一个结构来描述它，根据上面给出的结构信息，我们使用5个变量来描述这些位，段描述的位排列应该是紧凑的，所以指定编译器不启用结构体对齐

        #pragma pack(1)
        typedef struct _segment_desc_t
        {
            uint16_t limit15_0;
            uint16_t base15_0;
            uint8_t base23_16;
            uint16_t attr;
            uint8_t base31_24;
        }segment_desc_t;
        #pragma pack()

3. GDT表是段描述符的集合，我们使用一个数组来描述它

        #define GDT_TABLE_SIZE  256
        static segment_desc_t gdt_table[GDT_TABLE_SIZE];

    由于我们配置的变成环境是单处理器，因此整个操作系统中只有一个全局GDT表，所以定义为静态类型，存储持续性为全局，链接性为内部

4. 对于属性字段，我们不想每次设置属性时查找intel编程手册，因此我们使用宏来表示属性字段的值，保留位不进行设置

        #define SEG_G (1 << 15) //控制limit的单位 单位4kb 1
        #define SEG_D (1 << 14) //控制代码段和栈段是16位或32位， 32位 1
        //L
        //AVL保留位
        //limit4位
        #define SEG_P_PRESENT   (1 << 7)    //段描述符存在标志，存在 1

        #define SEG_DPL0 (0 << 5)  //段描述符指向的段的特权级 4位
        #define SEG_DPL3 (3 << 5)  //段描述符指向的段的特权级

        #define SEG_S_SYSTEM    (0 << 4)    //决定代码是系统段 0,中断门TSS等
        #define SEG_S_NORMAL    (1 << 4)    //还是应用段 1，代码段数据段

        //TYPE 4位,除了以下的设置还能设置是否可扩展和一致性
        #define SEG_TYPE_CODE   (1 << 3)    //第三位决定是代码段 1
        #define SEG_TYPE_DATA   (0 << 3)    //还是数据段 0
        #define SEG_TYPE_TSS    (9 << 0)    //根据手册的设置TSS的TYPE值

        #define SEG_TYPE_RW     (1 << 1)    //对于数据段为1可读写，对于代码段为1是可读取执行为0是只能执行

### 接口实现

1. 一些接口声明 
由于GDT是全局唯一的，并且后面需要频繁设置，因此我们需要提供接口供后面使用

        //段描述符表项设置
        void segment_desc_set(int selector, uint32_t base, uint32_t limit, uint16_t attr);

        //表初始化
        void init_gdt(void);

        //在GDT表中为TSS选择子找空闲位置,返回选择子
        int gdt_alloc_desc();

        void switch_to_tss(int tss_sel);

        //释放描述符
        void gdt_free_sel(int sel);

*暂且提供以上接口，工具接口的实现就不在这里列出，感兴趣的请查看本人github*

2. 初始化函数的定义 
后面内核代码需要加载进内存的代码段，运行时需要使用内存的数据段，因此我们首先设置内核代码段和内核数据段

        void init_gdt(void)
        {
            for(int i=0; i< GDT_TABLE_SIZE; i++)
                segment_desc_set((i << 3), 0, 0, 0);

            segment_desc_set(KERNEL_SELECTOR_CS, 0, 0xFFFFFFFF, SEG_P_PRESENT | SEG_DPL0 | SEG_S_NORMAL | SEG_TYPE_CODE | SEG_TYPE_RW | SEG_D | SEG_G);

            segment_desc_set(KERNEL_SELECTOR_DS, 0, 0xFFFFFFFF, SEG_P_PRESENT | SEG_DPL0 | SEG_S_NORMAL | SEG_TYPE_DATA | SEG_TYPE_RW | SEG_D | SEG_G);

            lgdt((uint32_t)gdt_table, sizeof(gdt_table));
        }

    由于是逻辑地址采用平坦模型，代码段和数据段都从地址0开始，大小为4GB，最大地址为0xFFFFFFFF，注意，在GDT中设置的是逻辑地址而不是真正的物理地址，因此不必担心数据被覆盖<br>
    段存在标志应设为1、特权级为设置为数值0，即最高特权级<br>
    段类型设置为1，表示应用段，为0表示系统段<br>
    TYPE位的值根据段类型的值决定，内核代码段和内核数据段都为应用段，但是它们的TYPE不同<br>
    段读写权限都设置为可读写，具体权限需要结合特权级，特权级字段函数里没有设置，默认初始化为0，最高特权级(需要注意的是这里的权限是指访问段资源的进程所需的最低权限，而段选择子中的权限是访问段的进程本身的权限，具体的权限系统后面会涉及)<br>
    描述段大小需要一个单位，因此段描述符中有单位控制位，我们设为1表示单位是4kb，这与物理页的单位一致<br>
    描述代码段和栈的位数也需要一个控制位，我们是32位系统，设置为1，如果是16位系统设置为0


    以上，我们设置好了必须设置的位的值，剩余的其他位默认置0，对于保留位来说任何值都不影响。

    最后使用封装内联汇编的函数，调用lgdt指令存储GDT表地址到寄存器GDTR，当内核运行时cpu会主动使用GDTR中保存的位置，将GDT从磁盘的这个位置加载到内存






    









