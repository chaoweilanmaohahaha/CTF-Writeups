# blukat

题目来源：https://pwnable.kr/

这道题的题面非常有意思，意思是说如果你觉得这道题很难的话，就说明你是一个很有技巧的 pwner。好的，意思是不能把这道题想的太复杂。

文件夹里一共有三个文件，其中可以看到比较关键的是这个 password 文件，如果习惯性用 cat 查看会发现显示没有权限。但是有意思的是这道题就这样欺骗了过去。如果使用 ls -al 命令查看文件信息，会发现 password 属于 root 用户，属于 blukat_pwn 用户组，我们如果使用 id 命令查看 blukat 用户的用户组时，你可以惊奇发现，blukat 也属于 blukat_pwn 用户组，那么也就是说 blukat 用户对 password 文件是有读权限的。

所以这道题中这个 password 文件的内容就是：

> cat: password: Permission denied

那么有了这个先验知识，再来看程序的逻辑：

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
char flag[100];
char password[100];
char* key = "3\rG[S/%\x1c\x1d#0?\rIS\x0f\x1c\x1d\x18;,4\x1b\x00\x1bp;5\x0b\x1b\x08\x45+";
void calc_flag(char* s){
	int i;
	for(i=0; i<strlen(s); i++){
		flag[i] = s[i] ^ key[i];
	}
	printf("%s\n", flag);
}
int main(){
	FILE* fp = fopen("/home/blukat/password", "r");
	fgets(password, 100, fp);
	char buf[100];
	printf("guess the password!\n");
	fgets(buf, 128, stdin);
	if(!strcmp(password, buf)){
		printf("congrats! here is your flag: ");
		calc_flag(password);
	}
	else{
		printf("wrong guess!\n");
		exit(0);
	}
	return 0;
}
```

仔细看 main 函数的逻辑，很简单，就是首先打开 password 文件然后读取里面的内容，随后和用户输入的内容进行对比，如果相同就计算 flag，而 flag 就是按照 password 中的内容和 key 变量逐个元素异或得到的。那么我们既然已经知道 password 中的内容了，这道题就已经结束了。

---

这道题我虽然也做出来了，但是并没有想到文件权限这一个方面，而是正常使用 gdb 来查看程序，这样当然也能做出来，需要在调试中去查看 password 数组中的内容。

