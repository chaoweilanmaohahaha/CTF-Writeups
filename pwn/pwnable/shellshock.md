# shellshock

题目来源：https://pwnable.kr/

这道题题目很短可以先上代码：

```c
#include <stdio.h>
int main() {
	setresuid(getegid(), getegid(), getegid());
	setresgid(getegid(), getegid(), getegid());
	system("/home/shellshock/bash -c 'echo shock_me'");
	return 0;
}
```

先简单分析一下代码流程。前两个函数的目的是：

```
setresuid: 设置真实、有效、被保存的 uid;
setresgid: 设置真实、有效、被保存的 gid;
```

在 linux 存在三种用户 id 类型：

* real id(uid) ：这个是用户真实 id，而在系统中是用来决定某个进程属于谁，比如我们能够在 ls 中看到进程的的拥有者；
* effective id(eid)：这个是有效 id，是内核来判断当前的用户是否允许用户访问进程资源的标志，所以系统是使用这一个 id 来确定访问权限的，所以这个变量是操作系统层面上的；
* saved set-user-id(suid)：目前的理解是保存下来的 uid，这个是 euid 的备份。

在当中还有一个 suid 的概念，其实在之前也提到过，这个概念是相对于文件而言的，如果在文件权限中出现了 s，说明 suid 被打开了，也就是说当前的用户会临时拥有拥有文件的用户的权限。

这个的目的是为了后面想要拿到 shell 做铺垫。

这里真正的重头戏是 shellshock 这个漏洞，这道题考察的就是对这个漏洞的了解。

shellshock 漏洞是在 2014 年发布的，是一个相当严重的漏洞，漏洞编号 CVE-2014-6271。这个漏洞和 bash 对环境变量的解析过程有关。

因为在 bash 中是允许用户定义函数的，但是 bash 还有一种特有的功能，就是使用环境变量来定义函数。当我们在环境变量中定义一个以 (){ 开头的值，就会认为是一个导入函数的定义。**漏洞的问题在于 bash 会把函数定义后面的字符串当作是命令执行。**当用户顶一个环境变量为 [函数定义] + [任意命令] 时，此时会先去执行后面的命令。bash 在执行时会扫描环境变量，当 bash 解析完函数定义时，它自动导入的解析器会跳过函数，接着执行后面的代码。所以如果构造一下：

```
export X='() { :;}; bash'
```

则就能拿到系统的 bash。

这里还有两点要提及：

1. bash 中的脚本对于空格的要求很高，要注意；
2. shellshock 漏洞发生在 bash 4.3 版本以及之前，后来的版本进行了修复。