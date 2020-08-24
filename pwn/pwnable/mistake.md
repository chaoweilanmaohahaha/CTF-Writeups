# mistake

题目来源：https://pwnable.kr/

这道题的出发点考察的是操作符优先级会带来的软件漏洞，这个问题是一个很细小的问题，所以一般都很难发现，如果不是对操作符优先级非常熟悉的人基本认为该程序就是对的。

先来看一下代码分析一下执行流程：

```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}
```

分析 main 函数的流程不难得知，程序的目的是要先打开 password 文件，这个文件没有一定的权限是无法打开的，随后从 password 中读入密钥。用户输入密钥后先需要进行一个异或操作，随后与真正的密钥对比，如果相同则通过验证。

确实乍看起来没有任何的问题，从流程到每个函数的调用，确实这道题我看了很多遍也没有找到问题所在。但是运行这个程序的时候你会发现和程序书写的逻辑会不太一样。理论上 sleep 函数只会让程序休眠最多 19s。但是你会发现就算等待了 19s 之后程序依旧会停在那里，并且你可以对他输入。

其实这个程序的问题出在 main 函数第三行的判断语句上。这个里面有一个 = 和 < 两个操作符，**而且 < 的优先级是大于 = 号的**。换句话说对于这整个语句，会先执行后面的语句，然后结果赋值为 fd。分析一下，因为在当前目录是存在 password 这个文件的，所以 open 理论上一定能成功并且返回一个正整数作为文件描述符，这样后面半句语句一定为假，即 0。有意思的是这个 0 恰好赋值给 fd 之后充当 stdin，而程序下面的读入也变成了从标准输入中读取 password。

有了以下的分析可以知道，这个 password 文件就变得形同虚设，其实真正的密码是用户能够随便给出的。根据这个发现以及下面的异或准则，很好构造密码为 000000000，用户输入为 1111111111。

---

最后附上 c 语言的运算符优先级吧：

[https://baike.baidu.com/item/%E8%BF%90%E7%AE%97%E7%AC%A6%E4%BC%98%E5%85%88%E7%BA%A7/4752611?fr=aladdin](https://baike.baidu.com/item/运算符优先级/4752611?fr=aladdin)