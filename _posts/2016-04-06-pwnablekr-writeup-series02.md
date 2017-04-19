---
layout:     post
title:      pwnable.kr WriteupSeries02 
subtitle:   Toddlers Bottle [col]
date:       2016-04-06
author:     jiayy
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - pwnable.kr
    - Toddlers Bottle
    - writeup
---

按照题目提示登陆 ssh　后台, 密码是　guest

```swift
ssh col@pwnable.kr -p2222
```
登陆后，在当前目录有三个文件: 
'col.c' 是代码文件
'col' 是一个可执行文件
'flag' 文件是我们的目标．

在当前目录执行 'ls -l' 
发现'flag'文件无法直接读取，'col' 文件带有's'标志位，且跟　'flag' 文件属于同一个用户'col_pwn', 思路很直接，就是通过执行 'col' 帮我们读取'flag'

```swift
col@ubuntu:~$ ls -l
total 16
-r-sr-x--- 1 col_pwn col     7341 Jun 11  2014 col
-rw-r--r-- 1 root    root     555 Jun 12  2014 col.c
-r--r----- 1 col_pwn col_pwn   52 Jun 11  2014 flag
```

要达到这个目的需要看 'col.c' 的代码，如下:

```swift
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

这个代码的作用是从执行参数读取20个字节，
将读到的字符串强制转为 int 数组，由于　sizeof(int) = 4, 
所以20个字节的字符串可以转为5个int，将它们的值加起来等于　hashcode　即可．

首先，我们需要将　hashcode 拆分为 5 个数的和，只需要除以 5 , 然后将余数
加到最后一个数上面即可．

```swift
r = 0x21DD09EC
a = r / 5
b = r - a * 4
print hex(a), hex(b)
```

结果是0x6c5cec8 0x6c5cecc

接下去，我们需要构造参数，利用python的print函数可以输出指定二进制内容，

```swift
python -c 'print("\x6c\xc5\xce\xc8"*4 + \x6c\xc5\xce\xcc)'
```
上述输出内容依然不对，原因是int是4个字节一起取值，需要根据小端排序将char数组重排一下,

```swift
python -c 'print("\xc8\xce\xc5\x06"*4 + \xcc\xce\xc5\x06)'
```

最后的解题方法如下：

```swift
./col $(python -c 'print ("\xc8\xce\xc5\x06" * 4 + "\xcc\xce\xc5\x06")')
```

#### 参考:

- [pwnable](http://pwnable.kr/play.php)
