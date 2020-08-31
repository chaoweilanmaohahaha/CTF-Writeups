一些被遗忘的知识点：
1. strcpy( )：`char *strcpy(char* dest, const char *src);`，把从src地址开始且含有NULL结束符的字符串复制到以dest开始的地址空间。
2. strcat( )：`extern char *strcat(char *dest, const char *src);`，把src所指向的字符串（包括“\0”）复制到dest所指向的字符串后面（删除*dest原来末尾的“\0”）。
3. memset( )：`void *memset(void *s, int ch, size_t n);`，将s中当前位置后面的n个字节 （typedef unsigned int size_t ）用 ch 替换并返回s。
4. %x：以十六进制数形式输出整数。
5. %o：以八进制数形式输出整数。
6. %u：以十进制数输出unsigned型数据(无符号数)。
7. %e：以指数形式输出实数。
8. Python其它进制转十进制：`int(数字，进制类型)`例如`int('0x11',16)`，10进制转16进制`hex(10)`。
9. `int main(int argc, char* argv[])`：argc是命令行总的参数个数，argv[ ]是argc个参数，其中第0个参数是程序的全名
10. UPX脱壳：`upx.exe -d filename`
11. `uncompyle6 -o pcat.py pcat.pyc`反编译pyc文件
12. ord( ) 函数是 chr( ) 函数的配对函数，前者输入字符输出ASCLL码，后者可输入任何进制数字输出ASCLL码
13. Base64编码方式，`base64.b64encode( )`和`base64.b64decode( )`
14. `^`异或操作，当两对应的二进位相异时，结果为1
