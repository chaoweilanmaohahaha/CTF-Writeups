# cgpwn2

题目来源：https://adworld.xctf.org.cn/

这道题的题面是要寻找一个字符串能够获取 flag，其实从这里题目的尿性来看都是拿到控制台的权限，然后就能查看 flag，所以盲猜一下需要获取的字符串就是 /bin/sh。前面确实有题目在数据段提供了这个字符串，但是这道题里面并没有提供，所以需要答题者自己构造。

先看一下程序，main 函数只是调用了 hello 函数，所以就此掠过，而 pwn 函数中就是调用了 system 函数，这个举动印证了上面的猜想，只要想办法获取 /bin/sh 字符串。

下面分析一下 hello 函数：

```C
char *hello()
{
  char *v0; // eax
  signed int v1; // ebx
  unsigned int v2; // ecx
  char *v3; // eax
  char s; // [esp+12h] [ebp-26h]
  int v6; // [esp+14h] [ebp-24h]

  v0 = &s;
  v1 = 30;
  if ( (unsigned int)&s & 2 )
  {
    *(_WORD *)&s = 0;
    v0 = (char *)&v6;
    v1 = 28;
  }
  v2 = 0;
  do
  {
    *(_DWORD *)&v0[v2] = 0;
    v2 += 4;
  }
  while ( v2 < (v1 & 0xFFFFFFFC) );
  v3 = &v0[v2];
  if ( v1 & 2 )
  {
    *(_WORD *)v3 = 0;
    v3 += 2;
  }
  if ( v1 & 1 )
    *v3 = 0;
  puts("please tell me your name");
  fgets(name, 50, stdin);
  puts("hello,you can leave some message here:");
  return gets(&s);
}
```

这个函数在上面有一大段花里胡哨的代码，一开始以为这段代码可能会影响到下面获取字符串的逻辑，但是仔细看了一下发现就是一些无关的计算代码，与目标代码一点关系都没有，纯属混淆。最关键的就只有最后的四行。首先输入一个名字，随后输入一个信息。

name 变量是一个全局变量，定义在 bss 段也就是数据段中；而 s 是一个局部变量，定义在栈上。因为在最后处理 s 字符串的时候使用了 gets 函数，很明显这是一个溢出点，需要溢出然后运行 system 函数。那么 name 变量的输入就是为了给我们提供 /bin/sh 的地址。只要将 name 输入为 /bin/sh，把 name 的地址作为 system 函数的参数就可以构造一个缓冲区溢出的栈结构了。

