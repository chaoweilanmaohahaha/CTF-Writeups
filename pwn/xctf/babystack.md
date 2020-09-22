# babystack

题目来源：https://adworld.xctf.org.cn/task/

这道题虽然有点坑，但是从中可以学到很多知识，很全面的一道题，其实做这道题主要培养一个思路，如果思路清晰了就不难做出来了，这道题在文件夹中给了 libc，说明后面需要借助 libc 中的函数来构造，先看一下代码：

```C
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  int v3; // eax
  char s; // [rsp+10h] [rbp-90h]
  unsigned __int64 v6; // [rsp+98h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  setvbuf(stdin, 0LL, 2, 0LL);
  setvbuf(stdout, 0LL, 2, 0LL);
  setvbuf(stderr, 0LL, 2, 0LL);
  memset(&s, 0, 0x80uLL);
  while ( 1 )
  {
    sub_4008B9();
    v3 = sub_400841();
    switch ( v3 )
    {
      case 2:
        puts(&s);
        break;
      case 3:
        return 0LL;
      case 1:
        read(0, &s, 0x100uLL);
        break;
      default:
        sub_400826("invalid choice");
        break;
    }
    sub_400826((const char *)&unk_400AE7);
  }
}
```

sub_4008B9 是输出固定的文本，包括 sub_400841 也是一样，需要我们输入一个选项。主要就是这么一个 switch，一个代表读入数据，一个代表输出数据。很明显观察到，在 read 上有一个缓冲区溢出问题的，说明漏洞利用就是从这里入手，看一下安全防护：

```
Arch:     amd64-64-little                                           
RELRO:    Full RELRO                                                    
Stack:    Canary found                                                    
NX:       NX enabled                                                   
PIE:      No PIE (0x400000)
```

很恐怖，几乎全开，RELRO 和 GOT 表的写有关，Full 说明 GOT 表不可写，不过 PIE 没开说明对 ROP 限制并没有那么大。这里第一个难点就是存在 canary，这样没法直接构造缓冲区溢出。这里就学到一招，如何泄露 canary。

这里仅仅用 puts 就泄露了 canary。canary 的大致位置在 rbp 下，看 ida 翻译出来的数据也能发现就在 rbp -0x8 处，或者可以用 ida 的动态调试，这是因为我使用本地 gdb 不好调试，学了这么一招，在这里挂一下链接：https://blog.csdn.net/abc_670/article/details/80066817。

canary 有一个有意思的特点，就是它的低位是 0x00，这么设计的理由是在普通情况下，puts 和 printf 都无法打印出 canary 后面的内容，因为到它的低位就碰到 0x00 这个终止符了，所以在一定程度上缓解了泄露，但是假设溢出的足够多，这个防护还是没什么用。比如在这里把这个 0x00 一起覆盖了，就能拿到对应的 canary 了。所以第一次需要构造字符串 'a' * 0x88。

当获得了 canary 后就能正常进行溢出。当然在这个文件中没有有关控制台的函数，所以需要自己去构造，给了我们 libc，就是说明要从 libc 中找 system 和 bin/sh，这些都好弄，直接用 pwntool 就行(但是这里有个坑，后面再说)。

最重要的一步就是，我们拿到的只是函数在 libc 文件中的偏移，但是 libc 到底加载到了内存的哪里是不知道的，所以这还需要想办法暴露 libc 的基址。这里学习一下一般步骤，其实即使抓住 libc 中的某个函数比如 write，然后在 got 表中存放着 write 的真实内存地址，减去 write 在 libc 文件中的偏移就是 libc 的基址，但是如何获取 write 的真实地址，这个需要通过溢出和 ROP 的配合。

32 位的泄露方法和 64 位的泄露方法是有些许不同的，这是由于函数的调用规定是不同的。就比如这里，如果要借助 puts 去泄露 write 的 got，则 puts 的参数是由  rdi 传入的，而不是通过栈，这是 gcc 下的函数调用规定。所以还需要从本文件中获得 pop rdi 的 gadget 的地址：

```
'a' * 0x88 + p64(canary) + 'a' * 0x08 + p64(pop_rdi) + p64(write_got) + p64(puts_plt) + p64(main)
```

只有这样拿到了 write 的真实地址，就能计算 system 和 /bin/sh 的地址，同样构造最终的字符串的时候注意调用规范：

``` 
'a' * 0x88 + p64(canary) + 'a'*0x08 + p64(pop_rdi) + p64(sh_addr) + p64(system_addr)
```

---

这里说一下坑点：我就这样搞了半天，怎么一直不对，一开始我用的 pwntools 去解析了 libc 中的 system 等函数的偏移，但是后来我得知文件夹中给出的 libc 和题目服务器上的 libc 的版本并不完全对应，所以用 pwntools 直接解析本地的 libc 文件是不能正确得到答案的。

理论上需要去查找正确版本的 libc，方法是使用函数地址的最后 12 位去识别 libc 版本，这是由于在加载 libc 的时候一定是页对齐的，那么每个指令的地址的最后 12 位一定是不变的。

但是其实在 github 上已经有了项目 LibcSearcher，以后碰到对 libc 处理的题目可以直接用这个库来操作。