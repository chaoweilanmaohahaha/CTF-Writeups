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
15. `strlen( )`所计算的长度并不受数组长度的限制，而是以`\0`作为结束的标志，`strcpy( )`同理
16. `&运算`：按位与运算，二者都是1时为1，其余情况为0。
17. `remove( )`：原型`int remove(char * filename)`，用于删除指定的文件。
18. `fseek( )`：原型`int fseek(FILE * stream, long offset, int whence)`，参数offset 为根据参数whence 来移动读写位置的位移数。参数 whence 为下列其中一种:SEEK_SET 从距文件开头offset 位移量为新的读写位置；SEEK_CUR 以目前的读写位置往后增加offset 个位移量；SEEK_END 将读写位置指向文件尾后再增加offset 个位移量；当whence 值为SEEK_CUR 或SEEK_END 时, 参数offset 允许负值的出现。
19. `db`单个字节，`dw`两个字节，`dd`四个字节。
20. 查看数据堆栈时用`Hex View`比较。
21. `__m128i _mm_load_si128(__m128i *p);`方法是取得后面参数地址128bit的值
22. `void _mm_storeu_si128 (__m128i *p, __m128i a);`是将第二个参数的值放入第一个参数地址中
23. 假设一个十六进制数0x12345678，大端的存储方式是：12,34,56,78，然后读取的时候也是从前往后读，小端的存储方式是：78,56,34,12，然后读取的时候是从后往前读取
24. `char* setlocale (int category, const char* locale);`：函数既可以用来对当前程序进行地域设置（本地设置、区域设置），也可以用来获取当前程序的地域设置信息，使用setlocale需要两个参数，第一个参数category用来设置地域设置的影响范围，第二个参数locale用来设置地域设置的名称（字符串），也就是设置为哪种地域，对于不同的平台和不同的编译器，地域设置的名称可能会不同。
25. `wchar_t`是128bit类型的数据，ASCLL可用`char`表示，UNICODE需要`wchar_t`表示。
