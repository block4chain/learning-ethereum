# 节点发现\(v5\)

## 网络层

### 连接

Ethereum传输层采用UDP协议，每个UDP连接在实现时用一个`conn`接口抽象，golang包`net.UDPConn`是它的一个实现:

{% tabs %}
{% tab title="p2p/discv5/udp.go" %}
```go
type conn interface {
	ReadFromUDP(b []byte) (n int, addr *net.UDPAddr, err error)
	WriteToUDP(b []byte, addr *net.UDPAddr) (n int, err error)
	Close() error
	LocalAddr() net.Addr
}
```
{% endtab %}
{% endtabs %}

`p2p.sharedUDPConn`也是它的实现，用来与v4版本的节点发现共用UDP连接，所有v4版本无法处理的包都会转给v5处理:

{% tabs %}
{% tab title="p2p/discv5/server.go" %}
```go
func (srv *Server) setupDiscovery() error {
    //省略部分代码
    if !srv.NoDiscovery {
			if srv.DiscoveryV5 {
			    unhandled = make(chan discover.ReadPacket, 100)
			    sconn = &sharedUDPConn{conn, unhandled}
		   }
		   //省略代码
		}
    if srv.DiscoveryV5 {
		var ntab *discv5.Network
		var err error
		if sconn != nil {
			ntab, err = discv5.ListenUDP(srv.PrivateKey, sconn, "", srv.NetRestrict)
		} else {
			ntab, err = discv5.ListenUDP(srv.PrivateKey, conn, "", srv.NetRestrict)
		}
		if err != nil {
			return err
		}
		if err := ntab.SetFallbackNodes(srv.BootstrapNodesV5); err != nil {
			return err
		}
		srv.DiscV5 = ntab
	}
	return nil
}
```
{% endtab %}
{% endtabs %}

### 协议

v5版本的节点发现网络协议是两层结构，外层协议是结构:

| 字段 | 大小\(字节\) | 描述 |
| :--- | :--- | :--- |
| 协议版本 | len\("temporary discovery v5"\) | 固定值: temporary discovery v5 |
| 签名 | 65 | 子协议类型和数据的签名 |
| 子协议类型 | 1 | 嵌套协议类型 |
| 子协议数据 | 可变 | 子协议数据 |

协议数据被接收被编码成`discv5.ingressPacket`结构:

{% tabs %}
{% tab title="p2p/discv5/udp.go" %}
```go
type ingressPacket struct {
	remoteID   NodeID   //远端节点ID
	remoteAddr *net.UDPAddr //远端节点udp地址
	ev         nodeEvent 
	hash       []byte  //
	data       interface{} // 内层嵌套协议
	rawData    []byte
}
```
{% endtab %}
{% endtabs %}

对于不同请求，`ingressPacket.data`承载具体协议数据:

{% tabs %}
{% tab title="p2p/discv5/net.go" %}
```go
//子协议类型
const (
  pingPacket = iota + 1
	pongPacket
	findnodePacket
	neighborsPacket
	findnodeHashPacket
	topicRegisterPacket
	topicQueryPacket
	topicNodesPacket
}
```
{% endtab %}

{% tab title="ping" %}
```go
//ping请求协议
ping struct {
		Version    uint
		From, To   rpcEndpoint
		Expiration uint64

		// v5
		Topics []Topic

		// Ignore additional fields (for forward compatibility).
		Rest []rlp.RawValue `rlp:"tail"`
}
```
{% endtab %}

{% tab title="pong" %}
```go
pong struct {
		To rpcEndpoint

		ReplyTok   []byte
		Expiration uint64

		// v5
		TopicHash    common.Hash
		TicketSerial uint32
		WaitPeriods  []uint32

		Rest []rlp.RawValue `rlp:"tail"`
	}
```
{% endtab %}

{% tab title="findnode" %}
```go
findnode struct {
		Target     NodeID
		Expiration uint64
		
		Rest []rlp.RawValue `rlp:"tail"`
}
```
{% endtab %}

{% tab title="findnodeHash" %}
```go
findnodeHash struct {
		Target     common.Hash
		Expiration uint64
		
		Rest []rlp.RawValue `rlp:"tail"`
}
```
{% endtab %}

{% tab title="neighbors" %}
```go
neighbors struct {
		Nodes      []rpcNode
		Expiration uint64
		// Ignore additional fields (for forward compatibility).
		Rest []rlp.RawValue `rlp:"tail"`
}
```
{% endtab %}

{% tab title="topicRegister" %}
```go
topicRegister struct {
		Topics []Topic
		Idx    uint
		Pong   []byte
}
```
{% endtab %}

{% tab title="topicQuery" %}
```go
topicQuery struct {
		Topic      Topic
		Expiration uint64
}
```
{% endtab %}

{% tab title="topicNodes" %}
```go
topicNodes struct {
		Echo  common.Hash
		Nodes []rpcNode
}
```
{% endtab %}
{% endtabs %}

### 编解码

数据在发送到网络时需要进行编码:

