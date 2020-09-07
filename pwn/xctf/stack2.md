# Stack2

题目来源：https://adworld.xctf.org.cn/task/

这道题虽然做起来觉得很坑，但是确实教会了我很多，漏洞的发现其实不难，但是真正能够处理这个漏洞确实需要足够地仔细，先来看一下代码：

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v3; // eax
  unsigned int v5; // [esp+18h] [ebp-90h]
  unsigned int v6; // [esp+1Ch] [ebp-8Ch]
  int v7; // [esp+20h] [ebp-88h]
  unsigned int j; // [esp+24h] [ebp-84h]
  int v9; // [esp+28h] [ebp-80h]
  unsigned int i; // [esp+2Ch] [ebp-7Ch]
  unsigned int k; // [esp+30h] [ebp-78h]
  unsigned int l; // [esp+34h] [ebp-74h]
  char v13[100]; // [esp+38h] [ebp-70h]
  unsigned int v14; // [esp+9Ch] [ebp-Ch]

  v14 = __readgsdword(0x14u);
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
  v9 = 0;
  puts("***********************************************************");
  puts("*                      An easy calc                       *");
  puts("*Give me your numbers and I will return to you an average *");
  puts("*(0 <= x < 256)                                           *");
  puts("***********************************************************");
  puts("How many numbers you have:");
  __isoc99_scanf("%d", &v5);
  puts("Give me your numbers");
  for ( i = 0; i < v5 && (signed int)i <= 99; ++i )
  {
    __isoc99_scanf("%d", &v7);
    v13[i] = v7;
  }
  for ( j = v5; ; printf("average is %.2lf\n", (double)((long double)v9 / (double)j)) )
  {
    while ( 1 )
    {
      while ( 1 )
      {
        while ( 1 )
        {
          puts("1. show numbers\n2. add number\n3. change number\n4. get average\n5. exit");
          __isoc99_scanf("%d", &v6);
          if ( v6 != 2 )
            break;
          puts("Give me your number");
          __isoc99_scanf("%d", &v7);
          if ( j <= 0x63 )
          {
            v3 = j++;
            v13[v3] = v7;
          }
        }
        if ( v6 > 2 )
          break;
        if ( v6 != 1 )
          return 0;
        puts("id\t\tnumber");
        for ( k = 0; k < j; ++k )
          printf("%d\t\t%d\n", k, v13[k]);
      }
      if ( v6 != 3 )
        break;
      puts("which number to change:");
      __isoc99_scanf("%d", &v5);
      puts("new number:");
      __isoc99_scanf("%d", &v7);
      v13[v5] = v7;
    }
    if ( v6 != 4 )
      break;
    v9 = 0;
    for ( l = 0; l < j; ++l )
      v9 += v13[l];
  }
  return 0;
}
```

代码有点长且冗余，但是仔细看这个代码一共就是四件事，首先需要用户输入一些数字，随后可以有四个选项，展示目前的数据、在数组末尾添加一个数、修改某一个指定位置的数、求当前数据的平均数。

先看一下这道题的安全防护：

```
	Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

有 canary 在就说明不能直接利用缓冲区溢出漏洞了。但是在这道题里确实提供了 hackhere 的函数，这个函数就是用来打开 bash 的。那么在没有其他方法执行它的情况下一定是覆盖掉返回地址来触发这个函数，那怎么处理呢？

可以观察到在这个 main 函数中的 v13 变量，它是一个没有边界限制的变量，无论是添加数据还是修改数据，都不会检查数据是否超过了边界。**那么因为 canary 添加数据不现实了，所以就考虑修改数据，为了绕过 canary，直接找到返回地址的偏移，随后修改这个数据单元**。

这道题第一个坑就是，要注意 v13 变量是一个 char 类型的数据，所以每一个元素是 1 个字节的，要计算好输入的位置；

第二个坑在于，从一般的推理来说，因为数组的输入位置是在 ebp - 0x70 处，则返回地址应该在 ebp + 0x4 处，但是没想到这道题不是这样。我使用了 gdb 对其进行了查看，通过计算返回地址的位置，以及 ebp 的位置可以发现偏移的值为 0x84，如果查看汇编代码会发现在 push ebp 之前还处理了对齐。

第三个坑在于，当你所有的数据都准备好运行之后，返回的结果是 /bin/bash: not found，这是很少会出现的，原因竟然是目标主机上没有 /bin/bash 程序。这个我求助了一下网上的说法，原来只需要在代码中执行 system('sh')，也能拿到控制台，因为在 linux 中就有这个 sh 命令。

> sh命令是shell命令语言解释器，执行命令从标准输入读取或从一个文件中读取。通过用户输入命令，和内核进行沟通！

而 sh 字符串就出现在 /bin/bash 字符串当中，那就只需要计算 sh 字符串的起始地址就可以了。