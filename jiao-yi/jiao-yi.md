---
description: 交易代表以太坊世界状态的一次状态迁移。以太坊中所有交易的执行都需要消耗一定的燃料(Gas)，交易发起方可以指定Gas的单价，并设定最大的Gas消耗量。
---

# 交易

## 定义

{% tabs %}
{% tab title="core/types/transaction.go" %}
```go
type Transaction struct {	data txdata	// caches	hash atomic.Value	size atomic.Value	from atomic.Value}type txdata struct {	//帐户交易计数	AccountNonce uint64          `json:"nonce"    gencodec:"required"`	//Gas单价	Price        *big.Int        `json:"gasPrice" gencodec:"required"`	//Gas上限	GasLimit     uint64          `json:"gas"      gencodec:"required"`	//交易接收方帐户地址	Recipient    *common.Address `json:"to"       rlp:"nil"`	//交易转帐金额	Amount       *big.Int        `json:"value"    gencodec:"required"`	//交易附加数据	Payload      []byte          `json:"input"    gencodec:"required"`	// 交易secp256k1签名数据	V *big.Int `json:"v" gencodec:"required"`	R *big.Int `json:"r" gencodec:"required"`	S *big.Int `json:"s" gencodec:"required"`	// This is only used when marshaling to JSON.	Hash *common.Hash `json:"hash" rlp:"-"`}
```
{% endtab %}
{% endtabs %}



