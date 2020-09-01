# string

题目来源：https://adworld.xctf.org.cn/task

这道题还是使用格式化字符串漏洞，可惜的是这道题就差临门一脚我就做出来了，但是卡在了一个输入的处理上，很是可惜，不过到目前对这个输入的处理还是保留了一点疑问，不过这道题的思路是很明确的：

```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  _DWORD *v3; // rax
  __int64 v4; // ST18_8

  setbuf(stdout, 0LL);
  alarm(0x3Cu);
  sub_400996();
  v3 = malloc(8uLL);
  v4 = (__int64)v3;
  *v3 = 68;
  v3[1] = 85;
  puts("we are wizard, we will give you hand, you can not defeat dragon by yourself ...");
  puts("we will tell you two secret ...");
  printf("secret[0] is %x\n", v4, a2);
  printf("secret[1] is %x\n", v4 + 4);
  puts("do not tell anyone ");
  sub_400D72(v4);
  puts("The End.....Really?");
  return 0LL;
}
```

main 函数说会告诉我们一个秘密，这个秘密就是 v4 变量，也就是 v3 变量的地址，然后程序在最开始初始化了 v3 变量的两个初值。这个在后面肯定是有用的。随后看函数 sub_400D72:

```C
unsigned __int64 __fastcall sub_400D72(__int64 a1)
{
  char s; // [rsp+10h] [rbp-20h]
  unsigned __int64 v3; // [rsp+28h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  puts("What should your character's name be:");
  _isoc99_scanf("%s", &s);
  if ( strlen(&s) <= 0xC )
  {
    puts("Creating a new player.");
    sub_400A7D();
    sub_400BB9();
    sub_400CA6((_DWORD *)a1);
  }
  else
  {
    puts("Hei! What's up!");
  }
  return __readfsqword(0x28u) ^ v3;
}
```

这里就略去一些和逻辑关系不大的函数，只要读懂控制流，就能写出脚本应付，这里直接看到函数 sub_400BB9 和函数 sub_400CA6:

```c
unsigned __int64 sub_400BB9()
{
  int v1; // [rsp+4h] [rbp-7Ch]
  __int64 v2; // [rsp+8h] [rbp-78h]
  char format; // [rsp+10h] [rbp-70h]
  unsigned __int64 v4; // [rsp+78h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  v2 = 0LL;
  puts("You travel a short distance east.That's odd, anyone disappear suddenly");
  puts(", what happend?! You just travel , and find another hole");
  puts("You recall, a big black hole will suckk you into it! Know what should you do?");
  puts("go into there(1), or leave(0)?:");
  _isoc99_scanf("%d", &v1);
  if ( v1 == 1 )
  {
    puts("A voice heard in your mind");
    puts("'Give me an address'");
    _isoc99_scanf("%ld", &v2);
    puts("And, you wish is:");
    _isoc99_scanf("%s", &format);
    puts("Your wish is");
    printf(&format, &format);
    puts("I hear it, I hear it....");
  }
  return __readfsqword(0x28u) ^ v4;
}

unsigned __int64 __fastcall sub_400CA6(_DWORD *a1)
{
  void *v1; // rsi
  unsigned __int64 v3; // [rsp+18h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  puts("Ahu!!!!!!!!!!!!!!!!A Dragon has appeared!!");
  puts("Dragon say: HaHa! you were supposed to have a normal");
  puts("RPG game, but I have changed it! you have no weapon and ");
  puts("skill! you could not defeat me !");
  puts("That's sound terrible! you meet final boss!but you level is ONE!");
  if ( *a1 == a1[1] )
  {
    puts("Wizard: I will help you! USE YOU SPELL");
    v1 = mmap(0LL, 0x1000uLL, 7, 33, -1, 0LL);
    read(0, v1, 0x100uLL);
    ((void (__fastcall *)(_QWORD, void *))v1)(0LL, v1);
  }
  return __readfsqword(0x28u) ^ v3;
}
```

除了这些函数之外没有其他有用的函数了，开来这道题的目标是想让我们拿到控制台，从而获取 flag。首先先看 sub_400CA6，这个函数会比较 a1[0] 和 a1[1] 的值，然后使用 mmap 开辟一个空间，最后由用户写入一些东西并执行它，很显然这是为了让我们增加可执行的 shellcode。那么 a1 这个变量是什么，如果查看数据依赖关系可以发现，这个 a1 就是一开始的 v3。但是如果按照正常的逻辑 v3[0] 和 v3[1] 是不一样的。

回到函数 sub_400BB9，在这里给了一点提示，这里有两个非常可疑的输入，一个要你输入一个地址，一个要你输入一个格式，而注意下面的语句：

```
printf(&format, &format)
```

这个相当于人为给了一个格式化字符串漏洞，那么根据该漏洞的一般处理方式，找到目标地址在栈中的位置，然后写入值就可以了。像这里一定是想把 v3[0] 或者 v3[1] 的地址放到栈中(在 main 函数里知道了)，然后知道了它的位置后，写入适当的值就可以了。

---

这里我卡壳的地方在于，`scanf("%s", &format)` 输入格式的时候，是不能直接输入地址的，否则无法得到正确结果，应该是和 %s 的格式有关，这里我卡了很久以为是错了。所以这里必须借助上面的 v2 变量输入 v3 的地址。