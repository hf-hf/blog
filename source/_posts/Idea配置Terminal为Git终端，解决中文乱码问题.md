---
title: Idea配置Terminal为Git终端，解决中文乱码问题
tags:
  - idea
cover: /upload/homePage/20190104095503.jpg
abbrlink: f1df3c29
categories: uncategorized
date: 2019-01-04 11:12:17
---
在Idea中，Ctrl + Alt + S或者选择File -> Settings -> Tools -> Terminal，配置Shell Path为Git安装目录\bin\bash.exe即可。

![idea_terminal_git_1.png](/upload/ideaTerminalGit/idea_terminal_git_1.png)

若在Terminal中输入中文显示乱码，则需修改Git安装路径下etc的bash.bashrc文件，在最后添加如下两行：

```
export LANG="zh_CN.UTF-8"
export LC_ALL="zh_CN.UTF-8"
```

添加后，重新打开Terminal，再输入中文就显示正常了。



