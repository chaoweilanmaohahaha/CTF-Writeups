# cookie
## 题目描述：X老师告诉小宁他在cookie里放了些东西，小宁疑惑地想：‘这是夹心饼干的意思吗？’

### 思路

http://220.249.52.133:53983  
点进题目链接：  
![avatar](./picture/cookie_1.png)

（废话，我当然知道。）  
用 F12 打开开发者工具，定格到 network 界面，然后刷新一下网页，寻找和 cookie 相关的信息。  
![avatar](./picture/cookie_2.png)

行啊，整了个 cookie.php 的文件，访问一下试试：  
http://220.249.52.133:53983/cookie.php  
![avatar](./picture/cookie_3.png)

哦？又来？再度进入开发者工具，然后刷新页面，查看 cookie.php 的 http response headers：  
![avatar](./picture/cookie_4.png)  
轻松获得 flag。
