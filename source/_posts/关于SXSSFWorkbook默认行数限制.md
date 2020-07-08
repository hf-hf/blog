---
title: 关于SXSSFWorkbook默认行数限制
tags:
  - poi
cover: /upload/homePage/20181017164430.jpg
abbrlink: 2d5a085d
categories: uncategorized
date: 2018-10-12 18:13:21
---
在使用SXSSFWorkbook时需要注意，若不指定rowAccessWindowSize，则默认窗体大小仅有100行，详情见源码。

```
/**
 * Specifies how many rows can be accessed at most via getRow().
 * When a new node is created via createRow() and the total number
 * of unflushed records would exceed the specified value, then the
 * row with the lowest index value is flushed and cannot be accessed
 * via getRow() anymore.
 */
public static final int DEFAULT_WINDOW_SIZE = 100;

public SXSSFWorkbook(XSSFWorkbook workbook){
    this(workbook, DEFAULT_WINDOW_SIZE);
}
```