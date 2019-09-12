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

