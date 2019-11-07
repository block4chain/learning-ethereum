---
description: >-
  区块是以太坊区块链的核心组成部分，区块与区块之间通过哈希引用串接在一起构成区块链这样一种数据结构。区块也是交易的集合，以太坊状态机以区块作为原子执行单元。区块通过共识算法在全网达成共识。
---

# 区块

![](../.gitbook/assets/yi-tai-fang-zu-cheng.jpg)

todo: 以区块为研究核心，探究区块的组成、创建、执行、到状态提交等思路研究以太坊

## 区块结构

以太坊区块\(Block\)由三部分组成:

* 区块头
* 叔块头列表
* 交易列表

{% tabs %}
{% tab title="core/types/block.go" %}
```go
//区块type Block struct {	header       *Header   //区块块	uncles       []*Header  //叔块头列表	transactions Transactions  //交易列表	//省略代码}//区块头type Header struct {	ParentHash  common.Hash   //父块hash	UncleHash   common.Hash    //叔块头列表rlp编码的hash值	Coinbase    common.Address  //矿工地址	Root        common.Hash   //statedb的hash	TxHash      common.Hash     //交易hash	ReceiptHash common.Hash  	Bloom       Bloom 	Difficulty  *big.Int  	Number      *big.Int  	GasLimit    uint64 	GasUsed     uint64	Time        uint64  	Extra       []byte 	MixDigest   common.Hash 	Nonce       BlockNonce   //pow随机值}
```
{% endtab %}
{% endtabs %}

## 块号、父块

区块链中的每个块都有块号唯一标识`Header.Number`，块号连续递增。

每个块会引用前一个块号作为自己的父块，并将父块的哈希值`Header.ParentHash`保存在自己的区块头。子块保存父块的哈希值可以保证父块没有篡改。

## 叔块

在以太坊中，除了引用父块外，还可以引用一些非主链的合法块，即叔块`Block.uncles`，叔块的引入可以带来以下好处:

* 提升出块效率，提高共识速度
* 减少算力浪费
* 提升安全性

## 费用

`Header.GasLimit`指定当前块能消耗的最大Gas数，根据父块的GasLimit计算得出:

{% tabs %}
{% tab title="core/block\_validator.go" %}
```go
 //params.GasLimitBoundDivisor: 1024 //gasFloor, gasCeil是配置项，默认都是8000000func CalcGasLimit(parent *types.Block, gasFloor, gasCeil uint64) uint64 {	// contrib = (parentGasUsed * 3 / 2) / 1024	contrib := (parent.GasUsed() + parent.GasUsed()/2) / params.GasLimitBoundDivisor	// decay = parentGasLimit / 1024 -1	decay := parent.GasLimit()/params.GasLimitBoundDivisor - 1	limit := parent.GasLimit() - decay + contrib	if limit < params.MinGasLimit {		limit = params.MinGasLimit	}	if limit < gasFloor {		limit = parent.GasLimit() + decay		if limit > gasFloor {			limit = gasFloor		}	} else if limit > gasCeil {		limit = parent.GasLimit() - decay		if limit < gasCeil {			limit = gasCeil		}	}	return limit}
```
{% endtab %}
{% endtabs %}

`Header.GasUsed`是当前块中所有的交易消耗的Gas总数，并且需要满足:

$$
Block.GasUsed <= Block.GasLimit
$$

## 共识

以太坊当前以POW做为共识算法，区块头中包含了共识相关的信息。

`Header.Difficulty`定义了当前区块的POW目标难度，这个值根据父块和时间计算出来.

```go
//参考君士坦丁堡版本//bombDelay: 5000000func makeDifficultyCalculator(bombDelay *big.Int) func(time uint64, parent *types.Header) *big.Int {	bombDelayFromParent := new(big.Int).Sub(bombDelay, big1)	return func(time uint64, parent *types.Header) *big.Int {		// https://github.com/ethereum/EIPs/issues/100.		bigTime := new(big.Int).SetUint64(time)		bigParentTime := new(big.Int).SetUint64(parent.Time)		x := new(big.Int)		y := new(big.Int)		// (2 if len(parent_uncles) else 1) - (block_timestamp - parent_timestamp) // 9		x.Sub(bigTime, bigParentTime)		x.Div(x, big9)		if parent.UncleHash == types.EmptyUncleHash {			x.Sub(big1, x)		} else {			x.Sub(big2, x)		}		// max((2 if len(parent_uncles) else 1) - (block_timestamp - parent_timestamp) // 9, -99)		if x.Cmp(bigMinus99) < 0 {			x.Set(bigMinus99)		}		// parent_diff + (parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((timestamp - parent.timestamp) // 9), -99))		y.Div(parent.Difficulty, params.DifficultyBoundDivisor)		x.Mul(y, x)		x.Add(parent.Difficulty, x)		// minimum difficulty can ever be (before exponential factor)		if x.Cmp(params.MinimumDifficulty) < 0 {			x.Set(params.MinimumDifficulty)		}		// calculate a fake block number for the ice-age delay		// Specification: https://eips.ethereum.org/EIPS/eip-1234		fakeBlockNumber := new(big.Int)		if parent.Number.Cmp(bombDelayFromParent) >= 0 {			fakeBlockNumber = fakeBlockNumber.Sub(parent.Number, bombDelayFromParent)		}		// for the exponential factor		periodCount := fakeBlockNumber		periodCount.Div(periodCount, expDiffPeriod)		// the exponential factor, commonly referred to as "the bomb"		// diff = diff + 2^(periodCount - 2)		if periodCount.Cmp(big1) > 0 {			y.Sub(periodCount, big2)			y.Exp(big2, y, nil)			x.Add(x, y)		}		return x	}}
```

