---
layout:     post
title:      pwnable.kr Toddlers Bottle [fd] 
subtitle:   pwnable.kr writeup series 01
date:       2016-04-05
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
ssh fd@pwnable.kr -p2222
```
登陆后，在当前目录有三个文件: 
'fd.c' 是代码文件
'fd' 是一个可执行文件
'flag' 文件是我们的目标．

在当前目录执行 'ls -l' 
发现'flag'文件无法直接读取，'fd' 文件带有's'标志位，且跟　'flag' 文件属于同一个用户'fw_pwn', 思路很直接，就是通过执行 'fd' 帮我们读取'flag'

```swift
fd@ubuntu:~$ ls
fd  fd.c  flag
fd@ubuntu:~$ ls -l
total 16
-r-sr-x--- 1 fd_pwn fd   7322 Jun 11  2014 fd
-rw-r--r-- 1 root   root  418 Jun 11  2014 fd.c
-r--r----- 1 fd_pwn root   50 Jun 11  2014 flag
```

要达到这个目的需要看 'fd.c' 的代码，如下:

```swift
#include <stdlib.h>  
#include <string.h>  
char buf[32];  
int main(int argc, char* argv[], char* envp[]){  
    if(argc<2){  
        printf("pass argv[1] a number\n");  
        return 0;  
    }  
    int fd = atoi( argv[1] ) - 0x1234;  
    int len = 0;  
    len = read(fd, buf, 32);  
    if(!strcmp("LETMEWIN\n", buf)){  
        printf("good job :)\n");  
        system("/bin/cat flag");  
        exit(0);  
    }  
    printf("learn about Linux file IO\n");  
    return 0;  
  
}
```

这个代码的作用是从一个文件读取32个字节，如果读出来的内容开头是
字符串'LETMEWIN\n'则会执行'/bin/cat flag'．　
读取的文件fd是根据执行参数算出来的(fd = atoi( argv[1] ) - 0x1234),
所以我们让参数为　0x1234 (4660) 这样算出来的　fd = 0, 即　stdin,
随即在命令行输入字符串　'LETMEWIN' 回车即可 

#### 参考:

- [pwnable](http://pwnable.kr/play.php)
