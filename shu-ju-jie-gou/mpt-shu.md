---
description: >-
  MPT树(Merkle Patricia Trie树)是Ethereum用来存储键值(key,
  value)对的数据结构，它提供了一种密码学证明，保证数据不会篡改。MPT树的查找、插入、删除操作的时间复杂度是O(log(n))，相比红黑树也易于理解。
---

# MPT树

## 定义

![](../.gitbook/assets/mpt-tree.png)

### 结点类型

#### 叶子结点\(Leaf\)

一种终结结点，代表唯一确定一个\(key, value\)键值对:

* 拥有一个nibble数组，采用HP编码，并且终结标志置1。遍历到该结点所经历的nibble序列加上该结点的nibble数组唯一的构成key.
* 结点拥有的值为key对应的值

#### 扩展结点\(Extension\)

一种非终结结点。代表两个不同的key的共享的一段nibble数组:

* 拥有一个nibble数组, 采用HP编码，并且终结标志置0。nibble数组为至少两个不同的key的共享部分。
* 结点拥有一个值，该值为一个hash，指向其它分支结点。

#### 分支结点\(Branch\)

分支结点是一个Trie树的结点:

* 结点的度为17，其中16个用于索引\(nibble的取值空间\)其它结点，第17个用于存放分支结点的值\(此时结点是一个终结结点\)
* 如果结点有值，则该分支结点也是一个终结结点。

### 数学定义

![](../.gitbook/assets/selection_009.png)

![](../.gitbook/assets/selection_007.png)

![](../.gitbook/assets/selection_008.png)

* 当结点的RLP编码后的数据长度不超过32B, 则直接存储结点本身
* 当结点的RLP编码后的数据长度超过32B，则引用该结点的哈希值

## 源码解析

### 结点类型

```go
type node interface {
	fstring(string) string
	cache() (hashNode, bool)
}

type nodeFlag struct {
	hash  hashNode // 结点的hash缓存
	dirty bool     //脏结点标志
}

type (
    //分支结点
	fullNode struct {
		Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
		flags    nodeFlag
	}
	//扩展/叶子结点
	shortNode struct {
		Key   []byte
		Val   node
		flags nodeFlag
	}
	
	hashNode  []byte  //代表一个哈希
	valueNode []byte  //代表一个值
)
```

### 结点解析

* 解析分支结点

```go
//分支结点
func decodeFull(hash, elems []byte) (*fullNode, error) {
	n := &fullNode{flags: nodeFlag{hash: hash}}
	//分支结点包含16个分支
	for i := 0; i < 16; i++ {
		cld, rest, err := decodeRef(elems)  //解析分支引用
		if err != nil {
			return n, wrapError(err, fmt.Sprintf("[%d]", i))
		}
		n.Children[i], elems = cld, rest
	}
	//解析分支结点的值
	val, _, err := rlp.SplitString(elems)
	if err != nil {
		return n, err
	}
	if len(val) > 0 {
		n.Children[16] = append(valueNode{}, val...)
	}
	return n, nil
}
```

* 解析扩展/叶子结点

```go
func decodeShort(hash, elems []byte) (node, error) {
	kbuf, rest, err := rlp.SplitString(elems)
	if err != nil {
		return nil, err
	}
	flag := nodeFlag{hash: hash}
	key := compactToHex(kbuf)
	if hasTerm(key) {
		//设置终结标识被设置，所以是一个叶子结点
		val, _, err := rlp.SplitString(rest)
		if err != nil {
			return nil, fmt.Errorf("invalid value node: %v", err)
		}
		return &shortNode{key, append(valueNode{}, val...), flag}, nil
	}
	r, _, err := decodeRef(rest)
	if err != nil {
		return nil, wrapError(err, "val")
	}
	return &shortNode{key, r, flag}, nil
}
```

* 子结点引用

分支结点、扩展结点和叶子结点都会引用其它值结点或哈希结点，Ethereum采用一个decodeRef方法处理

```go
func decodeRef(buf []byte) (node, []byte, error) {
	kind, val, rest, err := rlp.Split(buf)
	if err != nil {
		return nil, buf, err
	}
	switch {
	case kind == rlp.List:
		//当前子结点RLP编码后的数据长度小于32字节则直接嵌套在父结点中
		if size := len(buf) - len(rest); size > hashLen {
			err := fmt.Errorf("oversized embedded node (size is %d bytes, want size < %d)", size, hashLen)
			return nil, buf, err
		}
		n, err := decodeNode(nil, buf)
		return n, rest, err
	case kind == rlp.String && len(val) == 0:
		// empty node
		return nil, rest, nil
	case kind == rlp.String && len(val) == 32:
		return append(hashNode{}, val...), rest, nil
	default:
		return nil, nil, fmt.Errorf("invalid RLP string size %d (want 0 or 32)", len(val))
	}
}
```

### Trier树

#### 定义

```go
type Trie struct {
	db   *Database  //Trie数据存储
	root node  //根结点索引, root可以为空，空root意味是空树
}
//公开方法
func (t *Trie) Get(key []byte) []byte
func (t *Trie) TryGet(key []byte) ([]byte, error)
func (t *Trie) Update(key, value []byte)
func (t *Trie) TryUpdate(key, value []byte) error
func (t *Trie) Delete(key []byte)
func (t *Trie) TryDelete(key []byte) error
func (t *Trie) Hash() common.Hash
func (t *Trie) Commit(onleaf LeafCallback) (root common.Hash, err error)
```

#### 插入

查找

删除

## 参考资料

* [https://github.com/ethereum/wiki/wiki/Patricia-Tree\#tries-in-ethereum](https://github.com/ethereum/wiki/wiki/Patricia-Tree#tries-in-ethereum)
* [http://www.cnspirit.com/2018/04/go-ethereum-4.html](http://www.cnspirit.com/2018/04/go-ethereum-4.html)
* [https://ethfans.org/toya/articles/588](https://ethfans.org/toya/articles/588)
* 以太坊黄皮书

