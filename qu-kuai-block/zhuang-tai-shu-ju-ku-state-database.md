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

## 状态数据库

状态数据库是以帐户地址作为键的KV数据库。

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
type StateDB struct {
	db   Database  //存储mpt树, 合约代码
	trie Trie //世界状态对应的mpt树

   //stateObject内存缓存
	stateObjects      map[common.Address]*stateObject
	//发生过修改的stateObject
	stateObjectsDirty map[common.Address]struct{}

	dbErr error  //数据库发生的错误

	refund uint64

	thash, bhash common.Hash //当前evm正在处理的Block, Transaction哈希值
	txIndex      int   //当前正在处理的事务索引
	logs         map[common.Hash][]*types.Log //合约日志记录
	logSize      uint //合约日志条数

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
	suicided  bool  //帐户被销毁标记
	deleted   bool //是否已经删除
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
//返回帐户地址
func (s *stateObject) Address() common.Address
//设置帐户交易计数器
func (s *stateObject) SetNonce(nonce uint64)
//返回帐户交易计数
func (s *stateObject) Nonce() uint64 

//获取帐户数据中指定key的值，该值可能是还未提交的值，也是已经提交到storage trie的值
func (s *stateObject) GetState(db Database, key common.Hash) common.Hash 
//设置新的帐户数据
func (s *stateObject) SetState(db Database, key, value common.Hash)
//从帐户storage trie中获取已经提交的值.
func (s *stateObject) GetCommittedState(db Database, key common.Hash) common.Hash
//提交storage trie树
func (s *stateObject) CommitTrie(db Database) error

//返回帐户余额
func (s *stateObject) Balance() *big.Int
//增加帐户余额
func (s *stateObject) AddBalance(amount *big.Int) 
//减少帐户余额
func (s *stateObject) SubBalance(amount *big.Int)
//修改帐户余额
func (s *stateObject) SetBalance(amount *big.Int)

//返回合约代码
func (s *stateObject) Code(db Database) []byte
//设置合约代码
func (s *stateObject) SetCode(codeHash common.Hash, code []byte)
//返回合约哈希值
func (s *stateObject) CodeHash() []byte
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 方法

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
//获取数据库操作最后一次错误
func (self *StateDB) Error() error
//重置StateDB对象
func (self *StateDB) Reset(root common.Hash) error
//添加一条合约日志
func (self *StateDB) AddLog(log *types.Log)
//根据交易hash获取合约日志序列
func (self *StateDB) GetLogs(hash common.Hash) []*types.Log
//获取所有的合约日志
func (self *StateDB) Logs() []*types.Log

func (self *StateDB) AddPreimage(hash common.Hash, preimage []byte)
func (self *StateDB) Preimages() map[common.Hash][]byte

func (self *StateDB) AddRefund(gas uint64)
func (self *StateDB) SubRefund(gas uint64)
func (self *StateDB) GetRefund() uint64 

func (self *StateDB) Exist(addr common.Address) bool
func (self *StateDB) Empty(addr common.Address)

func (self *StateDB) GetBalance(addr common.Address) *big.Int
func (self *StateDB) GetNonce(addr common.Address) uint64
//增加帐户余额
func (self *StateDB) AddBalance(addr common.Address, amount *big.Int)
//减少帐户余额
func (self *StateDB) SubBalance(addr common.Address, amount *big.Int)
//设置帐户余额
func (self *StateDB) SetBalance(addr common.Address, amount *big.Int)
//设置帐户交易计数
func (self *StateDB) SetNonce(addr common.Address, nonce uint64)

func (self *StateDB) TxIndex() int
func (self *StateDB) BlockHash() common.Hash

//设置帐户合约代码
func (self *StateDB) SetCode(addr common.Address, code []byte)
func (self *StateDB) GetCode(addr common.Address) []byte
func (self *StateDB) GetCodeSize(addr common.Address) int
func (self *StateDB) GetCodeHash(addr common.Address) common.Hash

func (self *StateDB) GetProof(a common.Address) ([][]byte, error)
func (self *StateDB) GetStorageProof(a common.Address, key common.Hash) ([][]byte, error)

//获取帐户数据中某个key的值，有可能该值还未提交
func (self *StateDB) GetState(addr common.Address, hash common.Hash) common.Hash
//获取帐户数据中某个key已经提交的值
func (self *StateDB) GetCommittedState(addr common.Address, hash common.Hash) common.Hash
//设置帐户数据新的状态
func (self *StateDB) SetState(addr common.Address, key, value common.Hash)

//返回StateDB的后端存储
func (self *StateDB) Database() Database
//返回帐户数据的Trie树，并提交内存状态修改到Trie树
func (self *StateDB) StorageTrie(addr common.Address) Trie

//帐户是否被主动销毁
func (self *StateDB) HasSuicided(addr common.Address) bool

//销毁帐户
func (self *StateDB) Suicide(addr common.Address)

//获取或新建帐户stateObject
func (self *StateDB) GetOrNewStateObject(addr common.Address) *stateObject
//新建帐户
func (self *StateDB) CreateAccount(addr common.Address)
//遍历帐户数据所有的状态
func (db *StateDB) ForEachStorage(addr common.Address, cb func(key, value common.Hash) bool) error
//深拷贝StateDB
func (self *StateDB) Copy() *StateDB 

//StateDB快照
func (self *StateDB) Snapshot() int
//回滚到指定的快照
func (self *StateDB) RevertToSnapshot(revid int)

//更新StateDB当前的block, tx, tx index
func (self *StateDB) Prepare(thash, bhash common.Hash, ti int)

//触发Finalise操作，同时清理snapshot和refund，为下一次交易处理做准备
func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash
//提交StateDB到持久化存储
//1. 持久化存储合约代码
//2. 提交帐户状态trie树到持久化存储
//3. 提交世界状态trie树到持久化存储
func (s *StateDB) Commit(deleteEmptyObjects bool) (root common.Hash, err error)
//将对帐户和帐户状态的修改提交到对应的trie树，但还没有持久存储
func (s *StateDB) Finalise(deleteEmptyObjects bool)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 日志\(journal\)

### 实现

状态数据库所有的状态修改操作都会通过日志进行记录.

{% code-tabs %}
{% code-tabs-item title="core/state/journal.go" %}
```go
type journal struct {
	entries []journalEntry         // Current changes tracked by the journal
	dirties map[common.Address]int // Dirty accounts and the number of changes
}

type journalEntry interface {
	// revert undoes the changes introduced by this journal entry.
	revert(*StateDB)
	// dirtied returns the Ethereum address modified by this journal entry.
	dirtied() *common.Address
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

journalEntry是一个接口，根据不同类型的状态修改，以太坊提供了不同的实现.

{% code-tabs %}
{% code-tabs-item title="core/state/journal.go" %}
```go
//创建新的stateObject
createObjectChange struct {
		account *common.Address
}
//重置stateObject
resetObjectChange struct {
		prev *stateObject
}
//销毁stateObject
suicideChange struct {
		account     *common.Address
		prev        bool // whether account had already suicided
		prevbalance *big.Int
}

//变更余额
balanceChange struct {
		account *common.Address
		prev    *big.Int
}
//变更交易计数
nonceChange struct {
		account *common.Address
		prev    uint64
}
//帐户数据状态变更
storageChange struct {
		account       *common.Address
		key, prevalue common.Hash
}
//帐户合约代码变更
codeChange struct {
		account            *common.Address
		prevcode, prevhash []byte
}

//退款变更
refundChange struct {
		prev uint64
}
//添加合约日志
addLogChange struct {
		txhash common.Hash
}
//添加preimage
addPreimageChange struct {
		hash common.Hash
}

//帐户第一次增加余额
touchChange struct {
		account   *common.Address
		prev      bool
		prevDirty bool
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 方法

{% code-tabs %}
{% code-tabs-item title="core/state/journal.go" %}
```go
//新建journal
func newJournal() *journal {
	return &journal{
		dirties: make(map[common.Address]int),
	}
}

//追加一条journal条目
func (j *journal) append(entry journalEntry)
//回滚到指定的snapshot id
func (j *journal) revert(statedb *StateDB, snapshot int)
//指定地址的修改次数加1
func (j *journal) dirty(addr common.Address)
//journal条目数
func (j *journal) length() int
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 回滚

状态数据库可以回滚到指定的snapshot id, 回滚的方法如下:

```go
//参数snapshot是journal的一个索引，以类似栈的方式回滚历史修改
func (j *journal) revert(statedb *StateDB, snapshot int) {
	for i := len(j.entries) - 1; i >= snapshot; i-- {
		// Undo the changes made by the operation
		j.entries[i].revert(statedb)

		// Drop any dirty tracking induced by the change
		if addr := j.entries[i].dirtied(); addr != nil {
			if j.dirties[*addr]--; j.dirties[*addr] == 0 {
				delete(j.dirties, *addr)
			}
		}
	}
	j.entries = j.entries[:snapshot]
}
```

## 快照

StateDB支持快照，从而在运行时回滚历史修改,  StateDB定义中支持快照的字段包括:

```go
type StateDB struct {
    //...
    journal        *journal   //状态修改journal
    validRevisions []revision //有效修改集
	nextRevisionId int  //下一个快照id
	//....
}
```

StateDB快照的实现基于journal. StateDB.validRevisions保存了历史快照，revision结构的定义如下:

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
type revision struct {
	id           int   //快照id
	journalIndex int  //journal中entry的索引，journal中索引大于或等于journalIndex的修改包含在当前快照中
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 创建快照

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
func (self *StateDB) Snapshot() int {
	id := self.nextRevisionId
	self.nextRevisionId++
	self.validRevisions = append(self.validRevisions, revision{id, self.journal.length()})
	return id
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 恢复快照

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
func (self *StateDB) RevertToSnapshot(revid int) {
	// Find the snapshot in the stack of valid snapshots.
	idx := sort.Search(len(self.validRevisions), func(i int) bool {
		return self.validRevisions[i].id >= revid
	})
	if idx == len(self.validRevisions) || self.validRevisions[idx].id != revid {
		panic(fmt.Errorf("revision id %v cannot be reverted", revid))
	}
	snapshot := self.validRevisions[idx].journalIndex

	// Replay the journal to undo changes and remove invalidated snapshots
	self.journal.revert(self, snapshot)
	self.validRevisions = self.validRevisions[:idx]
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

清理快照

{% code-tabs %}
{% code-tabs-item title="core/state/statedb.go" %}
```go
func (s *StateDB) clearJournalAndRefund() {
	s.journal = newJournal()
	s.validRevisions = s.validRevisions[:0]
	s.refund = 0
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 存储结构

StateDB是世界状态数据库在运行时的实例，状态数据存放在一个持久化的数据库中，并通过Trie树索引访问

```go
type StateDB struct {
	db   Database  //持久化数据库
	trie Trie //内存Trie索引树
	//....
}
```

### 接口`state.Database`

`StateDB.db`是接口`state.Database`的一个实例。接口`state.Database`用于构造世界状态内存Trie索引树和用户状态内存Trie索引树，通过接口`state.Database`也可以获取帐户下合约代码。最后接口state.Database可以获得后端持久化存储。

{% code-tabs %}
{% code-tabs-item title="core/state/database.go" %}
```go
type Database interface {
	// OpenTrie opens the main account trie.
	OpenTrie(root common.Hash) (Trie, error)
	// OpenStorageTrie opens the storage trie of an account.
	OpenStorageTrie(addrHash, root common.Hash) (Trie, error)
	// CopyTrie returns an independent copy of the given trie.
	CopyTrie(Trie) Trie

	// ContractCode retrieves a particular contract's code.
	ContractCode(addrHash, codeHash common.Hash) ([]byte, error)
	// ContractCodeSize retrieves a particular contracts code's size.
	ContractCodeSize(addrHash, codeHash common.Hash) (int, error)

	// TrieDB retrieves the low level trie database used for data storage.
	TrieDB() *trie.Database
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

接口`state.Database`在以太坊中只有一种实现:

{% code-tabs %}
{% code-tabs-item title="core/state/database.go" %}
```go
type cachingDB struct {
	db            *trie.Database
	codeSizeCache *lru.Cache  //用于缓存contract code，默认大小为100000
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 类型`trie.Database`

类型trie.Database是一个数据库中间层，它用于聚合上层的写数据库请求，然后周期性的将请求批量的写入到持久化存储中。

{% code-tabs %}
{% code-tabs-item title="trie/database.go" %}
```go
type Database struct {
	diskdb ethdb.KeyValueStore // 持久化kv存储

	cleans  *bigcache.BigCache          //缓存
	dirties map[common.Hash]*cachedNode // Data and references relationships of dirty nodes
	oldest  common.Hash                 // Oldest tracked node, flush-list head
	newest  common.Hash                 // Newest tracked node, flush-list tail

	preimages map[common.Hash][]byte // Preimages of nodes from the secure trie
	seckeybuf [secureKeyLength]byte  // Ephemeral buffer for calculating preimage keys

	gctime  time.Duration      // Time spent on garbage collection since last commit
	gcnodes uint64             // Nodes garbage collected since last commit
	gcsize  common.StorageSize // Data storage garbage collected since last commit

	flushtime  time.Duration      // Time spent on data flushing since last commit
	flushnodes uint64             // Nodes flushed since last commit
	flushsize  common.StorageSize // Data storage flushed since last commit

	dirtiesSize   common.StorageSize // Storage size of the dirty node cache (exc. metadata)
	childrenSize  common.StorageSize // Storage size of the external children tracking
	preimagesSize common.StorageSize // Storage size of the preimages cache

	lock sync.RWMutex
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

