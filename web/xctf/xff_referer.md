# xff_referer
## 题目描述
X老师告诉小宁其实xff和referer是可以伪造的。  
## 思路
http://220.249.52.133:30860  
点开题目链接：  
![avatar](./picture/xff_referer_1.png)  

可以，需要伪装 IP ，那么我们用 burp 去搞一下：  
![avatar](./picture/xff_referer_2.png)  

继续给出下一步的指令：  
![avatar](./picture/xff_referer_3.png)  

这个需要伪装 Referer：  
![avatar](./picture/xff_referer_4.png)  

轻松得到 flag：  
![avatar](./picture/xff_referer_5.png)  
