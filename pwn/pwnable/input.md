# input

题目来源：https://pwnable.kr/

这道题的意思非常明确，就是考察 5 种输入的技巧，但是里面确实涉及了很多 linux 变成的技巧，有很多我暂时还不知道，还需要后续学习。我选择的是用 C 语言来写，不过也是参考了网上写的，有些地方确实还不清楚为什么这么设计。

不过我倒是看网上还有 python 脚本的写法，也可以拿来分析分析。

首先先上任务代码，然后一部分一部分分析：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
    if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");	
	return 0;
}

```

代码也很清晰，一共把总的任务分为了五个部分。首先第一个部分是控制台参数的输入，由于需要输入的字符串很多，直接手输肯定不行，所以执行这个程序也要借助程序。比如在 C 种可以使用 execve 函数来运行这个程序，并且明确参数：

```
int execve(const char * filename,char * const argv[ ],char * const envp[ ]);
```

C 语言中的 main 函数一共有三个参数，argc 是参数个数，argv 就是参数列表，envp 是环境变量列表。其中 argv 参数列表中，第一个元素就是程序本身的名字，然后才是输入到程序中的参数。那么像这里要求 argc 是 100，就要求有一个程序名和 99 个参数，别忘了 argv 最后需要添上一共 NULL 表示列表的结束。其中要求第 65 个参数和第 66 个参数是指定参数，这个很好修改，所以第一个任务并不难。

先跳过第二个任务看一下第三个，这是要求当前运行的进程中有指定的环境变量。这就要用到 execve 中的第三个参数，同样也可以将环境变量传递进去，注意环境变量是一个 "key=value" 的字符串，并且数组的最后一项也是一个 NULL 作为结束标志。使用 envp 传入的环境变量参数就是子进程的环境变量，不是附加。

第四个任务也很简单，是文件读写。只需要创建一个名字叫做 0x0a 文件，然后往里面写入相应四个字节。

第五个任务是 linux socket 编程，很显然题目中的是一个服务器程序，题目中程序监听从任意地址发来的，监听的端口由 argv['C'] 参数指定，当连接上后，只需要向该 socket 传送相应的内容就可以了。

最麻烦的我觉得是第二个任务，这个任务简单的说就是需要读取 stdin 和 stderr 中的字符串，如果能够匹配上目标字符串才可以。难弄的其实就是这个 stderr，因为有一点是这个 stderr 只要往其中写数据会立马输出不会有任何缓存。最后参考了答案中的思路，使用了管道和重定向。

目的很简单，如果能够将这个进程中的 stderr 文件描述符重定向到一个能够控制的文件中，并将数据直接存放到那个文件，则在进程中就能得到这个字符串。这一步可以使用 dup2 这个系统调用完成。那么就需要解决如何输入，完成这一步需要借助管道 pipe：

```
int pip(int pipfd[2])
```

函数调用成功返回两个文件描述符，0 号代表读 1 代表写，而这个管道读写区域就是内核缓冲区。假设此时 fork 出子进程时，子进程会继承父进程的文件描述符表，那么同样也有这样的管道，这就完成了父子进程的通信。有了这个的帮助，很容易想到能用子进程写入管道，然后让父进程将 stderr 重定向到管道然后读取数据。

这里必须补充，父子进程是有相同的文件描述符内容的，除非是自己有单独的代码修改，如果使用  exec 运行新的程序的时候，会根据 exec_on_close 标志位来判断是否要关闭。