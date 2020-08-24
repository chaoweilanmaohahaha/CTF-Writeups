# cmd2

题目来源：https://pwnable.kr/

这道题与 cmd1 类似，也是在添加了一定的过滤条件后执行某个命令，代码如下：

```C
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "=")!=0;
	r += strstr(cmd, "PATH")!=0;
	r += strstr(cmd, "export")!=0;
	r += strstr(cmd, "/")!=0;
	r += strstr(cmd, "`")!=0;
	r += strstr(cmd, "flag")!=0;
	return r;
}

extern char** environ;
void delete_env(){
	char** p;
	for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
	delete_env();
	putenv("PATH=/no_command_execution_until_you_become_a_hacker");
	if(filter(argv[1])) return 0;
	printf("%s\n", argv[1]);
	system( argv[1] );
	return 0;
}
```

这一次该程序做得更绝一点，首先它保证将进程运行时可能使用到的系统环境变量清除。

`environ` 变量本身存储着系统的环境变量，如果直接运行这个进程，那么默认使用的环境变量经过实验查看就是这个系统环境变量。而且在 main 函数中使用 putenv 进一步修改了 PATH 的变量值，这样使得无论如何都无法修改该进程运行时 PATH 变量的值了。

同样使用 system 来运行命令，而这个函数会继承父进程的环境变量。

看一下 filter 过滤的函数，这个对我们输入的命令提出了限制，照限制的做法，它不允许我们在下面再对环境变量进行修改，同时也不允许使用 /，这个无疑是禁止使用绝对路径来执行某个程序。理论上之前提到过 linux 可以使用三种方法来执行程序，由于环境变量和路径的封锁，感觉是一种方法都无法实施。

但是我们的目的一定是想要执行 cat 命令来查看 flag 文件，要想执行的命令类似 `/bin/cat flag`。那么焦点问题在于怎么模拟出路径。网上的资料教了一招。

`pwd` 命令是用来打印当前工作目录的，那么假设当前的工作目录就是 / 根目录，则 pwd 的值就是 /。这个用处就在于在 linux shell 下，可以使用 $() 来将执行命令后的结果作为参数，这么一想原命令就转变为了类似 `$(pwd)bin$(pwd)cat flag`。但是很奇怪的是，不是所有的命令都因为环境变量的问题被禁止了，比如 ls，pwd 为什么还是可以执行的呢？

> 这里要提到 linux 中的内部命令和外部命令
>
> linux 系统会将一些轻量级的命令在启动时加载道内存中供 shell 调用，这部分命令是内部命令；而外部命令则是存储在硬盘上只有被调用时才会被加载。也就是说内部命令就是 shell 的一部分，而外部命令则需要 shell 通过执行路径查找、加载来执行。

我们在 linux 上可以通过 `type cmd` 来查看一个命令是内部的还是外部的：

```
cmd2@pwnable:~$ type pwd
pwd is a shell builtin
```

而对于 pwd 这个命令，同样有一个 /bin/pwd 命令，后者就属于外部命令。