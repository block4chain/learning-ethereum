# 节点

## 地址表示

### Enode URL

p2p网络中节点地址采用`enode url`格式

```text
格式: enode://<NodeId>@<IP>:<TCP PORT>?discport=<UDP PORT>
示例: 
enode://6f8a80d14311c39f35f516fa664deaaaa13e85b2f7493f37f6144d86991ec012937307647bd3b9a82abe2974e1407241d54947bbb39763a4cac9f77166ad92a0@10.3.58.6:30303?discport=30301
```

### Ethereum Node Record

[EIP-778](https://eips.ethereum.org/EIPS/eip-778)引入的新的节点表示方法:

```text
示例: 
enr:-IS4QHCYrYZbAKWCBRlAy5zzaDZXJBGkcnh4MHcBFZntXNFrdvJjX04jRzjzCBOonrkTfj499SZuOh8R33Ls8RRcy5wBgmlkgnY0gmlwhH8AAAGJc2VjcDI1NmsxoQPKY0yuDUmstAHYpMa2_oxVtw0RW_QAdpzBQA8yWM0xOIN1ZHCCdl8
```

## encode.Node类型

p2p网络节点用类型`encode.Node`表示

{% tabs %}
{% tab title="encode.Node" %}
{% code title="p2p/encode/node.go" %}
```go
type ID [32]byte

type Node struct {
	r  enr.Record
	id ID
}
```
{% endcode %}
{% endtab %}

{% tab title="enr.Record" %}
{% code title="p2p/enr/enr.go" %}
```go
type Record struct {
	seq       uint64 // sequence number
	signature []byte // the signature
	raw       []byte // RLP encoded record
	pairs     []pair // sorted list of all key/value pairs
}

type pair struct {
	k string
	v rlp.RawValue
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

