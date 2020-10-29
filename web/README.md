# php的小tips
## php的别名
php2, php3, php4, php5, phps, pht, phtm, phtml  
这些别名，尤其在绕过后缀过滤时用得到。  
## php内置协议
### php://file — 访问本地文件系统
#### 说明
文件系统是PHP使用的默认封装协议，展现了本地文件系统。当指定了一个相对路径（不以/、\、\\或 Windows 盘符开头的路径）提供的路径将基于当前的工作目录。在很多情况下是脚本所在的目录，除非被修改了。 
#### 用法
- /path/to/file.ext
- relative/path/to/file.ext
- fileInCwd.ext
- C:/path/to/winfile.ext
- C:\path\to\winfile.ext
- \\smbserver\share\path\to\winfile.ext
- file:///path/to/file.ext
- php://filter:/resource=http://www.baidu.com
#### 参数列表
- read：读取
- write：写入
- resource：数据来源(必须的)
#### read的参数值
- string.strip_tags：将数据流中的所有html标签清除
- string.toupper：将数据流中的内容转换为大写
- string.tolower：将数据流中的内容转换为小写
- convert.base64-encode：将数据流中的内容转换为base64编码
- convert.base64-decode：与上面对应解码
#### 例子
http://220.249.52.133:53620/index.php?page=php://filter/read=convert.base64-encode/resource=index.php  
查看该网页的源码。需要先用base64编码，否则会在每一个page的地方出现index.php的页面。  
  
## 一些系统函数
### 查看php版本信息
phpinfo()
### 执行系统外部命令
#### exec()
function exec(string $command, array[optional] $output, int[optional] $return_value)  
exec 执行系统外部命令时不会输出结果，而是返回结果的最后一行，如果你想得到结果你可以使用第二个参数，让其输出到指定的数组，此数组一个记录代表输出的一行，即如果输出结果有20行，则这个数组就有20条记录，所以如果你需要反复输出调用不同系统外部命令的结果，你最好在输出每一条系统外部命令结果时清空这个数组，以防混乱。第三个参数用来取得命令执行的状态码，通常执行成功都是返回０。  
#### passthru()
function passthru(string $command, int[optional] $return_value)  
passthru 与 system 一样，直接将结果输出到浏览器，不需要使用 echo 或 return 来查看结果，但 passthru 不返回任何值，且其可以输出二进制，比如图像数据。  
#### system()
function system(string $command, int[optional] $return_value)  
system 和 exec 的区别在于 system 在执行系统外部命令时，直接将结果输出到浏览器，不需要使用 echo 或 return 来查看结果，如果执行命令成功则返回 true，否则返回 false。第二个参数与 exec 第三个参数含义一样。  
#### 反撇号(`)和shell_exec()
shell_exec() 函数实际上仅是反撇号 (`) 操作符的变体  
shell_exec(string $command)  
函数的返回值即是命令执行的输出，如果执行过程中发生错误或者进程不产生输出，则返回 NULL。因此需要 echo 或 return 来查看结果。
