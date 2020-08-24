# level2

题目来源：https://adworld.xctf.org.cn/

这道题的题面是要我们使用 ROP 的攻击手段去破解，但是我觉得题面的提示反而是一个坑，因为这道题根本不用 ROP 去破解，使用一般的栈的构造就可以破解这道题了。

那么先看一下这道题的代码，因为题目给的文件是 elf32 的，在 ida64 中无法反汇编成 c 伪代码，所以就直接看一下汇编代码来分析控制流：

```asm
.text:08048480                 lea     ecx, [esp+4]
.text:08048484                 and     esp, 0FFFFFFF0h
.text:08048487                 push    dword ptr [ecx-4]
.text:0804848A                 push    ebp
.text:0804848B                 mov     ebp, esp
.text:0804848D                 push    ecx
.text:0804848E                 sub     esp, 4
.text:08048491                 call    vulnerable_function
.text:08048496                 sub     esp, 0Ch
.text:08048499                 push    offset aEchoHelloWorld ; "echo 'Hello World!'"
.text:0804849E                 call    _system
.text:080484A3                 add     esp, 10h
.text:080484A6                 mov     eax, 0
.text:080484AB                 mov     ecx, [ebp+var_4]
.text:080484AE                 leave
.text:080484AF                 lea     esp, [ecx-4]
.text:080484B2                 retn
```

这是 main 函数部分的汇编代码，如果我们手写它的 C 伪代码可以写成如下：

```
main {
	vulnerable_function();
	system("echo 'hello world'");
}
```

所以看来秘诀藏在了 vulnerable_function() 中，继续查看这个函数的汇编代码：

```asm
.text:0804844B                 push    ebp
.text:0804844C                 mov     ebp, esp
.text:0804844E                 sub     esp, 88h
.text:08048454                 sub     esp, 0Ch
.text:08048457                 push    offset command  ; "echo Input:"
.text:0804845C                 call    _system
.text:08048461                 add     esp, 10h
.text:08048464                 sub     esp, 4
.text:08048467                 push    100h            ; nbytes
.text:0804846C                 lea     eax, [ebp+buf]
.text:08048472                 push    eax             ; buf
.text:08048473                 push    0               ; fd
.text:08048475                 call    _read
.text:0804847A                 add     esp, 10h
.text:0804847D                 nop
.text:0804847E                 leave
.text:0804847F                 retn
```

继续翻译这个汇编代码转换为 C 代码：

```
vulnerable_function() {
	system("echo Input:");
	read(0, buf, 0x100);
}
```

观察可以发现这个 buf 的长度一定比 0x100 短，因为在开辟栈帧的时候，buf 的位置是 rbp-0x88。如果使用 checksec 去检查这个程序的安全项会发现这个程序并没有打开 canary，也就是说可以构造缓冲区溢出攻击。

而这道题因为在题面里没有直接跳转拿到 shell 的函数，所以应该是要自己构造控制流，而这里有一个注意点，就是这个程序中都是使用 system 这个函数打印信息，而不是直接使用 puts 或者 printf，所以这里就是在提示我们使用 system 函数来获取 shell。

> 当使用 system 来构造栈帧的时候，要警觉这个程序中的数据段中有没有 /bin/sh 

查看该程序的数据段能够发现 /bin/sh 字符串的地址：

```
.data:0804A024 hint            db '/bin/sh',0
```

因此只需要构造缓冲区溢出漏洞，将返回地址改为 call system 语句的地址，然后在这一句语句上方构造参数，就是填充 sh 字符串的地址就可以了。