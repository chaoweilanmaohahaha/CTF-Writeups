# level0

题目来源：https://adworld.xctf.org.cn/task/answer?type=pwn&number=2&grade=0&id=5053&page=1

这道题题面非常简单，就是溢出拿到 shell，这一道题目看完了就知道它是一个典型的 `return to text` 的题目，也就是跳转到该程序内已有的一个代码段。因为这道题只给了二进制文件，为了方便起见，先用 IDA 查看一下，借助 IDA 中的插件，可以解析出差不多已经是 C 语言的代码：

```c
int callsystem()
{
  return system("/bin/sh");
}

ssize_t vulnerable_function()
{
  char buf; // [rsp+0h] [rbp-80h]

  return read(0, &buf, 0x200uLL);
}

int __cdecl main(int argc, const char **argv, const char **envp)
{
  write(1, "Hello, World\n", 0xDuLL);
  return vulnerable_function();
}
```

这是已经经过插件处理过的代码了，程序的流程非常简单，打印一个 hello world，随后触发一个带有缓冲区溢出漏洞的函数，接着借助这个函数触发 callsystem 函数，为了做的细节一点，先用 checksec 看一下安全的限制。

```
Arch:     amd64-64-little                                          
RELRO:    No RELRO                                                    
Stack:    No canary found                                             
NX:       NX enabled                                                    
PIE:      No PIE (0x400000)
```

可以看到没有 canary 防护，可以触发溢出。查看汇编代码发现需要填充 0x88 个其他字符后再将 callsystem 的地址放在后面就可以了。