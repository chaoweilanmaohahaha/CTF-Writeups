# hello_pwn

题目来源：https://adworld.xctf.org.cn/

这道题是一道水题，而且不知道为啥题目里会提到段错误，但是我也没看到源码中会触发，不是很清楚，简单看一下代码，基本看一下就知道怎么做了。

```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  alarm(0x3Cu);
  setbuf(stdout, 0LL);
  puts("~~ welcome to ctf ~~     ");
  puts("lets get helloworld for bof");
  read(0, &unk_601068, 0x10uLL);
  if ( dword_60106C == 1853186401 )
    sub_400686();
  return 0LL;
}
```

unk_601068 是一个全局变量，像这个全局变量中读入了 0x10 长度的内容，但是很奇怪的是下面比较的是另一个全局变量 dword_60106c。然后看一下变量在数据段中的分布：

```
.bss:0000000000601068 unk_601068      db    ? ;               ; DATA XREF: main+3B↑o
.bss:0000000000601069                 db    ? ;
.bss:000000000060106A                 db    ? ;
.bss:000000000060106B                 db    ? ;
.bss:000000000060106C dword_60106C    dd ?                    ; DATA XREF: main+4A↑r
```

按照这个数据段的分布，unk_601068 变量占了 4 个字节，下面就是 dword_60106c，而题目中要求往 unk_601068 中读入 0x10 个字节，这样就造成了溢出，而只要读入的字符串溢出部分能够覆盖 dword_60106c 处并写入正确的值就可以调用 sub 函数，内容就是获取 flag。

最后注意写入整数时要注意大小端！