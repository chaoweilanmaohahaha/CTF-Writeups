# coin1

题目来源：https://pwnable.kr/

这道题是考察编程和算法的题目，先分析一下题面，本身这道题不需要我们读懂什么代码，就是玩一个小游戏，游戏规则是这样的：

这是一个从硬币堆里找出劣质硬币的游戏，劣质硬币的重量会比真实硬币轻 1 个单位，真实硬币重 10 个单位。首先会给出两个数 N 和 C，N 代表你当前硬币堆一共有多少个硬币，C 代表你当前有几次机会去查找。查找过程中你可以向程序询问某些序号的硬币的重量总和是多少，然后去判断劣质硬币在不在你询问的这些硬币中，最后你要给出你认为是劣质硬币的序号。

题目的要求是连续找出 100 个劣质硬币就算赢了。

这道题很明显地使用二分查找，一步一步缩小劣质硬币存在的范围，没有什么多说的，唯一的问题就在于如果是在本地使用 remote 去运行进程，会遇到超时问题，因为题目要求是在 60 s 内找出，所以需要扔到服务器上。不过服务器上似乎对 pwntools 不是很友好，因此最好是使用 socket 直接编程。

下面记录一下使用 pwntools 的程序吧：

```python
from pwn import *
import time

conn = remote("pwnable.kr", 9007)

print(conn.recv())
time.sleep(3)
while True:
    gamestr = conn.recv()
    print gamestr
    N = int(gamestr.split(' ')[0].split('=')[1])
    C = int(gamestr.split(' ')[1].split('=')[1])
    print N,C
    left = 0
    right = N-1
    weight = 0
    for i in range(int(C)):
        half = (left + right)/2
        tmpstr = ""
        for i in range(half - left+1):
            tmpstr += str(i+left) + " "
        print tmpstr
        conn.sendline(tmpstr)
        weight = int(conn.recv())
        print weight
        if weight%10 == 0:
            left = half+1
        else:
            right = half
    if weight == 10:
        conn.sendline(str(right))
    else:
        conn.sendline(str(left))
    print(conn.recv())
```

