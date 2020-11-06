



[TOC]
# 练习1

理解通过make生产执行文件的过程
====

1. **ucore.img如何生成的(详细解释Makefile中每一条相关命令和命令参数的含义并说明产生的结果)**
2. **一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？**

### **问题1:**

1. 通过make V=获取make执行的命令，下面为笔者简化后的命令(只保留了主要部分，去除了部分参数)，按顺序输入了个人的简单理解

```shell
							# -c指只编译不链接：产生.o即obj文件不产生执行文件 -o指定输出文件名
							
+ cc kern/init/init.c   	
gcc -c kern/init/init.c -o obj/kern/init/init/o
							# 编译init.c为init.o，通过查看init.c文件发现其功能为：初始化控制台、物理内存管理、
							# 中断控制器、中断描述表、时钟中断、打开irq(中断请求)。
							
+ cc kern/libs/readline.c
gcc -c kern/libs/readline.c -o obj/kern/libs/readline.o
							# 编译readline.c为readline.o
							# readline - get a line from stdin
							# @prompt:        the string to be written to stdout
							# 此文件为一个函数：从标准输入获取一行作为标准输出

+ cc kern/libs/stdio.c
gcc -c kern/libs/stdio.c -o obj/kern/libs/stdio.o
							# 编译stdio.c为stdio.o
							# HIGH level console I/O  高级控制台I/O
							# 里面包含了标准输入输出函数

+ cc kern/debug/kdebug.c
gcc -c kern/debug/kdebug.c -o obj/kern/debug/kdebug.o
							# 编译kdebug.c为kdebug.o
							# debug information about a particular instruction pointer
							# 调试有关特定指令指针的信息
							
+ cc kern/debug/kmonitor.c
gcc -c kern/debug/kmonitor.c -o obj/kern/debug/kmonitor.c
							# 编译kmonitor.c为kmonitor.o
							# 简单的命令行内核监视器，可用于控制内核和以交互方式浏览系统
							
+ cc kern/debug/painc.c
gcc -c kern/debug/panic.c -o obj/kern/debug/panic.o
							# 编译painc.c为painc.o
							# '恐慌'程序：若出现无法解决的致命错误，就调用'恐慌'，打印出'panic:"message"'
							# 			 然后进入内核监视器
						
+ cc kern/driver/clock.o
gcc -c kern/driver/clock.c -o obj/kern/driver/clock.o
							# 支持与时间相关的硬件小工具-8253计时器，该计时器在IRQ-0上生成中断。

+ cc kern/driver/console.c
gcc -c kern/driver/console.c -o obj/kern/driver/console.o
							# 历史PC设计缺陷所必需的愚蠢I/O延迟例程

+ cc kern/driver/intr.c
gcc -c kern/driver/intr.c -o obj/kern/driver/intr.o
							# 控制irq中断的开关	irq：Interupt ReQuest中断请求

+ cc kern/driver/picirq.c
gcc -c kern/driver/picirq.c -o obj/kern/driver/picirq.o
							# two programmable interrupt controllers
							# 两个可编程中断控制器

+ cc kern/trap/trap.c
gcc -c kern/trap/trap.c -o obj/kern/trap/trap.o
							# Interrupt descriptor table
							# 中断描述符表的初始化
							# 初始化IDT到kern/trap/vectors.S中的每个入口点
							
+ cc kern/trap/trapentry.S
gcc -c kern/trap/trapentry.S -o obj/kern/trap/trapentry.o
							# 建立陷阱(异常)框架
							# vectors.S sends all traps here

+ cc kern/trap/vectors.S
gcc -c kern/trap/vectors.S -o obj/kern/trap/vectors.o
							# 将所有陷阱发送到trapentry.S

+ cc kern/mm/pmm.c
gcc -c kern/mm/pmm.c -o obj/kern/mm/pmm.o
							# 初始化物理内存管理physical memory management

+ cc libs/printfmt.c
gcc -c libs/printfmt.c -o obj/libs/printfmt.o
							# 格式输出

+ cc libs/string.c
gcc -c libs/string.c -o obj/libs/string.o
							# 定义了string的一些操作函数

+ ld bin/kernel
ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel ....（上面的所有.o文件）
							# ld为GUN Linker链接器
							# '-m elf_i386'表示输出文件的格式为elf_i386
							# '-nostdlib'指出不连接系统标准启动文件和标准库文件，只把指定的文件传递给连接器
							# '-T'指定所用的链接器脚本为'tools/kernel.ld'
							# '-o'指定输出文件的名称
							# 链接.o文件,最终生成可执行目标文件'bin/kernel'

+ cc boot/bootasm.S
gcc -c boot/bootasm.S -o obj/boot/bootasm.o
							# 此文件功能：启动CPU，选择32位保护模式并跳到bootmain.c

+ cc boot/bootmain.c
gcc -c boot/bootmain.c -o obj/boot/bootmain.o
							# 一个非常简单的启动加载程序，唯一的工作就是从第一个IDE硬盘启动ELF内核映像

+ cc tools/sign.c
gcc -c tools/sign.c -o obj/sign/tools/sign.o
gcc obj/sign/tools/sign.o -o bin/sign
							# 编译且链接为可执行文件bin/sign
							# 其功能为给定输入文件和输出文件地址：通过输入文件建立512字节的引导扇区文件放到
							# 输出文件地址
							# 建立步骤大概为：1、将输入文件扩展为512字节的文件
							# 				2、向文件最后两个字节输入2个地址'0x55'和'0xAA'
							# 				3、输出文件

+ ld bin/bootblock
ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7c00 obj/boot/bootasm.o 
obj/boot/bootmain.o -o obj/bootblock.o
							# '-N'指定读取/写入文本和数据段
							# '-e'使用指定的符号作为程序的初试执行点
							# start为bootasm.S中定义的运行引导程序bootloader的启动地址：0:7c00
							# -Ttext使用指定地址作为文本段的起始点
							# 链接'bootasm.o'和'bootmain.o',生成目标文件'bootblock.o'

							# 综上所述上面完成了kernel和bootblock.o文件的生成
							# 根据老师讲的生产bootloader和kernel,那么bootloader就是bootblock.o!!
							
							# 此处为上面sign的成功执行输出
'obj/bootblock.out' size: 472 bytes						# 输入文件为472字节大小的'obj/bootblock.out'文件
build 512 bytes boot sector: 'bin/bootblock' success!	# 输出512字节大小的'bin/bootblock'引导扇区文件
dd if=/dev/zero of=bin/ucore.img count=10000			# 使用虚拟设备零设备生成5120000字节的空白文件
10000+0 records in										# 一块512字节
10000+0 records out
5120000 bytes (5.1 MB) copied, 0.0610333 s, 83.9 MB/s
dd if=bin/bootblock of=bin/ucore.img conv=notrunc		# 将bootblock加载入ucore.img中,写到第一块
1+0 records in											# 'notrunc'指定不截短ucore.img文件
1+0 records out	
512 bytes (512 B) copied, 0.000120716 s, 4.2 MB/s
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc	# 将kernel加载入ucore.img中，'seek=1'
138+1 records in										# 指从512字节(第一块)后复制，亦即放到前面加载的引导块后
138+1 records out
70775 byres (71 kB) copied, 0.000346563 s, 204 MB/s


# 查找Makefile文件，找到创建ucore.img的部分
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)		# 定义变量UCOREIMG且将bin/ucore.img赋给UCOREIMG
$(UCOREIMG): $(kernel) $(bootblock)				# 构建UCOREIMG文件依赖kernel和bootblock
	$(V)dd if=/dev/zero of=$@ count=10000		
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
												# 'V := @'，V变量为'@'。'@'在前意味着输出后面信息到屏幕上
												# 'dd'用指定大小的块拷贝一个文件，并拷贝的同时进行指定的转换
												# 'if'为输入文件名，'of'为输出文件名
												# '$@'为目标文件亦即'UCOREIMG'即'bin/ucore.img'
												# '/dev/zero'为linux的虚拟设备——‘零’设备，可无限提供空字符
												# 0x00
												# 'conv=notrunc'指定输出文件不截短
												# 'seek=1'指定从第一个块之后加载			
$(call create_target,ucore.img)					# 创建目标文件'ucore.img'
```

