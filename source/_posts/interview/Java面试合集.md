---
title: Java面试合集
category: interview
---

# Java面试合集



### String底层为和将char[]改成了byte[]

在 Java 9 之前，String 的底层实现使用的是 char[]数组来存储字符数据。然而，随着 Unicode 编码的普及和多语言环境的需求增加，char 类型无法满足所有情况下的字符表示要求。因此，在 Java 9 中，String 的底层实现被修改为使用 byte[]数组来存储字符数据。