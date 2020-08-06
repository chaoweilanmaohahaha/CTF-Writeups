# collision

题目来源：https://pwnable.kr

这道题看起来有关 hash 碰撞，实际和 hash 没什么关系，当使用 ssh 登录到服务器时，会发现文件夹的权限和第一题是一样的，不许我们创建脚本，并且只能查看 col.c 与运行 col。有了第一题的经验很容易想到这道题就是要求我们能够正确运行 col。先上 col 代码：

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}                                                                                         

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}                                                                                      
	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}                              
```

这段代码没有复杂的逻辑，重点就是放在 check_password 这个函数中。很明显该程序的输入必须是一个 20 字节的字符串，在 check_password 中会将这个字符串强制转换成一个整型数组。这样每四个字节就需要拼接成为一个整数，随后累加得到一个值与 0x21DD09EC 比较。

其实思想很简单，就是要找五个数进行累加，其实这个很容易想到，但是问题在于如何输入到这个程序中，这个我搜索了一下资料，原来在 linux 的控制台命令中也有一个打印命令：

> printf 命令：
>
> printf [format] [var1] [var2]
>
> 这个命令和 c 语言很类似，字符串也是要加上双引号。当然还有一个回显的命令 echo，不过注意 echo 本身时没有格式变量替换功能的，只是做到回显，而 printf 是有格式替换功能的，e.g：
>
> print "%d" 1
>
> 1

我们可以通过随便给出一些数字，只要累加得到的数是满足条件的，这里选择了四个 0x01010101 和 0x1DD905E8。为什么选择十六进制也是因为方便输入，并且本身也支持 16 进制的表示。

最后一个问题是如何将这个命令的输出传给可执行程序。在 linux 中也有类似的符号功能：

> 在 linux 中有变量替换和命令替换的符号。bash 中使用 $() 和 `` 都可以进行命令替换，所谓命令替换，就是指将符号内的命令执行的结果替换出来，重新组成新的命令，比如：
>
> echo now is $(date)
>
> now is 20200803
>
> 换句话说就是先执行了符号内的命令，随后用该结果替换这个位置的命令，随后再执行外部的命令。
>
> 那么再 bash 中还有一个 ${} 可以用作变量替换，这个就很好理解了。

有了上面的工具，就可以构造执行的一个指令，将数据输入到这个程序中，最后需要注意的一点是目前一般系统都是小端的，所以取 int 的时候存储和看到的是不同的，这在构造是需要小心。

执行下面命令拿到 flag：

```
./col `printf "\xe8\x05\xd9\x1d\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01"`
```

