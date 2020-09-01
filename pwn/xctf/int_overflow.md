# int_overflow

题目来源：https://adworld.xctf.org.cn/

这道题太细节了，虽然一眼能够看出问题在哪，但是就是因为忽视了其中的一个细节，所以不清楚应该如何构造溢出。

使用 IDA32 来看一下程序伪代码，其中 main 函数没有任何用处，只是一个进入 login 函数的界面，所以直接从 login 函数进去查看：

```c
char *login()
{
  char buf; // [esp+0h] [ebp-228h]
  char s; // [esp+200h] [ebp-28h]

  memset(&s, 0, 0x20u);
  memset(&buf, 0, 0x200u);
  puts("Please input your username:");
  read(0, &s, 0x19u);
  printf("Hello %s\n", &s);
  puts("Please input your passwd:");
  read(0, &buf, 0x199u);
  return check_passwd(&buf);
}
```

虽然说看起来开辟的缓冲区非常大，但是从这个函数中并看不出什么问题，接下去会调用 check_passwd：

```C
char *__cdecl check_passwd(char *s)
{
  char *result; // eax
  char dest; // [esp+4h] [ebp-14h]
  unsigned __int8 v3; // [esp+Fh] [ebp-9h]

  v3 = strlen(s);
  if ( v3 <= 3u || v3 > 8u )
  {
    puts("Invalid Password");
    result = (char *)fflush(stdout);
  }
  else
  {
    puts("Success");
    fflush(stdout);
    result = strcpy(&dest, s);
  }
  return result;
}
```

先不分析 check_passwd 函数，先再从函数列表中查看，发现还有一个未被调用的函数 what_is_this，这个函数会调用 `system('cat flag')` 获取最后的答案。

很明显要我们构造溢出，然后调用 what_is_this 这个函数。分析一下 check_passwd，参数 s 代表的是之前输入的 passwd，如果长度小于等于 3 或者大于 8 则退出，否则就执行 strcpy 函数。很明显溢出点就在这个 strcpy 上。

```
char *strcpy(char* dest, const char *src);
目的是从 src 处开始将字符串复制到 dest 处，直到遇到 NULL。但是这个函数不安全在于不会检测 src 字符串实际长度，因此很可能产生溢出。所以最安全的应该调用 strncpy
```

这里就有一个细节了，所以为什么题目叫做 int_overflow。如果从 ida 的代码中仔细查看，会发现 v3 这个变量是一个 int8 类型，也就是最大数值是 255。而输入密码的缓冲区长度远不止 255，而 strlen 函数的性质，又是要判断字符串是否到达了 NULL 才会停止。所以这里就会产生整型的溢出，比如我们构造密码长度为 260 也能通过安全检测，那么只需要将 what_is_this 的地址藏在这个 passwd 参数中，因为你可以查看这个 dest 变量在栈中的位置。

那么问题是，如果不借助 ida，只有汇编代码应该如何判断 v3 是 int8 类型呢，回过头看汇编代码。并不能从栈中开辟空间来判断，目前而言只能从使用到的寄存器宽度来判断变量的宽度：

```
.text:080486B8                 mov     [ebp+var_9], al
```

