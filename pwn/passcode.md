# Passcode

题目来源：https://pwnable.kr

这道题同样连接到服务器，随后可以看到服务器上有一个可执行的 passcode 文件和一个可读的 passcode.c 文件，需要借助这个可执行文件打开 flag 文件。

先看一下 passcode.c 的代码：

```c
#include <stdio.h>
#include <stdlib.h>                                                                       

void login(){
	int passcode1;
	int passcode2;                                                                           
	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);                                                                           
	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
	scanf("%d", passcode2);                                                                   
	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
		printf("Login OK!\n");
		system("/bin/cat flag");
	}
	else{
		printf("Login Failed!\n");
		exit(0);
	}
}                                                                                          
void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}                                                                                                                           
int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");                                                                                  
	welcome();
	login();                                                                                                                         
	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;
}
```

主函数没有什么好分析的，主要的焦点是在 login 这个函数内，可以发现 login 的目的就是输入两个密码，随后判断这个密码是否符合要求，如果是正确的就可以获得 flag。但是问题就在于这个 scanf 语句中，因为参数没有加上 地址符号，所以盲目地输入会导致段错误的产生。

scanf 后的参数加上 & 是为了将输入放入到为 passcode1 预留的地址中，但是假设没有 & 符号，就相当于从预留地址中的值作为地址，往这个值代表的地址中写入数据，那么就很容易造成段错误了。其实最容易想到的就是是否在预留的地址中留下一个合法的地址值，这样就并不会造成段错误，所以这是我最开始想到的，来观察一下栈的情况。使用 objdump：

```asm
08048564 <login>:
 8048564:	55                   	push   %ebp
 8048565:	89 e5                	mov    %esp,%ebp
 8048567:	83 ec 28             	sub    $0x28,%esp
 804856a:	b8 70 87 04 08       	mov    $0x8048770,%eax
 804856f:	89 04 24             	mov    %eax,(%esp)
 8048572:	e8 a9 fe ff ff       	call   8048420 <printf@plt>
 8048577:	b8 83 87 04 08       	mov    $0x8048783,%eax
 804857c:	8b 55 f0             	mov    -0x10(%ebp),%edx
 804857f:	89 54 24 04          	mov    %edx,0x4(%esp)
 8048583:	89 04 24             	mov    %eax,(%esp)
 8048586:	e8 15 ff ff ff       	call   80484a0 <__isoc99_scanf@plt>
 804858b:	a1 2c a0 04 08       	mov    0x804a02c,%eax
 8048590:	89 04 24             	mov    %eax,(%esp)
 8048593:	e8 98 fe ff ff       	call   8048430 <fflush@plt>
 8048598:	b8 86 87 04 08       	mov    $0x8048786,%eax
 804859d:	89 04 24             	mov    %eax,(%esp)
 80485a0:	e8 7b fe ff ff       	call   8048420 <printf@plt>
 80485a5:	b8 83 87 04 08       	mov    $0x8048783,%eax
 80485aa:	8b 55 f4             	mov    -0xc(%ebp),%edx
 80485ad:	89 54 24 04          	mov    %edx,0x4(%esp)
 80485b1:	89 04 24             	mov    %eax,(%esp)
 80485b4:	e8 e7 fe ff ff       	call   80484a0 <__isoc99_scanf@plt>
 80485b9:	c7 04 24 99 87 04 08 	movl   $0x8048799,(%esp)
 80485c0:	e8 8b fe ff ff       	call   8048450 <puts@plt>
 80485c5:	81 7d f0 e6 28 05 00 	cmpl   $0x528e6,-0x10(%ebp)
 80485cc:	75 23                	jne    80485f1 <login+0x8d>
 80485ce:	81 7d f4 c9 07 cc 00 	cmpl   $0xcc07c9,-0xc(%ebp)
 80485d5:	75 1a                	jne    80485f1 <login+0x8d>
 80485d7:	c7 04 24 a5 87 04 08 	movl   $0x80487a5,(%esp)
 80485de:	e8 6d fe ff ff       	call   8048450 <puts@plt>
 80485e3:	c7 04 24 af 87 04 08 	movl   $0x80487af,(%esp)
 80485ea:	e8 71 fe ff ff       	call   8048460 <system@plt>
 80485ef:	c9                   	leave  
 80485f0:	c3                   	ret    
 80485f1:	c7 04 24 bd 87 04 08 	movl   $0x80487bd,(%esp)
 80485f8:	e8 53 fe ff ff       	call   8048450 <puts@plt>
 80485fd:	c7 04 24 00 00 00 00 	movl   $0x0,(%esp)
 8048604:	e8 77 fe ff ff       	call   8048480 <exit@plt>
```

