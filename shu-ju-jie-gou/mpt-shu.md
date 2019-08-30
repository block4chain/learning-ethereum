---
description: >-
  MPT树(Merkle Patricia Trie树)是Ethereum用来存储键值(key,
  value)对的数据结构，它提供了一种密码学证明，保证数据不会篡改。MPT树的查找、插入、删除操作的时间复杂度是O(log(n))，相比红黑树也易于理解。
---

# MPT树

定义

![](../.gitbook/assets/mpt-tree.png)

### 结点类型

#### 叶子结点\(Leaf\)

一种终结结点，代表唯一确定一个\(key, value\)键值对:

* 拥有一个nibble数组，采用HP编码，并且终结标志置1。遍历到该结点所经历的nibble序列加上该结点的nibble数组唯一的构成key.
* 结点拥有的值为key对应的值

#### 扩展结点\(Extension\)

一种非终结结点。

#### 分支结点\(Branch\)



