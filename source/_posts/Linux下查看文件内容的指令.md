---
title: Linux下查看文件内容的指令
tags:
  - linux
categories: uncategorized
abbrlink: 37bc6184
date: 2018-06-21 16:58:53
---

## Linux下查看文件内容的命令
### 查看文件内容的命令：

vi      编辑方式查看，可修改

cat     显示全部文件内容，由第一行开始显示内容，并将所有内容输出

tac     从最后一行倒序显示内容，并将所有内容输出

more    根据窗口大小，分页显示文件内容

less    和more类似，但其优点可以往前翻页，而且进行可以搜索字符

head    只显示头几行

tail    只显示最后几行

nl      类似于cat -n，显示时输出行号

tailf   类似于tail -f 