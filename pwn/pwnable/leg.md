# leg

题目来源：https://pwnable.kr/

这道题考察的是 arm 汇编的知识点，我查找了一些网站和材料，但是如果题目再复杂一点似乎还是不够用，有必要再积累一定：https://www.cnblogs.com/hilfloser/p/10516610.html

那么先看看代码来分析：

```c
#include <stdio.h>
#include <fcntl.h>
int key1(){
	asm("mov r3, pc\n");
}
int key2(){
	asm(
	"push	{r6}\n"
	"add	r6, pc, $1\n"
	"bx	r6\n"
	".code   16\n"
	"mov	r3, pc\n"
	"add	r3, $0x4\n"
	"push	{r3}\n"
	"pop	{pc}\n"
	".code	32\n"
	"pop	{r6}\n"
	);
}
int key3(){
	asm("mov r3, lr\n");
}
int main(){
	int key=0;
	printf("Daddy has very strong arm! : ");
	scanf("%d", &key);
	if( (key1()+key2()+key3()) == key ){
		printf("Congratz!\n");
		int fd = open("flag", O_RDONLY);
		char buf[100];
		int r = read(fd, buf, 100);
		write(0, buf, r);
	}
	else{
		printf("I have strong leg :P\n");
	}
	return 0;
}
```

main 函数的流程非常简单，重点就是一个判断语句 `(key1()+key2()+key3()) == key`。那么如果查看三个函数会发现都是使用 arm 汇编写的，但是很奇怪的是这道题还给了 leg 文件的汇编代码文件。仔细看一下 arm 函数代码，会发现有几个比较陌生的寄存器：

>R0-R3:用于函数参数及返回值的传递
R4-R6, R8, R10-R11:没有特殊规定，就是普通的通用寄存器
R7:栈帧指针(Frame Pointer).指向前一个保存的栈帧(stack frame)和链接寄存器(link register， lr)在栈上的地址。
R9:操作系统保留
R12:又叫IP(intra-procedure scratch )
R13:又叫SP(stack pointer)，是栈顶指针
R14:又叫LR(link register)，存放函数的返回地址。
R15:又叫PC(program counter)，指向当前指令地址。

像 PC 和 LR 这两个寄存器都是需要获取当前的地址的，那么先看一下汇编代码吧：

**key1：**

```asm
Dump of assembler code for function key1:                                                 0x00008cd4 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
0x00008cd8 <+4>:     add     r11, sp, #0
0x00008cdc <+8>:     mov     r3, pc
0x00008ce0 <+12>:    mov     r0, r3
0x00008ce4 <+16>:    sub     sp, r11, #0
0x00008ce8 <+20>:    pop     {r11}           ; (ldr r11, [sp], #4)
0x00008cec <+24>:    bx      lr 
```

r0 寄存器可以相当于 x86 中的 eax，这里看了一下汇编的地址分布可以判断这是 arm32，那么寄存器是 32 位宽度。所以这个函数返回值存放在 r0 中，而其值在地址 0x8cdc 处给出，理论上应该是指向下一条指令的地址。

> PC 的值是正在取指的指令的地址！！！

arm处理器有一个特性就是它是使用三级流水线，所以当执行第 n 条指令时其实是第 n+2 条指令正处于取值阶段。所以此时 PC 的值为 n+2 条指令的地址，这里即 0x8ce4。

**key3：**

```asm
(gdb) disass key3
Dump of assembler code for function key3:
0x00008d20 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
0x00008d24 <+4>:     add     r11, sp, #0
0x00008d28 <+8>:     mov     r3, lr
0x00008d2c <+12>:    mov     r0, r3
0x00008d30 <+16>:    sub     sp, r11, #0
0x00008d34 <+20>:    pop     {r11}           ; (ldr r11, [sp], #4)
0x00008d38 <+24>:    bx      lr   
```

lr 寄存器存放的是返回地址，这是一个函数，返回地址一定是跳转这个函数后的下一条指令，如果观察主函数的汇编，找到跳转指令 bl key3，很容易发现返回地址为：0x8d80

**key2：**

```asm
(gdb) disass key2
Dump of assembler code for function key2:
0x00008cf0 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
0x00008cf4 <+4>:     add     r11, sp, #0
0x00008cf8 <+8>:     push    {r6}            ; (str r6, [sp, #-4]!)
0x00008cfc <+12>:    add     r6, pc, #1
0x00008d00 <+16>:    bx      r6
0x00008d04 <+20>:    mov     r3, pc
0x00008d06 <+22>:    adds    r3, #4
0x00008d08 <+24>:    push    {r3}
0x00008d0a <+26>:    pop     {pc}
0x00008d0c <+28>:    pop     {r6}            ; (ldr r6, [sp], #4)
0x00008d10 <+32>:    mov     r0, r3
0x00008d14 <+36>:    sub     sp, r11, #0
0x00008d18 <+40>:    pop     {r11}           ; (ldr r11, [sp], #4)
0x00008d1c <+44>:    bx      lr
```

最困惑的是 key2，中间部分还好说，但是有一个 r6 的接入着实很是迷惑。结果后来查找里资料得知，这里有一个特殊的指令 bx，这个 bx 起到跳转的同时，还会起到状态切换的作用。在 arm 汇编中有 arm 和 thumb 两种状态，而在这里 arm 状态下 pc 是 32 位宽度，thumb 是 16 位宽度，并且切换到 thumb 状态后许多寄存器的作用也发生了变化(*这里具体还是要查找一下资料*)。

所以这里获取到的 PC 的值应当是 0x8d08。

最终将三者累加起来就是最后的答案了。