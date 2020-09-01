# when_did_you_born

题目来源：https://adworld.xctf.org.cn/

这道题目其实没什么难度，可惜在做的时候拉跨，犯了不少错误，否则应该是秒做。先看一下代码：

```C
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  __int64 result; // rax
  char v4; // [rsp+0h] [rbp-20h]
  unsigned int v5; // [rsp+8h] [rbp-18h]
  unsigned __int64 v6; // [rsp+18h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  setbuf(stdin, 0LL);
  setbuf(stdout, 0LL);
  setbuf(stderr, 0LL);
  puts("What's Your Birth?");
  __isoc99_scanf("%d", &v5);
  while ( getchar() != 10 )
    ;
  if ( v5 == 1926 )
  {
    puts("You Cannot Born In 1926!");
    result = 0LL;
  }
  else
  {
    puts("What's Your Name?");
    gets(&v4);
    printf("You Are Born In %d\n", v5);
    if ( v5 == 1926 )
    {
      puts("You Shall Have Flag.");
      system("cat flag");
    }
    else
    {
      puts("You Are Naive.");
      puts("You Speed One Second Here.");
    }
    result = 0LL;
  }
  return result;
}
```

执行流程如下：首先输入变量 v5 的值，如果这个值一开始不能是 1926 否则直接失败，然后输入字符串 v4 作为名字，然后再比较一次 v5 的值，如果是 1926 则成功拿到 flag。也就是说这个逻辑是前后矛盾的。

我一开始没有用 checksec 去判断这个程序的保护情况，直接用栈溢出去做了，后来发现这个程序是受到 canary 保护的。那不能用栈溢出，只能分析变量在栈中的分布了，可以发现 v4 变量是在 v5 变量的下面，意思是通过 gets 函数是可以覆盖 v5 变量的，那么就很容易构造参数。

我主要踩的坑是 getchar 函数，这个函数原本以为希望我再输入一个字符，其实这个函数在这里的目的是为了吃掉 scanf 的回车，清空输入的缓冲区。