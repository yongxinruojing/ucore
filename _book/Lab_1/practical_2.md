[TOC]

# 练习2

#### 使用qemu执行并调试lab1中的软件。

为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习：

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
2. 在初始化位置0x7c00设置实地址断点,测试断点正常。
3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。



```makefile
# 0. 调整gdbinit即可从CPU加电后第一条指令开始
file bin/kernel
set architecture i8086		# 配置为i8086
target remote:1234
			# CPU加电后执行的第一条指令为'0xfff0:' 'add %al,(%bx,%si)'
			# 输入 i r观察寄存器值有'eip 0xfff0 0xfff0
			# 	  				  cs 0xf000 61440'
			# 表示EIP值为0xFFF0，CS为0xF000因此CPU执行第一条指令为CS:IP = 0xF000:0xFFF0
			# 他是一个长跳转指令JMP F000:E05B
			# 输入'si'查看下一个指令，发现地址为0xe05b亦即CS:e05b
# 1. 在gdbinit文件加入
	
b *0x7c00					# 设置实地址断点为'0x7c00'
b kern_init
continue
define hook-stop			# 强制反汇编当前指令
x /2i $pc					# 从当前地址pc(亦即EIP：下条执行命令的内存地址)往后显示2条指令
end
# 2. 单步运行时发现到了init.c文件中的'while(1)'就死循环
# 3. 单步'nexti',断点正常，跳到了'while(1)'死循环
# 4. 通过meld "file1" "file2"进行文件内容对比，比较bootasm.S和 bootblock.asm发现几乎完全一样且.asm文件还给出了注解
		# 对比代码：发现按照bootblock.asm从上到下的代码运行
		# 从0x7c4a跳转到ox7cd1
		# 从0x7ce8跳转到0x7c72
		# ...
		# 到了0x7ccc就不动了,注释是等待硬盘准备
# 5. 设置断点为'0x7c09'，正常
```

**与答案对比**

答案有补充

~~~makefile
# 改写Makefile文件
	debug: $(UCOREIMG)
		$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
		$(V)sleep 2
		$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

# 在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中。
# 为防止qemu在gdb连接后立即开始执行，删除了`tools/gdbinit`中的`continue`行。
~~~