结合源代码推测，在地址 0x804857c 的位置处就分配了相应的空间，这个位置在 ebp-0x10 处，如果希望中间的值是一个合法值，就必须要在之前覆盖这个位置，根据代码逻辑，能做到这一点的就只有 welcome 函数，观察一下 welcome 函数的代码：

```asm
08048609 <welcome>:
 8048609:	55                   	push   %ebp
 804860a:	89 e5                	mov    %esp,%ebp
 804860c:	81 ec 88 00 00 00    	sub    $0x88,%esp
 8048612:	65 a1 14 00 00 00    	mov    %gs:0x14,%eax
 8048618:	89 45 f4             	mov    %eax,-0xc(%ebp)
 804861b:	31 c0                	xor    %eax,%eax
 804861d:	b8 cb 87 04 08       	mov    $0x80487cb,%eax
 8048622:	89 04 24             	mov    %eax,(%esp)
 8048625:	e8 f6 fd ff ff       	call   8048420 <printf@plt>
 804862a:	b8 dd 87 04 08       	mov    $0x80487dd,%eax
 804862f:	8d 55 90             	lea    -0x70(%ebp),%edx
 8048632:	89 54 24 04          	mov    %edx,0x4(%esp)
 8048636:	89 04 24             	mov    %eax,(%esp)
 8048639:	e8 62 fe ff ff       	call   80484a0 <__isoc99_scanf@plt>
 804863e:	b8 e3 87 04 08       	mov    $0x80487e3,%eax
 8048643:	8d 55 90             	lea    -0x70(%ebp),%edx
 8048646:	89 54 24 04          	mov    %edx,0x4(%esp)
 804864a:	89 04 24             	mov    %eax,(%esp)
 804864d:	e8 ce fd ff ff       	call   8048420 <printf@plt>
 8048652:	8b 45 f4             	mov    -0xc(%ebp),%eax
 8048655:	65 33 05 14 00 00 00 	xor    %gs:0x14,%eax
 804865c:	74 05                	je     8048663 <welcome+0x5a>
 804865e:	e8 dd fd ff ff       	call   8048440 <__stack_chk_fail@plt>
 8048663:	c9                   	leave  
 8048664:	c3                   	ret 
```

welcome 函数的目的其实很简单，就是输入一个名字，不过很有意思地开辟了 100 个字节的空间，事实上我们可以看到在栈上开辟了 0x88 的空间，而这个 name 变量的地址在 ebp-0x70 处。也就是说中间差了 0x60 也就是 96 个字节，而因为 name 可以写 100 个字节，所以这个 passcode1 变量恰好和 name 变量重叠最后四个字节，这样就可以构造一个字符串，使得 scanf 函数是合法的。

---

以上是想到的，那么下面就是借鉴的，因为并没有想到这个思路。其实是经验问题，可以看到这个 login 函数中用到了 fflush 和很多 printf 这种动态库中的函数就应该警觉，这道题可能和 plt 与 got 有关。

plt 和 got 表是用来解决 PIC 问题，而动态链接中依靠这个机制来搜索需要使用的外部函数，由于在加载时会使用延迟加载的功能，所以 got 表应当是允许写入的，它需要在第一次执行该函数时将 got 中的跳转位置直接改写称目标函数的地址。

有了这个线索，也就是说我们可以通过 got 表项作为跳板跳转出去，跳转到哪呢？我们只希望执行完整的 system 函数，所以观察 login 的汇编代码能够知道，需要跳转到 0x80485e3。

这里最好的跳板时 fflush 函数，它只会被调用一次，而且就在第一个 scanf 下面，我们需要知道它的 got 表的地址。根据 plt 的性质，plt 中第一行代码必然是跳转到对应的 got 表项指向的那一行，我们可以查看一下地址：

```
08048430 <fflush@plt>:
 8048430:	ff 25 04 a0 04 08    	jmp    *0x804a004
 8048436:	68 08 00 00 00       	push   $0x8
 804843b:	e9 d0 ff ff ff       	jmp    8048410 <_init+0x30>
```

fflush 的 got 表项是 0x804a004，这样我们构造一个 name，有 96 个 a 和 0x804a004， 随后再输入 passcode1 为 system 函数的地址的十进制表示即可。

> 这里有个小技巧需要了解，python -c 可以运行单行的 python 脚本。

