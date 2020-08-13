# blackjack

题目来源：https://pwnable.kr/

这道题是分析一个 C 语言代码，找到其中的漏洞，本身的 flag 并不在这个源代码中，符合题目的要求后就会出现 flag。

这个 C 代码实现的是一个 21 点的游戏，具体的流程就是玩家初始有 500 美金的赌注，随后和电脑玩 21 点，每一盘都会让你下赌注，然后题目的意思是，如果你成为了百万富翁就可以拿到 flag。直觉就是肯定不是靠一把把玩到百万富翁的，一定有捷径。跳过代码的一些菜单相关的部分，直接看核心的代码：

源代码在这里：http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html

```c
void play() //Plays game
{
     
     int p=0; // holds value of player_total
     int i=1; // counter for asking user to hold or stay (aka game turns)
     char choice3;
     
     cash = cash;
     cash_test();
     printf("\nCash: $%d\n",cash); //Prints amount of cash user has
     randcard(); //Generates random card
     player_total = p + l; //Computes player total
     p = player_total;
     printf("\nYour Total is %d\n", p); //Prints player total
     dealer(); //Computes and prints dealer total
     betting(); //Prompts user to enter bet amount
       
     while(i<=21) //While loop used to keep asking user to hit or stay at most twenty-one times
                  //  because there is a chance user can generate twenty-one consecutive 1's
     {
         if(p==21) //If user total is 21, win
         {
             printf("\nUnbelievable! You Win!\n");
             won = won+1;
             cash = cash+bet;
             printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
             dealer_total=0;
             askover();
         }
     
         if(p>21) //If player total is over 21, loss
         {
             printf("\nWoah Buddy, You Went WAY over.\n");
             loss = loss+1;
             cash = cash - bet;
             printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
             dealer_total=0;
             askover();
         }
     
         if(p<=21) //If player total is less than 21, ask to hit or stay
         {         
             printf("\n\nWould You Like to Hit or Stay?");
             
             scanf("%c", &choice3);
             while((choice3!='H') && (choice3!='h') && (choice3!='S') && (choice3!='s')) // If invalid choice entered
	         {                                                                           
                 printf("\n");
		         printf("Please Enter H to Hit or S to Stay.\n");
		         scanf("%c",&choice3);
	         }


	         if((choice3=='H') || (choice3=='h')) // If Hit, continues
	         { 
                 randcard();
                 player_total = p + l;
                 p = player_total;
                 printf("\nYour Total is %d\n", p);
                 dealer();
                  if(dealer_total==21) //Is dealer total is 21, loss
                  {
                      printf("\nDealer Has the Better Hand. You Lose.\n");
                      loss = loss+1;
                      cash = cash - bet;
                      printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
                      dealer_total=0;
                      askover();
                  } 
     
                  if(dealer_total>21) //If dealer total is over 21, win
                  {                      
                      printf("\nDealer Has Went Over!. You Win!\n");
                      won = won+1;
                      cash = cash+bet;
                      printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
                      dealer_total=0;
                      askover();
                  }
             }
             if((choice3=='S') || (choice3=='s')) // If Stay, does not continue
             {
                printf("\nYou Have Chosen to Stay at %d. Wise Decision!\n", player_total);
                stay();
             }
          }
             i++; //While player total and dealer total are less than 21, re-do while loop 
     } // End While Loop
} // End Function
```

别的不管，把目光聚焦在关于处理金额的函数 betting 上，因为其他函数都是处理随机的函数：

```c
int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);

 if (bet > cash) //If player tries to bet more money than player has
 {
		printf("\nYou cannot bet more money than you have.");
		printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
 }
 else return bet;
} // End Function
```

就只分析上面的流程了可以发现，用户输入一个赌注，作者先和用户持有的金额进行对比，看是否足够，如果不够就需要用户重新输入一次。

这里就有一个很明显的漏洞，可以说有两个很明显的漏洞。首先是从逻辑上而言，在用户第一次输高了赌注之后，作者只对其判断了一次，而第二次就不判断了，那么很明显用户可以在第二次输入一个非常高的赌注；

其二也是最容易利用的一个漏洞，那就是溢出，作者没有判断赌注的上限以及下限，这样一旦赌注溢出了，就直接作用在用户持有的金额上了，无论输赢金额都会变大，利用这个漏洞你就可以立马成为百万富翁了。