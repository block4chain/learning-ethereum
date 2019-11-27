# 节点

## 节点ID（v4\)

v4版本的网络中，节点ID是一个32字节的字节数组

```go
type ID [32]byte
```

节点通过公钥计算得出:

```go
func PubkeyToIDV4(key *ecdsa.PublicKey) ID {
	e := make([]byte, 64)
	math.ReadBits(key.X, e[:len(e)/2])
	math.ReadBits(key.Y, e[len(e)/2:])
	return ID(crypto.Keccak256Hash(e))
}
```

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

解析方法:

{% code title="p2p/enode/node.go" %}
```go
func Parse(validSchemes enr.IdentityScheme, input string) (*Node, error) {
	//省略
	if !strings.HasPrefix(input, "enr:") {
		return nil, errMissingPrefix
	}
	//去掉`enr:`前辍，并base64解码
	bin, err := base64.RawURLEncoding.DecodeString(input[4:])
	if err != nil {
		return nil, err
	}
	var r enr.Record
	//将解码数据rlp反序列化成enr.Record实例
	if err := rlp.DecodeBytes(bin, &r); err != nil {
		return nil, err
	}
	//从enr.Record恢复Node数据
	return New(validSchemes, &r)
}
```
{% endcode %}

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

### 节点记录\(enr.Record\)

| 字段 | 含义 |
| :--- | :--- |
| seq | 版本号，当数据发生变化重新发布时递增这个值 |
| signature | sign\(\[seq, k, v, ...\]\) |
| raw | rlp\(\[sig, seq, k,v, ...\]\) |

### 节点Pair数据

| Key | Value | 描述 |
| :--- | :--- | :--- |
| id | v4 | 身份协议，当前是v4 |
| secp256k1 | compressed secp256k1 public key, 33 bytes | 压缩公钥数据 |
| ip | ipV4地址, 4字节 | 节点的IP地址 |
| tcp | TCP端口，4字节 | TCP端口，大端序 |
| udp | UDP端口，4字节 | UDP端口，大端序 |
| ip6 | ipv6地址，16字节 | 节点的IPV6地址 |
| tcp6 | ipv6特定的tcp端口地址，大端序 |  |
| udp6 | ipv6特定的udp端口地址，大端序 |  |

### 签名

```go
func SignV4(r *enr.Record, privkey *ecdsa.PrivateKey) error {
	// Copy r to avoid modifying it if signing fails.
	cpy := *r
	cpy.Set(enr.ID("v4"))
	cpy.Set(Secp256k1(privkey.PublicKey))

	h := sha3.NewLegacyKeccak256()
	rlp.Encode(h, cpy.AppendElements(nil))
	sig, err := crypto.Sign(h.Sum(nil), privkey)
	if err != nil {
		return err
	}
	sig = sig[:len(sig)-1] // remove v
	if err = cpy.SetSig(V4ID{}, sig); err == nil {
		*r = cpy
	}
	return err
}

func (r *Record) AppendElements(list []interface{}) []interface{} {
	list = append(list, r.seq)
	for _, p := range r.pairs {
		list = append(list, p.k, p.v)
	}
	return list
}
```

## 本地节点

### `LocalNode`结构

结构体`encode.LocalNode`代表本地节点:

{% code title="p2p/encode/localnode.go" %}
```go
type LocalNode struct {
	cur atomic.Value // holds a non-nil node pointer while the record is up-to-date.
	id  ID
	key *ecdsa.PrivateKey  //节点私钥
	db  *DB

	// everything below is protected by a lock
	mu        sync.Mutex
	seq       uint64
	entries   map[string]enr.Entry  //enr key/value对
	endpoint4 lnEndpoint
	endpoint6 lnEndpoint
}
type lnEndpoint struct {
	track                *netutil.IPTracker
	staticIP, fallbackIP net.IP
	fallbackUDP          int
}
```
{% endcode %}

在节点启动时，会创建`LocalNode`实例:

```go
func NewLocalNode(db *DB, key *ecdsa.PrivateKey) *LocalNode {
	ln := &LocalNode{
		id:      PubkeyToIDV4(&key.PublicKey),
		db:      db,
		key:     key,
		entries: make(map[string]enr.Entry),
		endpoint4: lnEndpoint{
			track: netutil.NewIPTracker(iptrackWindow, iptrackContactWindow, iptrackMinStatements),
		},
		endpoint6: lnEndpoint{
			track: netutil.NewIPTracker(iptrackWindow, iptrackContactWindow, iptrackMinStatements),
		},
	}
	ln.seq = db.localSeq(ln.id)  //从数据库中加载记录的enr seq号
	ln.invalidate()
	return ln
}
```

### sign方法

sign方法用于将`LocalNode`实例转变成节点`Node`实例

```go
func (ln *LocalNode) sign() {
	if n := ln.cur.Load().(*Node); n != nil {
		return // no changes
	}
	var r enr.Record
	for _, e := range ln.entries {
		r.Set(e)
	}
	ln.bumpSeq()  //增长序列号
	r.SetSeq(ln.seq)
	//用私钥对Record进行签名
	if err := SignV4(&r, ln.key); err != nil {
		panic(fmt.Errorf("enode: can't sign record: %v", err))
	}
	//生成Node实例
	n, err := New(ValidSchemes, &r)
	if err != nil {
		panic(fmt.Errorf("enode: can't verify local record: %v", err))
	}
	ln.cur.Store(n) //保存到缓存
}
```

