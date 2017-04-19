---
layout:     post
title:      pwnable.kr Toddlers Bottle [fd] 
subtitle:   pwnable.kr writeup series 02
date:       2017-04-06
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