{% tabs %}
{% tab title="p2p/discv5/udp.go" %}
```go
// zeroed padding space for encodePacket.
versionPrefix     = []byte("temporary discovery v5")
versionPrefixSize = len(versionPrefix)
sigSize           = 520 / 8  //65字节
headSize = versionPrefixSize + sigSize
var headSpace = make([]byte, headSize)

func encodePacket(priv *ecdsa.PrivateKey, ptype byte, req interface{}) (p, hash []byte, err error) {
	b := new(bytes.Buffer)
	b.Write(headSpace)  //协议头
	b.WriteByte(ptype)
	if err := rlp.Encode(b, req); err != nil {
		log.Error(fmt.Sprint("error encoding packet:", err))
		return nil, nil, err
	}
	packet := b.Bytes()
	sig, err := crypto.Sign(crypto.Keccak256(packet[headSize:]), priv)
	if err != nil {
		log.Error(fmt.Sprint("could not sign packet:", err))
		return nil, nil, err
	}
	copy(packet, versionPrefix)
	copy(packet[versionPrefixSize:], sig)
	hash = crypto.Keccak256(packet[versionPrefixSize:])
	return packet, hash, nil
}
```
{% endtab %}
{% endtabs %}

从网络上接收数据需要进行解码并转化成目标数据结构:

{% tabs %}
{% tab title="p2p/discv5/udp.go" %}
```go
func decodePacket(buffer []byte, pkt *ingressPacket) error {
	if len(buffer) < headSize+1 {
		return errPacketTooSmall
	}
	buf := make([]byte, len(buffer))
	copy(buf, buffer)
	prefix, sig, sigdata := buf[:versionPrefixSize], buf[versionPrefixSize:headSize], buf[headSize:]
	if !bytes.Equal(prefix, versionPrefix) {
		return errBadPrefix
	}
	fromID, err := recoverNodeID(crypto.Keccak256(buf[headSize:]), sig)
	if err != nil {
		return err
	}
	pkt.rawData = buf
	pkt.hash = crypto.Keccak256(buf[versionPrefixSize:])
	pkt.remoteID = fromID
	switch pkt.ev = nodeEvent(sigdata[0]); pkt.ev {
	case pingPacket:
		pkt.data = new(ping)
	case pongPacket:
		pkt.data = new(pong)
	case findnodePacket:
		pkt.data = new(findnode)
	case neighborsPacket:
		pkt.data = new(neighbors)
	case findnodeHashPacket:
		pkt.data = new(findnodeHash)
	case topicRegisterPacket:
		pkt.data = new(topicRegister)
	case topicQueryPacket:
		pkt.data = new(topicQuery)
	case topicNodesPacket:
		pkt.data = new(topicNodes)
	default:
		return fmt.Errorf("unknown packet type: %d", sigdata[0])
	}
	s := rlp.NewStream(bytes.NewReader(sigdata[1:]), 0)
	err = s.Decode(pkt.data)
	return err
}
```
{% endtab %}
{% endtabs %}

### 传输层

UDP连接创建完成后需要从连接上接收和发送数据包，ethereum抽象`udp`结构体完成这一工作:

{% tabs %}
{% tab title="p2p/discv5/udp.go" %}
```go
type udp struct {
	conn        conn
	priv        *ecdsa.PrivateKey
	ourEndpoint rpcEndpoint
	nat         nat.Interface
	net         *Network
}
func (t *udp) Close() //关闭连接
//向远端发送数据
func (t *udp) send(remote *Node, ptype nodeEvent, data interface{}) (hash []byte)
//向远端发送ping包
func (t *udp) sendPing(remote *Node, toaddr *net.UDPAddr, topics []Topic) (hash []byte)
//向远端发送查询ID请求
func (t *udp) sendFindnode(remote *Node, target NodeID)
func (t *udp) sendNeighbours(remote *Node, results []*Node)
func (t *udp) sendFindnodeHash(remote *Node, target common.Hash)
func (t *udp) sendTopicRegister(remote *Node, topics []Topic, idx int, pong []byte)
func (t *udp) sendTopicNodes(remote *Node, queryHash common.Hash, nodes []*Node)
func (t *udp) sendPacket(toid NodeID, toaddr *net.UDPAddr, ptype byte, req interface{}) (hash []byte, err error)
func (t *udp) readLoop()
func (t *udp) handlePacket(from *net.UDPAddr, buf []byte) error
```
{% endtab %}
{% endtabs %}

### 处理数据

创建UDP连接完成后会启动一个goroutine从网络上接收数据包:

{% tabs %}
{% tab title="p2p/discv5/udp.go" %}
```go
func ListenUDP(priv *ecdsa.PrivateKey, conn conn, nodeDBPath string, netrestrict *netutil.Netlist) (*Network, error) {
	//省略代码
	go transport.readLoop()
	return net, nil
}
//读取网络数据循环
func (t *udp) readLoop() {
	defer t.conn.Close()

	buf := make([]byte, 1280)  //包最大是1280
	for {
		nbytes, from, err := t.conn.ReadFromUDP(buf)
		if netutil.IsTemporaryError(err) {
			//临时错误忽略
			continue
		} else if err != nil {
			//网络连接问题，退出
			log.Debug(fmt.Sprintf("Read error: %v", err))
			return
		}
		t.handlePacket(from, buf[:nbytes]) //处理数据包
	}
}
//处理udp数据包
func (t *udp) handlePacket(from *net.UDPAddr, buf []byte) error {
	pkt := ingressPacket{remoteAddr: from}
	if err := decodePacket(buf, &pkt); err != nil {
		return err
	}
	t.net.reqReadPacket(pkt)  //将数据包委托给上层处理
	return nil
}
```
{% endtab %}
{% endtabs %}

