---
description: 以太坊是一个互联网范围下的状态机，每次的交易提交都是一个状态迁移。状态数据库(State Database)维护着以太坊中的所有状态。
---

# 状态数据库

![](../.gitbook/assets/0_e35cmd76dt-wwwa3.jpeg)

## 世界状态

世界状态\(World State\)是所有帐户\(Account\)状态组成的集合，通过一个MPT树进行组织，树根\(root\)存储在块结构中的stateRoot域中。

{% code-tabs %}
{% code-tabs-item title="core/blockchain.go" %}
```go
		//获取世界状态数据库
		parent := it.previous()
		if parent == nil {
			parent = bc.GetHeader(block.ParentHash(), block.NumberU64()-1)
		}
		statedb, err := state.New(parent.Root, bc.stateCache) //参见core/state/statedb.go
		if err != nil {
			return it.index, events, coalescedLogs, err
		}
```
{% endcode-tabs-item %}

{% code-tabs-item title="core/state/statedb.go" %}
```go
func New(root common.Hash, db Database) (*StateDB, error) {
	tr, err := db.OpenTrie(root)  ////根据stateRoot创建MPT树
	if err != nil {
		return nil, err
	}
	return &StateDB{
		db:                db,
		trie:              tr,
		stateObjects:      make(map[common.Address]*stateObject),
		stateObjectsDirty: make(map[common.Address]struct{}),
		logs:              make(map[common.Hash][]*types.Log),
		preimages:         make(map[common.Hash][]byte),
		journal:           newJournal(),
	}, nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 状态数据库

状态数据库是以帐户地址作为键的KV数据库。

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
type StateDB struct {
	db   Database  //存储mpt树, 合约代码
	trie Trie //世界状态对应的mpt树

	stateObjects      map[common.Address]*stateObject
	stateObjectsDirty map[common.Address]struct{}

	dbErr error  //数据库发生的错误

	refund uint64

	thash, bhash common.Hash
	txIndex      int
	logs         map[common.Hash][]*types.Log
	logSize      uint

	preimages map[common.Hash][]byte

	//以下字段用于snapshot
	journal        *journal
	validRevisions []revision
	nextRevisionId int
	.....
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="core/state/state\_object.go" %}
```go
type stateObject struct {
	address  common.Address  //帐户地址
	addrHash common.Hash // 帐户地址对应的hash
	data     Account //帐户地址对应的帐户状态值
	db       *StateDB //与这个stateObject关联的statedb

	dbErr error //数据库错误

	trie Trie // 帐户管理的局部状态MPT数
	code Code // 合约帐户的代码

	originStorage Storage // Storage cache of original entries to dedup rewrites
	dirtyStorage  Storage // Storage entries that need to be flushed to disk
	fakeStorage   Storage // Fake storage which constructed by caller for debugging purpose.

	// Cache flags.
	// When an object is marked suicided it will be delete from the trie
	// during the "update" phase of the state transition.
	dirtyCode bool // true if the code was updated
	suicided  bool
	deleted   bool
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 帐户数据

StateDB中帐户详细数据用stateObject结构表示:

{% code-tabs %}
{% code-tabs-item title="core/state/state\_object.go" %}
```go
type stateObject struct {
	address  common.Address //帐户地址
	addrHash common.Hash // 帐户地址哈希
	data     Account //帐户状态
	db       *StateDB //该stateObject关联的StateDB
	
	dbErr error  //数据库错误

	// Write caches.
	trie Trie // 帐户的合约状态数据trie数据库
	code Code // 帐户的合约代码

	originStorage Storage // 内存临时存储，缓存从trie中已经读取的数据
	dirtyStorage  Storage //内存临时存储，存放暂未提交的帐户kv
	fakeStorage   Storage //内存临时存储，方便调试

    //数据状态标记
	dirtyCode bool // 合约代码是否被修改
	suicided  bool
	deleted   bool
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### Trie树

以太坊中帐户可以包含一些持久化数据，这些数据以键值对的形式存储是一个Trie树中, Account.Root存放着这棵树的根, 在内存中通过stateObject.trie可以访问这棵树.

{% code-tabs %}
{% code-tabs-item title="core/state/state\_object.go" %}
```go
func (s *stateObject) getTrie(db Database) Trie {
	if s.trie == nil {
		var err error
		s.trie, err = db.OpenStorageTrie(s.addrHash, s.data.Root)  //s.addrHash暂时不用，只直接通过s.data.Root打开trie树
		if err != nil {
			s.trie, _ = db.OpenStorageTrie(s.addrHash, common.Hash{})
			s.setError(fmt.Errorf("can't create storage trie: %v", err))
		}
	}
	return s.trie
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### 方法

{% code-tabs %}
{% code-tabs-item title="core/state/state\_object.go" %}
```go
//stateObject进行RLP编码，只编码stateObject.data字段.
func (s *stateObject) EncodeRLP(w io.Writer) error
//获取帐户数据中指定key的值，该值可能是还未提交的值，也是已经提交到storage trie的值
func (s *stateObject) GetState(db Database, key common.Hash) common.Hash 
//从帐户storage trie中获取已经提交的值.
func (s *stateObject) GetCommittedState(db Database, key common.Hash) common.Hash
//提交storage trie树
func (s *stateObject) CommitTrie(db Database) error
//增加帐户余额
func (s *stateObject) AddBalance(amount *big.Int) 
//减少帐户余额
func (s *stateObject) SubBalance(amount *big.Int)
//修改帐户余额
func (s *stateObject) SetBalance(amount *big.Int)
//返回帐户地址
func (s *stateObject) Address() common.Address
//返回合约代码
func (s *stateObject) Code(db Database) []byte
//设置合约代码
func (s *stateObject) SetCode(codeHash common.Hash, code []byte)
//设置帐户交易计数器
func (s *stateObject) SetNonce(nonce uint64)
//返回合约哈希值
func (s *stateObject) CodeHash() []byte
//返回帐户余额
func (s *stateObject) Balance() *big.Int
//返回帐户交易计数
func (s *stateObject) Nonce() uint64 
```
{% endcode-tabs-item %}
{% endcode-tabs %}

