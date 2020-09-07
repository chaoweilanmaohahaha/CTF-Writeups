# dice_game

题目来源：https://adworld.xctf.org.cn/task

这道题在文件夹里还特地给了 libc，但是如果仔细读了代码的逻辑发现其实并不需要 libc 也可以拿到 flag 了，题目的内容是一个比较老套的漏洞，先看一下代码，首先是 main 函数：

```C
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  char buf[55]; // [rsp+0h] [rbp-50h]
  char v5; // [rsp+37h] [rbp-19h]
  ssize_t v6; // [rsp+38h] [rbp-18h]
  unsigned int seed[2]; // [rsp+40h] [rbp-10h]
  unsigned int v8; // [rsp+4Ch] [rbp-4h]

  memset(buf, 0, 0x30uLL);
  *(_QWORD *)seed = time(0LL);
  printf("Welcome, let me know your name: ", a2);
  fflush(stdout);
  v6 = read(0, buf, 0x50uLL);
  if ( v6 <= 49 )
    buf[v6 - 1] = 0;
  printf("Hi, %s. Let's play a game.\n", buf);
  fflush(stdout);
  srand(seed[0]);
  v8 = 1;
  v5 = 0;
  while ( 1 )
  {
    printf("Game %d/50\n", v8);
    v5 = sub_A20();
    fflush(stdout);
    if ( v5 != 1 )
      break;
    if ( v8 == 50 )
    {
      sub_B28((__int64)buf);
      break;
    }
    ++v8;
  }
  puts("Bye bye!");
  return 0LL;
}
```

这个游戏的内容又是需要我们答对 50 个随机数，才能拿到 flag，首先是初始化随机数的种子，然后输入你的用户名，通过 while 循环开始游戏，游戏中先调用 sub_A20 函数：

```C
signed __int64 sub_A20()
{
  signed __int64 result; // rax
  __int16 v1; // [rsp+Ch] [rbp-4h]
  __int16 v2; // [rsp+Eh] [rbp-2h]

  printf("Give me the point(1~6): ");
  fflush(stdout);
  _isoc99_scanf("%hd", &v1);
  if ( v1 > 0 && v1 <= 6 )
  {
    v2 = rand() % 6 + 1;
    if ( v1 <= 0 || v1 > 6 || v2 <= 0 || v2 > 6 )
      _assert_fail("(point>=1 && point<=6) && (sPoint>=1 && sPoint<=6)", "dice_game.c", 0x18u, "dice_game");
    if ( v1 == v2 )
    {
      puts("You win.");
      result = 1LL;
    }
    else
    {
      puts("You lost.");
      result = 0LL;
    }
  }
  else
  {
    puts("Invalid value!");
    result = 0LL;
  }
  return result;
}
```

可以看到这里的随机数被限制在了 1~6 的范围内，输入的数据只有和随机数是相同的，才会返回 1。回到刚才 main 的逻辑，当获胜了 50 次之后，就可以触发 sub_B28：

```C
int __fastcall sub_B28(__int64 a1)
{
  char s; // [rsp+10h] [rbp-70h]
  FILE *stream; // [rsp+78h] [rbp-8h]

  printf("Congrats %s\n", a1);
  stream = fopen("flag", "r");
  fgets(&s, 100, stream);
  puts(&s);
  return fflush(stdout);
}
```

这个函数就是打开 flag 然后获得答案。

这道题的问题点锁定在种子的获取和名字的输入上，首先可以看到的是虽然在输入完名字之后有一个判断，保证名字确实只能是 50 的长度，但是 read 函数允许读入的长度是 0x50。并且在这里可以看到种子是使用 seed[0]，它的初始化在输入名字之前，并且参数在栈中的位置是高于 name 变量的。所以这里就可以在输入 name 变量的时候覆盖掉 seed[0] 的值，将它替换为任意一个已知数，比如 1。这样可以自己另写一个程序来生成一个“随机数”序列。

比如使用 1 为种子后生成的序列如下：

> 25426251423232651155634433322261116425254446323361