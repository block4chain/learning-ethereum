---
description: 不同与Bitcoin的UTXO模型，以太坊基于帐户模型。
---

# 帐户\(Account\)

{% code title="core/state/state\_object.go" %}
```go
type Account struct {
	Nonce    uint64  //该帐户发出的交易序列号，单调递增
	Balance  *big.Int //该帐户的余额
	Root     common.Hash // 该帐户的状态存储, 一棵mpt树, root存储树根hash
	CodeHash []byte //合约帐户对应的合约代码hash，外部帐户该值为空
}
```
{% endcode %}

## 帐户地址

以太方帐户地址是一个20B字节数组。

{% code title="common/types.go" %}
```go
const (
	// AddressLength is the expected length of the address
	AddressLength = 20
)
// Address represents the 20 byte address of an Ethereum account.
type Address [AddressLength]byte
```
{% endcode %}

## 帐户类型

以太坊中存在两种类型的帐户: 外部帐户和合约帐户。

### 外部帐户

外部帐户，即用户帐户，由用户的私钥控制。

用户帐户的地址由用户的公钥哈希得出

{% tabs %}
{% tab title="accounts/keystore/key.go" %}
```go
//生成帐户公私钥, 基于secp256k1椭圆曲线, rand随机数生成私钥
func newKey(rand io.Reader) (*Key, error) {
	privateKeyECDSA, err := ecdsa.GenerateKey(crypto.S256(), rand)
	if err != nil {
		return nil, err
	}
	return newKeyFromECDSA(privateKeyECDSA), nil
}
//生成帐户的key
func newKeyFromECDSA(privateKeyECDSA *ecdsa.PrivateKey) *Key {
	id := uuid.NewRandom()
	key := &Key{
		Id:         id,
		Address:    crypto.PubkeyToAddress(privateKeyECDSA.PublicKey), //见crypto.go文件
		PrivateKey: privateKeyECDSA,
	}
	return key
}
```
{% endtab %}

{% tab title="crypto/crypto.go" %}
```go
//根据帐户公钥生成帐户20位地址, Keccak256返回公钥32B哈希值
func PubkeyToAddress(p ecdsa.PublicKey) common.Address {
	pubBytes := FromECDSAPub(&p)
	return common.BytesToAddress(Keccak256(pubBytes[1:])[12:])
}
```
{% endtab %}
{% endtabs %}

### 合约帐户

合约帐户由以太坊中运行的智能合约控制。

创建合约时，以太坊会为合约创建一个合约帐户，合约帐户地址由_**创建者的帐户地址**_和_**创建者帐户Nonce**_哈希生成

{% tabs %}
{% tab title="core/state\_processor.go" %}
```go
    // if the transaction created a contract, store the creation address in the receipt.
	if msg.To() == nil {
	    //vmenv.Context.Origin是合约创建者的帐户地址
	    //tx.Nonce是合约创建者发出创建合约交易时的Nonce
		receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce()) //参见crypto.go
	}
```
{% endtab %}

{% tab title="crypto/crypto.go" %}
```go
// CreateAddress creates an ethereum address given the bytes and the nonce
func CreateAddress(b common.Address, nonce uint64) common.Address {
	data, _ := rlp.EncodeToBytes([]interface{}{b, nonce})
	return common.BytesToAddress(Keccak256(data)[12:])
}
```
{% endtab %}
{% endtabs %}

#### EIP-1014

EIP-1014引入了一个新的创建智能合约的操作码，该操作码使用一种新的方式计算合约地址。新的计算方式让合约地址能够提前计算得出，便于合约在一些场景下的使用。

{% tabs %}
{% tab title="core/vm/evm.go" %}
```go
func (evm *EVM) Create2(caller ContractRef, code []byte, gas uint64, endowment *big.Int, salt *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
	codeAndHash := &codeAndHash{code: code}
	contractAddr = crypto.CreateAddress2(caller.Address(), common.BigToHash(salt), codeAndHash.Hash().Bytes())
	return evm.create(caller, codeAndHash, gas, endowment, contractAddr)
}
```
{% endtab %}

{% tab title="crypto/crypto.go" %}
```go
func CreateAddress2(b common.Address, salt [32]byte, inithash []byte) common.Address {
	return common.BytesToAddress(Keccak256([]byte{0xff}, b.Bytes(), salt[:], inithash)[12:])
}
```
{% endtab %}
{% endtabs %}

## 参考资料

* [https://github.com/ethereum/wiki/wiki/White-Paper\#ethereum-accounts](https://github.com/ethereum/wiki/wiki/White-Paper#ethereum-accounts)
* https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1014.md