`Header.MixDigest`是区块头的Keccak256哈希值, `Header.Nonce`是一个可变随机数，通过改过Nonce值从而使Pow结果达到目标难度

## 交易

### 交易树

区块包含一组交易,  区块头字段`Header.TxHash`是所有交易的MPT树的根哈希，防止交易被篡改。

{% tabs %}
{% tab title="core/types/block.go" %}
```go
func NewBlock(header *Header, txs []*Transaction, uncles []*Header, receipts []*Receipt) *Block {	b := &Block{header: CopyHeader(header), td: new(big.Int)}	// TODO: panic if len(txs) != len(receipts)	if len(txs) == 0 {		b.header.TxHash = EmptyRootHash	} else {		b.header.TxHash = DeriveSha(Transactions(txs))		b.transactions = make(Transactions, len(txs))		copy(b.transactions, txs)	}	//省略代码}
```
{% endtab %}

{% tab title="core/types/derive\_sha.go" %}
```go
func DeriveSha(list DerivableList) common.Hash {	keybuf := new(bytes.Buffer)	trie := new(trie.Trie)	for i := 0; i < list.Len(); i++ {		keybuf.Reset()		rlp.Encode(keybuf, uint(i))		trie.Update(keybuf.Bytes(), list.GetRlp(i))	}	return trie.Hash()}
```
{% endtab %}
{% endtabs %}

### 状态数据库

区块中包含的交易在执行过程中会修改区块链的状态，状态数据库中保存了当前块执行完后的最新状态值，`Header.Root`记录并索引了当前的状态数据库

## 交易结果

### 交易收据

每笔交易执行完成后会生成一个交易收据:

{% tabs %}
{% tab title="core/types/receipt.go" %}
```go
type Receipt struct {	PostState         []byte `json:"root"`  //交易完成后的中间状态数据库索引	Status            uint64 `json:"status"` //交易执行结果	//执行区块的交易过程中执行完本交易消耗的总Gas	CumulativeGasUsed uint64 `json:"cumulativeGasUsed" gencodec:"required"` 	//本交易Log的Bloom过滤器	Bloom             Bloom  `json:"logsBloom"         gencodec:"required"`	//本交易产生触发的Log	Logs              []*Log `json:"logs"              gencodec:"required"`	//交易hash	TxHash          common.Hash    `json:"transactionHash" gencodec:"required"`	//执行本交易的合约	ContractAddress common.Address `json:"contractAddress"`	//执行本交易消耗的Gas	GasUsed         uint64         `json:"gasUsed" gencodec:"required"`	//	BlockHash        common.Hash `json:"blockHash,omitempty"`	BlockNumber      *big.Int    `json:"blockNumber,omitempty"`	TransactionIndex uint        `json:"transactionIndex"`}
```
{% endtab %}
{% endtabs %}

### 收据树

区块中所有的收据会形成一个MPT收据树，树根\(Root\)会存储在`Header.ReceiptHash`中保证交易执行结果不会被篡改。

{% tabs %}
{% tab title="core/types/block.go" %}
```go
func NewBlock(header *Header, txs []*Transaction, uncles []*Header, receipts []*Receipt) *Block {	b := &Block{header: CopyHeader(header), td: new(big.Int)}		//省略代码	if len(receipts) == 0 {		b.header.ReceiptHash = EmptyRootHash	} else {		b.header.ReceiptHash = DeriveSha(Receipts(receipts))  //收据树根		b.header.Bloom = CreateBloom(receipts)	}	//省略代码}
```
{% endtab %}
{% endtabs %}

### 交易事件

交易在执行过程\(合约执行\)中可以向外发出事件，所有的事件会构造一个Bloom过滤器进行索引，并将过滤器存放在`Header.Bloom`中

{% tabs %}
{% tab title="core/types/block.go" %}
```go
func NewBlock(header *Header, txs []*Transaction, uncles []*Header, receipts []*Receipt) *Block {	b := &Block{header: CopyHeader(header), td: new(big.Int)}		//省略代码	if len(receipts) == 0 {		b.header.ReceiptHash = EmptyRootHash	} else {		b.header.ReceiptHash = DeriveSha(Receipts(receipts))  //收据树根		b.header.Bloom = CreateBloom(receipts)  //构造Bloom过滤器	}	//省略代码}
```
{% endtab %}
{% endtabs %}

