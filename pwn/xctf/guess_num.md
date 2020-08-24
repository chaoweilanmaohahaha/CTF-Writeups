# guess_num

题目来源：https://adworld.xctf.org.cn/

这道题考察的是对 rand 和 srand 函数的认识，通过 IDA 将源程序翻译成 C 伪代码分析一下控制流：

```C
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  int v4; // [rsp+4h] [rbp-3Ch]
  int i; // [rsp+8h] [rbp-38h]
  int v6; // [rsp+Ch] [rbp-34h]
  char v7; // [rsp+10h] [rbp-30h]
  unsigned int seed[2]; // [rsp+30h] [rbp-10h]
  unsigned __int64 v9; // [rsp+38h] [rbp-8h]

  v9 = __readfsqword(0x28u);
  setbuf(stdin, 0LL);
  setbuf(stdout, 0LL);
  setbuf(stderr, 0LL);
  v4 = 0;
  v6 = 0;
  *(_QWORD *)seed = sub_BB0();
  puts("-------------------------------");
  puts("Welcome to a guess number game!");
  puts("-------------------------------");
  puts("Please let me know your name!");
  printf("Your name:", 0LL);
  gets(&v7);
  srand(seed[0]);
  for ( i = 0; i <= 9; ++i )
  {
    v6 = rand() % 6 + 1;
    printf("-------------Turn:%d-------------\n", (unsigned int)(i + 1));
    printf("Please input your guess number:");
    __isoc99_scanf("%d", &v4);
    puts("---------------------------------");
    if ( v4 != v6 )
    {
      puts("GG!");
      exit(1);
    }
    puts("Success!");
  }
  sub_C3E();
  return 0LL;
}
```

上面的函数是 main 函数，执行的内容主要是：首先使用 sub_BB 来获取种子：

```C
__int64 sub_BB0()
{
  int fd; // [rsp+Ch] [rbp-14h]
  __int64 buf; // [rsp+10h] [rbp-10h]
  unsigned __int64 v3; // [rsp+18h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  fd = open("/dev/urandom", 0);
  if ( fd < 0 || (signed int)read(fd, &buf, 8uLL) < 0 )
    exit(1);
  if ( fd > 0 )
    close(fd);
  return buf;
}
```

sub_BB 获取种子的方法是通过从 /dev/urandom 文件中读取 8 个字节，由之前的了解得知，这个文件是 linux 里专门用来生成随机数的文件。理论上用该文件生成的种子是绝对随机的。

回到主函数中，接下去使用获得的 seed，一共需要生成 10 轮随机数，并且最后通过取模将范围限制在 1-6 之间，用户需要猜 10 次，如果全答对了就能够通过函数 sub_C3E 拿到 flag。

这道题首先能够看到在中间使用了不安全的函数 gets，这就是一个可能的漏洞点，那么可能可以造成溢出，但是使用 checksec 后发现该程序启用了 NX 和 canary 防护，不可能直接进行缓冲区溢出攻击。但是在源代码中，可以发现，seed 和 name 变量都是一个局部变量，且有意思的是程序先生成了 seed 后输入 name。从中查看 seed 变量和 name 变量的分布，可以发现 seed 变量又恰巧在 name 变量的后面。

因为 srand 函数使用 seed[0] 作为输入，并且必须知道 rand 产生的是伪随机数，只要种子是相同的，产生的随机数每次也都会是相同的。所以只需要将种子换成是已知的就可以知道 10 次随机数的值了。而这个就可以通过 gets 函数的输入控制，只需要输入 0x20 个无关变量，然后再输入一个已知整数就可以了。

我们能自己额外写一个程序来获取自己设置种子下 10 次随机数的生成情况。