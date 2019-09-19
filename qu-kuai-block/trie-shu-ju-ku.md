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
	diskdb ethdb.KeyValueStore // Persistent storage for matured trie nodes

	cleans  *bigcache.BigCache          // GC friendly memory cache of clean node RLPs
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



