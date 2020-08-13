# lotto

题目来源：https://pwnable.kr/

这道题的程序是一个类似乐透彩票的程序，先分析一下它的代码，看一下程序的逻辑：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

unsigned char submit[6];

void play(){
	
	int i;
	printf("Submit your 6 lotto bytes : ");
	fflush(stdout);

	int r;
	r = read(0, submit, 6);

	printf("Lotto Start!\n");
	//sleep(1);

	// generate lotto numbers
	int fd = open("/dev/urandom", O_RDONLY);
	if(fd==-1){
		printf("error. tell admin\n");
		exit(-1);
	}
	unsigned char lotto[6];
	if(read(fd, lotto, 6) != 6){
		printf("error2. tell admin\n");
		exit(-1);
	}
	for(i=0; i<6; i++){
		lotto[i] = (lotto[i] % 45) + 1;		// 1 ~ 45
	}
	close(fd);
	
	// calculate lotto score
	int match = 0, j = 0;
	for(i=0; i<6; i++){
		for(j=0; j<6; j++){
			if(lotto[i] == submit[j]){
				match++;
			}
		}
	}

	// win!
	if(match == 6){
		system("/bin/cat flag");
	}
	else{
		printf("bad luck...\n");
	}

}

void help(){
	printf("- nLotto Rule -\n");
	printf("nlotto is consisted with 6 random natural numbers less than 46\n");
	printf("your goal is to match lotto numbers as many as you can\n");
	printf("if you win lottery for *1st place*, you will get reward\n");
	printf("for more details, follow the link below\n");
	printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
	printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

	// menu
	unsigned int menu;

	while(1){

		printf("- Select Menu -\n");
		printf("1. Play Lotto\n");
		printf("2. Help\n");
		printf("3. Exit\n");

		scanf("%d", &menu);

		switch(menu){
			case 1:
				play();
				break;
			case 2:
				help();
				break;
			case 3:
				printf("bye\n");
				return 0;
			default:
				printf("invalid menu\n");
				break;
		}
	}
	return 0;
}
```

去掉无用的 exit 和 help 选项，程序主要的部分是 play 函数，可以看到首先会让用户随机输入六个字符，然后从 /dev/urandom 文件中获取 6 个字符作为随机数候选，然后通过模运算将随机数范围缩小到 1-45 当中。接着和用户输入的随机数进行对比，如果 match 数达到 6 个就猜中了。

首先认为这个随机数的获取可能是不随机的，因为它是从一个文件中读取，查找资料后知道：

> 在 linux 中本身就有两个文件 /dev/random 和 /dev/urandom。这两个文件都可以作为很好的随机数来源，但是 /dev/urandom 是一个伪随机生成器，而 /dev/random 的随机性要比前者强，几乎是真正的随机数生成器，所以前者的安全性相对较低，但是后者会被阻塞。

所以这里从 /dev/urandom 中读取的是随机的 6 个字符，无法预测，每一次也不相同。那么文章一定在下面，观察最终验证的代码片段可以发现，验证的逻辑非常奇怪：

```c
for(i=0; i<6; i++){
    for(j=0; j<6; j++){
        if(lotto[i] == submit[j]){
            match++;
        }
    }
}
```

这一段实际上我觉得它的目的应该是要让每一位都对应相等，但是很可惜的是这段代码的意图是循环检查 lotto 的每一位和 submit 的每一位的相等情况，一共要检查 36 次。我们很容易想到构造一个 6 位都相等的密码，这样只要猜中一位，也就满足了最后的成功条件。所以这道题其实用暴力也可以破解，我就是用暴力破解了，但是写一个脚本来反复测试也是可行的，比如我反复尝试了 '\*\*\*\*\*\*'。