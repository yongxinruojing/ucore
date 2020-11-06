[TOC]



# **练习0：填写已有实验**

本实验依赖实验1/2/3/4/5/6/7。请把你做的实验1/2/3/4/5/6/7的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6” /“LAB7”的注释相应部分。并确保编译通过。注意：为了能够正确执行lab8的测试应用程序，可能需对已完成的实验1/2/3/4/5/6/7的代码进行进一步改进。

```
trap.c
	lab8的内容有些不同暂时先不拷贝； 
	// There are user level shell in LAB8, so we need change COM/KBD interrupt processing.

找一下有没有需要更改的地方
find . -name "*.[chS]" -exec grep -Hn LAB8 {} \;
****************
/kern/process/proc.c:636:// this function isn't very correct in LAB8
./kern/trap/trap.c:254:        // There are user level shell in LAB8, so we need change COM/KBD interrupt processing.

寻找proc.c:636发现是lab8的练习，那不用管了
看trap.c:254
	case IRQ_OFFSET + IRQ_COM1:
    case IRQ_OFFSET + IRQ_KBD:
        // There are user level shell in LAB8, so we need change COM/KBD interrupt processing.
        c = cons_getc();
        {
          extern void dev_stdin_write(char c);
          dev_stdin_write(c);
        }
        break;
    不知道啥情况 先不管
	
最后别忘了proc增加的结构进行初始化struct files_struct *filesp; 
```

