# forgot

题目来源：https://adworld.xctf.org.cn/task/

这道题虽然在做的过程中看了一下别人的 writeup，但是方法是我想出来的并且没有用到别人的方法，只是确认了一下函数的用法。先看一下代码，这个代码解析出来着实有点复杂：

```C
int __cdecl main()
{
  size_t v0; // ebx
  char v2[32]; // [esp+10h] [ebp-74h]
  int (*v3)(); // [esp+30h] [ebp-54h]
  int (*v4)(); // [esp+34h] [ebp-50h]
  int (*v5)(); // [esp+38h] [ebp-4Ch]
  int (*v6)(); // [esp+3Ch] [ebp-48h]
  int (*v7)(); // [esp+40h] [ebp-44h]
  int (*v8)(); // [esp+44h] [ebp-40h]
  int (*v9)(); // [esp+48h] [ebp-3Ch]
  int (*v10)(); // [esp+4Ch] [ebp-38h]
  int (*v11)(); // [esp+50h] [ebp-34h]
  int (*v12)(); // [esp+54h] [ebp-30h]
  char s; // [esp+58h] [ebp-2Ch]
  int v14; // [esp+78h] [ebp-Ch]
  size_t i; // [esp+7Ch] [ebp-8h]

  v14 = 1;
  v3 = sub_8048604;
  v4 = sub_8048618;
  v5 = sub_804862C;
  v6 = sub_8048640;
  v7 = sub_8048654;
  v8 = sub_8048668;
  v9 = sub_804867C;
  v10 = sub_8048690;
  v11 = sub_80486A4;
  v12 = sub_80486B8;
  puts("What is your name?");
  printf("> ");
  fflush(stdout);
  fgets(&s, 32, stdin);
  sub_80485DD((int)&s);
  fflush(stdout);
  printf("I should give you a pointer perhaps. Here: %x\n\n", sub_8048654);
  fflush(stdout);
  puts("Enter the string to be validate");
  printf("> ");
  fflush(stdout);
  __isoc99_scanf("%s", v2);
  for ( i = 0; ; ++i )
  {
    v0 = i;
    if ( v0 >= strlen(v2) )
      break;
    switch ( v14 )
    {
      case 1:
        if ( sub_8048702(v2[i]) )
          v14 = 2;
        break;
      case 2:
        if ( v2[i] == 64 )
          v14 = 3;
        break;
      case 3:
        if ( sub_804874C(v2[i]) )
          v14 = 4;
        break;
      case 4:
        if ( v2[i] == 46 )
          v14 = 5;
        break;
      case 5:
        if ( sub_8048784(v2[i]) )
          v14 = 6;
        break;
      case 6:
        if ( sub_8048784(v2[i]) )
          v14 = 7;
        break;
      case 7:
        if ( sub_8048784(v2[i]) )
          v14 = 8;
        break;
      case 8:
        if ( sub_8048784(v2[i]) )
          v14 = 9;
        break;
      case 9:
        v14 = 10;
        break;
      default:
        continue;
    }
  }
  (*(&v3 + --v14))();
  return fflush(stdout);
}
```

这个代码的中心意思是检测邮箱地址字符串的合法性。其中每一个 case 就是它所谓的定义的某个状态，整体是用了有限自动机写的。首先查看一下这个文件的安全防护：

```
	Arch:     i386-32-little                                                                 RELRO:    Partial RELRO                                                                   Stack:    No canary found                                                                 NX:       NX enabled                                                                     PIE:      No PIE (0x8048000) 
```

可以看出来可能这里面存在栈溢出。但是我当时觉得没必要构造出一个溢出漏洞，因为里面存在着一个有意思的语句：

```
(*(&v3 + --v14))();
```

v3 通过观察能知道里面存放的是一个函数指针，v14 是偏移量，根据代码逻辑它受到输入的地址字符串的限制，满足某种情况能够得到某个偏移量，然后执行对应的函数。

假设我们现在随便输入一串字符串，比如 aaaaa，那么它触发的是 v4 指向的那个函数。那有个想法就萌生了，如果我可以输入普通的字符串，然后覆盖掉其中某个函数的地址，指向打印 flag 的函数(在这个程序中给出了，地址是 0x80486CC，其实这个可以通过计算，但是 ida 里面已经给出了地址)。

查看 v2 字符串，发现它在栈中的位置靠下，并且按照输入顺序和变量定义的顺序来看正好能够覆盖到，所以只需要构造：

```
'a' * 0x24 + chr(0xCC) + chr(0x86) + chr(0x04) + chr(0x08)
```

> 其实 p32 本身也能够构造字符串，并且适用于 scanf，gets，read

网上有很多解法说灵活控制 v14 的值，意思是要盖到 v14，我觉得这道题里面并没有这个必要。