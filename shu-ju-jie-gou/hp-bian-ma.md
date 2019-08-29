---
description: >-
  十六进制前缀(Hex-Prefix)编码用来将一个半字节(nibble)序列编码为一个字节数组。HP编码支持一个附加标记(flag),
  可以用于特定场合对数据类型进行区分.
---

# HP编码

HP编码的数学定义:

![](../.gitbook/assets/selection_006.png)

其中:

* Y是半字节序列集合
* t是一个附加标记

HP编码后的字节数组中第一个字节hp\[0\]=\(high\_nibble, low\_nibble\)具有以下结构:

* high\_nibble的右起第一位记录了编码前半字节数组的奇偶性
* high\_nibble的右起第二位记录了HP编码的附加标记是否被设置
* low\_nibble的所有位均置0