**Makefile相关参考答案**

~~~makefile
bin/ucore.img
| 生成ucore.img的相关代码为
| $(UCOREIMG): $(kernel) $(bootblock)
|	$(V)dd if=/dev/zero of=$@ count=10000
|	$(V)dd if=$(bootblock) of=$@ conv=notrunc
|	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
|
| 为了生成ucore.img，首先需要生成bootblock、kernel
|
|>	bin/bootblock
|	| 生成bootblock的相关代码为
|	| $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
|	|	@echo + ld $@
|	|	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ \
|	|		-o $(call toobj,bootblock)
|	|	@$(OBJDUMP) -S $(call objfile,bootblock) > \
|	|		$(call asmfile,bootblock)
|	|	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) \
|	|		$(call outfile,bootblock)
|	|	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
|	|
|	| 为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign
|	|
|	|>	obj/boot/bootasm.o, obj/boot/bootmain.o
|	|	| 生成bootasm.o,bootmain.o的相关makefile代码为
|	|	| bootfiles = $(call listf_cc,boot) 
|	|	| $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
|	|	|	$(CFLAGS) -Os -nostdinc))
|	|	| 实际代码由宏批量生成
|	|	| 
|	|	| 生成bootasm.o需要bootasm.S
|	|	| 实际命令为
|	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
|	|	| 	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
|	|	| 	-c boot/bootasm.S -o obj/boot/bootasm.o
|	|	| 其中关键的参数为
|	|	| 	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
|	|	|	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
|	|	| 	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
|	|	| 	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
|	|	|	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
|	|	| 	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
|	|	| 	-I<dir>  添加搜索头文件的路径
|	|	| 
|	|	| 生成bootmain.o需要bootmain.c
|	|	| 实际命令为
|	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
|	|	| 	-fno-stack-protector -Ilibs/ -Os -nostdinc \
|	|	| 	-c boot/bootmain.c -o obj/boot/bootmain.o
|	|	| 新出现的关键参数有
|	|	| 	-fno-builtin  除非用__builtin_前缀，
|	|	|	              否则不进行builtin函数的优化
|	|
|	|>	bin/sign
|	|	| 生成sign工具的makefile代码为
|	|	| $(call add_files_host,tools/sign.c,sign,sign)
|	|	| $(call create_target_host,sign,sign)
|	|	| 
|	|	| 实际命令为
|	|	| gcc -Itools/ -g -Wall -O2 -c tools/sign.c \
|	|	| 	-o obj/sign/tools/sign.o
|	|	| gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
|	|
|	| 首先生成bootblock.o
|	| ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
|	|	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
|	| 其中关键的参数为
|	|	-m <emulation>  模拟为i386上的连接器
|	|	-nostdlib  不使用标准库
|	|	-N  设置代码段和数据段均可读写
|	|	-e <entry>  指定入口
|	|	-Ttext  制定代码段开始位置
|	|
|	| 拷贝二进制代码bootblock.o到bootblock.out
|	| objcopy -S -O binary obj/bootblock.o obj/bootblock.out
|	| 其中关键的参数为
|	|	-S  移除所有符号和重定位信息
|	|	-O <bfdname>  指定输出格式
|	|
|	| 使用sign工具处理bootblock.out，生成bootblock
|	| bin/sign obj/bootblock.out bin/bootblock
|
|>	bin/kernel
|	| 生成kernel的相关代码为
|	| $(kernel): tools/kernel.ld
|	| $(kernel): $(KOBJS)
|	| 	@echo + ld $@
|	| 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
|	| 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
|	| 	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
|	| 		/^$$/d' > $(call symfile,kernel)
|	| 
|	| 为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o
|	|	kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
|	|	trapentry.o vectors.o pmm.o  printfmt.o string.o
|	| kernel.ld已存在
|	|
|	|>	obj/kern/*/*.o 
|	|	| 生成这些.o文件的相关makefile代码为
|	|	| $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
|	|	|	$(KCFLAGS))
|	|	| 这些.o生成方式和参数均类似，仅举init.o为例，其余不赘述
|	|>	obj/kern/init/init.o
|	|	| 编译需要init.c
|	|	| 实际命令为
|	|	|	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
|	|	|		-gstabs -nostdinc  -fno-stack-protector \
|	|	|		-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
|	|	|		-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \
|	|	|		-o obj/kern/init/init.o
|	| 
|	| 生成kernel时，makefile的几条指令中有@前缀的都不必需
|	| 必需的命令只有
|	| ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
|	| 	obj/kern/init/init.o obj/kern/libs/readline.o \
|	| 	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
|	| 	obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
|	| 	obj/kern/driver/clock.o obj/kern/driver/console.o \
|	| 	obj/kern/driver/intr.o obj/kern/driver/picirq.o \
|	| 	obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
|	| 	obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
|	| 	obj/libs/printfmt.o obj/libs/string.o
|	| 其中新出现的关键参数为
|	|	-T <scriptfile>  让连接器使用指定的脚本
|
| 生成一个有10000个块的文件，每个块默认512字节，用0填充
| dd if=/dev/zero of=bin/ucore.img count=10000
|
| 把bootblock中的内容写到第一个块
| dd if=bin/bootblock of=bin/ucore.img conv=notrunc
|
| 从第二个块开始写kernel中的内容
| dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```
~~~



#### 综上所述'ucore.img'的生成过程如下

+ 1. 首先编译但不链接'kernel'生成所需的文件后通过'kernel.ld'进行链接生成目标文件'kernel'
  2. 以相同的方式生成目标文件'bootblock.o'后通过'sign'文件对其扩展到512字节，并给倒数两个字节加入'0x55'和'0xAA'两个地址完成引导扇区文件'bootblock'的生成
  3. 前两个文件是生成'ucore.img'所依赖的文件，前两个完成后创建5120000字节的'ucore.img'文件，然后依次拷贝'bootblock.h'和'kernel'两个文件到'ucore.img'文件中。
  4. 生成'ucore.img'。



### **问题2:**

分析问题1的过程笔者探视过了'sign.c'文件，其为制定主引导扇区的文件，下面列出其中的关键代码：

```c
if (st.st_size > 510) {			/*若大于510返回出错提示,因为我后面还有加两个标志*/
        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
        return -1;
    }
    char buf[512];
    memset(buf, 0, sizeof(buf));		/*初始化0*/
    FILE *ifp = fopen(argv[1], "rb");	/*以二进制读文件*/
    int size = fread(buf, 1, st.st_size, ifp);		/*从文件中读取并存入buf中，每个数据块为1字节*/
    if (size != st.st_size) {						/*若读取后大小不一致，输出错误读取失败*/
        fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
        return -1;
    }
    fclose(ifp);
    buf[510] = 0x55;				/*设定秩[510]为0x55*/
    buf[511] = 0xAA;				/*设定秩[511]为0xAA*/
    FILE *ofp = fopen(argv[2], "wb+");/*读写方式打开或建立一个二进制文件*/
    size = fwrite(buf, 1, 512, ofp);/*每次1字节，写入文件512字节大小*/
```

#### 问题2总结

+ 1. 大小512字节
  2. 最后两个字节依次为'0x55'、'0xAA'





