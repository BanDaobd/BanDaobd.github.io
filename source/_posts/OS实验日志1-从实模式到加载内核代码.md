---
title: OS实验日志1-从实模式到加载内核代码
date: 2023-04-23 00:03:17
tags:
---

# 实验日志1
*此篇不是重点部分但涉及较多汇编和硬件知识，因此只解释引导流程的理论以及功能函数的理论和实现，函数调用流程以及代码参考本人博客中另一篇文章，[正式进入内核之前函数跳转流程](https://bandaobd.github.io/2023/04/23/正式进入内核之前函数跳转流程/)*

## bootloader

### boot启动流程

1. 设置段寄存器和栈空间

  	        .code16
 	        .text
	        .global _start
        _start:	
	        // 数据段寄存器置为0
	        mov $0, %ax
	        mov %ax, %ds
	        mov %ax, %ss
	        mov %ax, %es
	        mov %ax, %fs
	        mov %ax, %gs

	        // 使用0x7c00之前的空间作栈，大约有30KB的RAM，足够boot和loader使用
	        mov $_start, %esp

2. 显示加载完成提示

	        // 显示boot加载完成提示
	        mov $0xe, %ah
	        mov $'L', %al
	        int $0x10

3. 从磁盘加载loader进内存

    使用int13中断，流程固定

	    // 从磁盘加载loader进内存，只支持磁盘1
        read_loader:
	        mov $0x8000, %bx	// 读取到的内存地址
	        mov $0x2, %cx		// ch:磁道号，cl起始扇区号
	        mov $0x2, %ah		// ah: 0x2读磁盘命令，BIOS读磁盘从1号开始读
	        mov $64, %al		// al: 读取的扇区数量, 必须小于128，暂设置成32KB=64*512byte
	        mov $0x0080, %dx	// dh: 磁头号，没有跨磁头为0，dl驱动器号0x80(磁盘1)
	        int $0x13
	        jc read_loader		//jump if carry，读完磁盘后返回这里，为1读取失败循环读

	        // 跳转至c部分执行，再由c部分做一些处理
	        jmp boot_entry

4. 设置boot结束标志段

	        //设置结束标志结束段
	        .section boot_end, "ax"
        boot_sig: .byte 0x55, 0xaa

5. 跳转到loader

        __asm__(".code16gcc");


        #define LOADER_START_ADDR 0x8000

        void boot_entry(void) {
            //把地址强转成函数指针指向一个地址，将地址作为函数跳转
            ((void (*)(void))LOADER_START_ADDR)();
        } 



### loader引导流程

1. 在loader中检测内存容量

    * 硬件检测

        检测内存容量
        检测硬盘数量等

    * 目的:查找系统可用内存段

    * 方法 使用int0x15中断，中断函数的参数使用寄存器传递，每次调用前向寄存器输入一些值，调用结束后就能向寄存器返回一些值，EAX设置成E820表示遍历主机上全部内存，每次调用返回ARDS（地址范围描述符），描述符结构包括检测地址的起始位置和结束位置，本段内存类型等

    * 使用结构体数组存储内存段的起始位置和结束位置

            ARDS结构            以8字节为单位
            起始地址的低32位    0~3
            起始地址的高32位    4~7
            内存长度的低32位    8~11
            内存长度的高32位    12~15
            TYPE                15~16

            TYPE值为1表示内存系统可用
            TYPE值为2表示内存保留或正在使用，系统不可用
            TYPE值为其它时属于undefine，我们不使用
            
        ARDS存放在寄存器中，结构体不使用内存对齐而是自然排列用以存储ARDS返回值，第一个结构体存放每次调用的ARDS，第二个结构体数组存储所有返回的可用内存段的起始地址与结束地址

            #define BOOT_RAM_REGION_MAX     10

            typedef struct SMAP_entry {
            uint32_t BaseL; // base address uint64_t
            uint32_t BaseH;
            uint32_t LengthL; // length uint64_t
            uint32_t LengthH;
            uint32_t Type; // entry Type
            uint32_t ACPI; // extended
            }__attribute__((packed)) SMAP_entry_t;

            typedef struct _boot_info_t
            {
                struct
                {
                uint32_t start;
                uint32_t end;
                }ram_region_cfg[BOOT_RAM_REGION_MAX];
    
            int ram_region_count;

            }boot_info_t;

    * 流程


        第一次调用时，ES：DI存储保存读取的信息的存储位置

        EBX设为0

        EDX设为0x534D4150

        EAX设为0xE820

        ECX设为24

        执行INX 0x15

        返回结果：EAX=0x534D4150，CF标志位清0，EBX设为某个数值供下次调用，CL=读取的字节数

        后续调用

        EDX设为0x534D4150

        EAX重新设为0xE820

        ECX设为24

        执行INX 0x15

        返回结果：EAX=0x534D4150，CF标志位清0，若EBX=0表明读取完毕，否则当前的条目有效

        

    * 代码

        内联汇编语句参考说明[内联汇编](https://bandaobd.github.io/2023/04/23/内联汇编/)

            static void detect_memory(void)
            {
                uint32_t contID=0;
                uint32_t signature, bytes;
                SMAP_entry_t smap_entry;

                show_msg("detecting memory:");

                boot_info.ram_region_count = 0;
                for(int i=0;i<BOOT_RAM_REGION_MAX;i++)
                {

                    SMAP_entry_t *entry=&smap_entry;

                    __asm__ __volatile__
                    (
                        "int  $0x15"
			            : "=a"(signature), "=c"(bytes), "=b"(contID)
			            : "a"(0xE820), "b"(contID), "c"(24), "d"(0x534D4150), "D"(entry)
                    );

                    if(signature != 0x534D4150)
                    {
                        show_msg("failed\r\n");
                        return;
                    }

                    if (bytes > 20 && (entry->ACPI & 0x0001) == 0){
			            continue;
		            }

                    if(entry->Type==1)
                    {
                        boot_info.ram_region_cfg[boot_info.ram_region_count].start = entry->BaseL;
                        boot_info.ram_region_cfg[boot_info.ram_region_count].end = entry->LengthL;
                        boot_info.ram_region_count++;
                    }

                    if(contID == 0)
                    {
                        break;
                    }

                }

                show_msg("detect complete!\r\n");
            }

2. 在屏幕上显示检测结果
    
    * 方法:调用int10中断把值输入到指定寄存器

            static void show_msg(const char * msg)
            {
                char c;

                while((c = *msg++) != '\0')
                {
                    __asm__ __volatile__
                    (
	                "mov $0xe, %%ah\n\t"
	                "mov %[ch], %%al\n\t"
	                "int $0x10"::[ch]"r"(c)
                    );
                }
            }

3. 做加载操作系统到内存前的准备

    * 关中断

        因为是16位模式到32位模式的切换，需要防止后面的切换过程被中断打断导致不可预知的问题

    * 打开a20地址线（操作是固定的）

    * 清空流水线

        CPU将指令分为取指、译码、执行、写回，这些指令通过流水线进行，如果不清空流水线，那么在远跳转指令在执行阶段时，对指令的解释变为32位，然而下两条16位指令在此之前已经取指，分别处于译码和取指阶段，此时继续在32位模式下执行16位指令会出现错误。因此我们通过远跳转清空流水线，防止出现这种情况。

    * 初始化GDT表

        需要一个临时GDT表，先给出实现，具体后面解释

            uint16_t gdt_table[][4] = {
                {0, 0, 0, 0},
                {0xFFFF, 0x0000, 0x9A00, 0x00CF},
                {0xFFFF, 0x0000, 0x9200, 0x00CF},
            };
        
        上面四步的代码

            static void enter_protect_mode(void)
            {
                cli();

                //加载A20地址线
                uint8_t v=inb(0x92);
                outb(0x92, v | 0x2);

                // 加载GDT。由于中断已经关掉，IDT不需要加载
                lgdt((uint32_t)gdt_table, sizeof(gdt_table));

                //cr0置一
                uint32_t cr0 = read_cr0();
                write_cr0(cr0 | (1 << 0)); 

    
                //长跳转到32位入口处，清空流水线中的16位指令
                far_jump(8, (uint32_t)protect_mode_entry);
            }


    * 设置还在16位模式下的寄存器

                .code32
	            .text
	            //外部链接性
	            .global protect_mode_entry
	            .extern load_kernel
            //上面的loader_entry跳到这里
            protect_mode_entry:
	            //对还处于16位的寄存器随便做一些修改，清空之前的值
	            mov $16, %ax
	            mov %ax, %ds
	            mov %ax, %ss
	            mov %ax, %es
	            mov %ax, %fs
	            mov %ax, %gs

4. 正式进入32位模式

	            //真正进入32位模式
	            jmp $8, $load_kernel

5. 32位保护模式下的loader


    *   从磁盘加载操作系统文件

        由于保护模式下无法使用BIOS，我们使用使用LBA读取磁盘

        LBA48模式读取磁盘，流程较为固定

        主硬盘控制器具有 8 个端口，端口号从0x01f0到0x01f7。

        端口号：0x01f2，表示读取的扇区数

        设置起始 LBA 扇区号

        逻辑扇区编址方法 LBA28，用28位表示逻辑扇区号
        端口号：0x01f3，表示起始 LBA 扇区号 7~0 位
        端口号：0x01f4，表示起始 LBA 扇区号 15~8 位
        端口号：0x01f5，表示起始 LBA 扇区号 23~16 位
        端口号：0x01f6，
        低 4 位表示起始 LBA 扇区号的 27~24 位
        第 4 位指示硬盘号，0表示主盘，1表示从盘
        高 3 位是“111”；第 6 位为 1：表示 LBA 模式，为 0：表示 CHS 模式


        设置命令

        端口号：0x01f7，既是命令端口，又是状态端口
        当端口值为0x20时，表示读

        等待读写操作完成

        端口号：0x01f7，硬盘读写期间此端口表示硬盘状态
        第 0 位，1表示前一个命令执行错误，具体原因见端口号：0x01f1
        第 3 位，1表示硬盘已准备好交换数据
        第 7 位，1表示硬盘忙

        连续取出数据

        端口号：0x01f0，数据端口，这是一个 16 bit 端口。

        错误信息

        端口号：0x01f1，包含硬盘驱动器最后一次执行命令后的状态，错误原因。

            static void read_disk(uint32_t sector,uint32_t sector_count, uint8_t *buf)
            {
                outb(0x1F6, 0xE0);
                outb(0x1F2, (uint8_t)(sector_count >> 8));
                outb(0x1F3, (uint8_t)(sector >> 24));
                outb(0x1F4, 0);
                outb(0x1F5, 0);

                outb(0x1F2, (uint8_t)sector_count); 
                outb(0x1F3, (uint8_t)sector); 
                outb(0x1F4, (uint8_t)(sector >> 8)); 
                outb(0x1F5, (uint8_t)(sector >> 16)); 

                outb(0x1F7, 0x24);

                //每次从磁盘读取16位的数据
                //这里读取可能会发生一个错误没处理
                uint16_t *data_buf = (uint16_t *)buf;
                //读取磁盘前判断磁盘忙
                while(sector_count--)
                {
                    //没就绪原地等待
                    while((inb(0x1F7)  & 0x88) != 0x8)
                    {

                    }

                    //读扇区，一个扇区512字节
                    for(int i = 0;i<SECTOR_SIZE / 2;i++)
                    {
                        *data_buf++ = inw(0x1F0);
                    }
                }
            }

6. 内核中的相关操作

    * 如何加载内核映像文件

        我们使用lds链接脚本控制加载位置
    
        如果不做任何处理就从loader跳到内存0x100000中，不会出现任何问题，因为我们在kernel文件夹的cmakefile中将源码编译成二进制文件，只有代码和数据信息，放在image目录下

            . = 100000
            . text
            . rodata
            . = 200000
            .data

        . text和. rodata本来占空间不大，但是. = 200000强制把.data放在2M位置处，导致链接后文件过大，从磁盘加载到内存会慢，并且. rodata到. = 200000之间的空间会被清空，还会有权限问题

        我们可以用虚拟内存设置这些段的可读或可读可写，但是因为二进制文件又中并未标明段配置信息，所以不能直接用. = 200000指定位置，内核也就不能对数据进行设置

        因此我们不链接成二进制文件，而是链接成elf文件，大小合适又有配置信息又没有数据清空的问题

        关于elf文件见[elf文件](https://bandaobd.github.io/2023/04/23/elf文件/)

    * 从二进制文件改为elf文件后不能简单跳转到0x100000位置处，需要做elf解析

        我们将elf文件放到0x100000（1M）处，解析后的代码和数据放到0x10000（64kb）处，因此改一下链接脚本，loader里也跳到0x10000处（并且由elf解析函数解析出内核代码地址）

            //用于解析并加载elf文件到内存位置处
            static uint32_t reload_elf_file(uint8_t * file_buffer)
            {
                ELF32_Ehdr *elf_hdr =(ELF32_Ehdr *)file_buffer;
                //粗略检查文件头是否合法，检查e_ident[]标志符
                if (elf_hdr->e_ident[0] != 0x7F || elf_hdr->e_ident[1] != 'E' 
                    || elf_hdr->e_ident[2] != 'L' || elf_hdr->e_ident[3] != 'F')
                    return 0;
    
                for(int i=0;i < elf_hdr->e_phnum; i++)
                {
                    ELF32_Phdr *phdr = (ELF32_Phdr *)(file_buffer + elf_hdr->e_phoff)+i;

                    if(phdr->p_type != PT_LOAD)
                        continue;

                    //从内存位置offset拷到内存位置addr, 拷filesz字节
                    uint8_t *src = file_buffer + phdr->p_offset;
                    uint8_t *dest = (uint8_t *)phdr->p_paddr;
                    for(int j = 0; j < phdr->p_filesz; j++)
                        *dest++ = *src++;

                    //填0
                    dest = (uint8_t *)(phdr->p_paddr + phdr->p_filesz);
                    int n=(phdr->p_memsz - phdr->p_filesz);
                    for(int j=0 ;j < n; j++)
                        *dest++ =0;
                }

                //返回程序入口地址
                return elf_hdr->e_entry;

            }



    * 如何传递内存检测信息

        由于内核初始化也需要内存信息，boot内存检测得到的的信息需要通过loader传递

        1. 如果将boot_info写入到固定地址，kernel去取，需要事先约定，并且后续存储规划发生变化时要同时调整，过于麻烦

        2. 传递方法
        
        根据函数调用时栈的行为，在汇编中手动对这种行为进行操作达成传递参数的目的，在内核加载函数中使用
        
            ((void (*)(boot_info_t *))kernel_entry)(&boot_info);
        
        把地址强转成函数指针，编译器会认为是一个函数调用，从而跳转到_start处，所以我们将存储了之前的内存监测信息的全局变量boot_info作为参数传递给_startr函数，它作为参数被压入栈中，
        
        之后我们在汇编代码中通过栈基址寄存器ebp+偏移量取boot_info的值到寄存器%eax中，然后调用kernel_init函数，由于调用的新的函数会会把接下来的栈顶值作为参数，我们只需要在调用之前手动将%eax压栈即可

            void load_kernel(void)
            {
                //从磁盘读取内核代码
                //内核放在loader后面，100扇区起始，大小500扇区250kb
                read_disk(100, 500, (uint8_t *)SYS_KERNEL_LOAD_ADDR);

                //把elf文件加载到0x100000后还要从这个位置解析并加载到0x10000，
                //因为elf文件需要解析出各个段地址（段地址在脚本中聚合了），并且要得到程序代码入口地址（入口地址不是文件的第一个字节）
                uint32_t kernel_entry = reload_elf_file((uint8_t *)SYS_KERNEL_LOAD_ADDR);

                if(kernel_entry == 0)
                    die(-1);

                //不加载二进制文件而是加载elf文件，降低文件大小并且能够得知文件数据信息，方便做读写权限，从kernel_entry进入
                //kernel是解析出来的内核程序入口地址
                ((void (*)(boot_info_t *))kernel_entry)(&boot_info);
            }

        其中 die() 是一个简单的停机函数
