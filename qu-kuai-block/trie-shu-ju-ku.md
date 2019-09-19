---
description: >-
  以太坊中的持久化数据最终都以Key-Value的形式存储在Trie数据库中。Trie数据库以MTP树作为数据索引，并将索引数据和值数据存放在一个KV存储引擎中(LevelDB)。
---

# Trie数据库

## 数据库

trie.Database是Trie数据库在运行时的实例类型。trie.Database会对所有的写操作在内存中进行聚合，并周期性的批量写入KV存储引擎。

{% code-tabs %}
{% code-tabs-item title="trie/database.go" %}
```go
type Database struct {
	diskdb ethdb.KeyValueStore // 后端kv存储引擎

	cleans  *bigcache.BigCache          // GC friendly memory cache of clean node RLPs
	dirties map[common.Hash]*cachedNode // 被修改过的节点缓存
	oldest  common.Hash                 // 最早插入的节点key
	newest  common.Hash                 // 最晚插入的节点key

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

### 数据缓存

`trie.Database`用于存储MPT树节点和二进制Blob数据，当前这些数据写入时, trie.Database用一些内置类型对这些数据进行封装

{% code-tabs %}
{% code-tabs-item title="trie/database.go" %}
```go
type rawNode []byte //封装纯二进制blob数据
type rawFullNode [17]node //封装MPT分支节点
//封装MPT扩展/叶子结点
type rawShortNode struct {
	Key []byte
	Val node
} 
```
{% endcode-tabs-item %}
{% endcode-tabs %}

所有写入缓存的节点数据用一个双向链表进行组织管理，链表节点定义为:

![&#x5185;&#x5B58;&#x7F13;&#x5B58;&#x53CC;&#x5411;&#x94FE;&#x8868;](../.gitbook/assets/trie_database_cache.png)

{% code-tabs %}
{% code-tabs-item title="trie/database.go" %}
```go
type cachedNode struct {
	node node   // Cached collapsed trie node, or raw rlp data 
	size uint16 // Byte size of the useful cached data

	parents  uint32                 // 该节点包含在其它MPT子树的数量
	children map[common.Hash]uint16 // 以该节点作为根的MPT子树的节点集合

	flushPrev common.Hash // 链表前向引用
	flushNext common.Hash // 链表后向引用
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 常用方法

{% code-tabs %}
{% code-tabs-item title="trie/database.go" %}
```go
//返回后端KV存储引擎
func (db *Database) DiskDB() ethdb.KeyValueReader
//向数据库插入一条二进制数据, 以参数hash作为key
func (db *Database) InsertBlob(hash common.Hash, blob []byte)
//向数据库插入一条记录, hash为key, blob为数据, node是数据对应的MPT树节点
func (db *Database) insert(hash common.Hash, blob []byte, node node)
//返回数据库缓存的MPT树节点
func (db *Database) node(hash common.Hash) node
//从数据库中获取指定key的数据
func (db *Database) Node(hash common.Hash) ([]byte, error)
//枚举所有缓存的key
func (db *Database) Nodes() []common.Hash
//添加一个从parent到child的引用
func (db *Database) Reference(child common.Hash, parent common.Hash)
//解除对一个key的引用
func (db *Database) Dereference(root common.Hash)
//解除从parent到child的引用
func (db *Database) dereference(child common.Hash, parent common.Hash)
//提交内存缓存到kv存储引擎，直到内存缓存占用小于limit
func (db *Database) Cap(limit common.StorageSize) error
//提交指定的key到kv存储引擎
func (db *Database) Commit(node common.Hash, report bool) error
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 数据操作

### 内存管理



