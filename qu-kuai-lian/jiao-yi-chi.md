---
description: 交易池收集了以太坊节点本地提交或者网络接收的其它节点提交的所有交易。
---

# 交易池

## 定义

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
type TxPool struct {
	config      TxPoolConfig
	chainconfig *params.ChainConfig
	chain       blockChain
	gasPrice    *big.Int  //进入这个交易池的Gas单价下限
	txFeed      event.Feed
	scope       event.SubscriptionScope
	signer      types.Signer
	mu          sync.RWMutex

	istanbul bool // Fork indicator whether we are in the istanbul stage.

	currentState  *state.StateDB // Current state in the blockchain head
	pendingNonces *txNoncer      // Pending state tracking virtual nonces
	currentMaxGas uint64         // Current gas limit for transaction caps

	locals  *accountSet // Set of local transaction to exempt from eviction rules
	journal *txJournal  // Journal of local transaction to back up to disk

	pending map[common.Address]*txList   // All currently processable transactions
	queue   map[common.Address]*txList   // Queued but non-processable transactions
	beats   map[common.Address]time.Time // Last heartbeat from each known account
	all     *txLookup                    // All transactions to allow lookups
	priced  *txPricedList                // All transactions sorted by price

	chainHeadCh     chan ChainHeadEvent
	chainHeadSub    event.Subscription
	reqResetCh      chan *txpoolResetRequest
	reqPromoteCh    chan *accountSet
	queueTxEventCh  chan *types.Transaction
	reorgDoneCh     chan chan struct{}
	reorgShutdownCh chan struct{}  // requests shutdown of scheduleReorgLoop
	wg              sync.WaitGroup // tracks loop, scheduleReorgLoop
}
//关闭交易池
func (pool *TxPool) Stop()
//订阅新交易事件
func (pool *TxPool) SubscribeNewTxsEvent(ch chan<- NewTxsEvent) event.Subscription

//获取、设置交易的最低Gas单价
func (pool *TxPool) GasPrice() *big.Int
func (pool *TxPool) SetGasPrice(price *big.Int)

//获取帐户交易计数
func (pool *TxPool) Nonce(addr common.Address) uint64

func (pool *TxPool) Stats() (int, int)
func (pool *TxPool) Status(hashes []common.Hash) []TxStatus

func (pool *TxPool) Content() (map[common.Address]types.Transactions, map[common.Address]types.Transactions)
func (pool *TxPool) Pending() (map[common.Address]types.Transactions, error)
func (pool *TxPool) Locals() []common.Address

func (pool *TxPool) AddLocals(txs []*types.Transaction) []error
func (pool *TxPool) AddLocal(tx *types.Transaction) error
func (pool *TxPool) AddRemotes(txs []*types.Transaction) []error
func (pool *TxPool) AddRemotesSync(txs []*types.Transaction) []error
func (pool *TxPool) AddRemote(tx *types.Transaction) error
func (pool *TxPool) Get(hash common.Hash) *types.Transaction
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 辅助数据结构

### accountSet

`accountSet`记录一系列帐户构成的集合

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
type accountSet struct {
	accounts map[common.Address]struct{}  //帐户地址集合
	signer   types.Signer  //用于从交易中提取发送者地址
	cache    *[]common.Address
}
//判断是否包含一个地址
func (as *accountSet) contains(addr common.Address) bool
//判断交易的发送者是否包含在这个集合中
func (as *accountSet) containsTx(tx *types.Transaction) bool
//添加一个地址
func (as *accountSet) add(addr common.Address)
//将交易的发送者帐户添加到集合中
func (as *accountSet) addTx(tx *types.Transaction)
//返回所有帐户数组
func (as *accountSet) flatten() []common.Address
//全并两个集合
func (as *accountSet) merge(other *accountSet)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### txLookup

`txLookup`用于跟踪交易，方便查询

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
type txLookup struct {
	all  map[common.Hash]*types.Transaction
	lock sync.RWMutex
}
func (t *txLookup) Get(hash common.Hash) *types.Transaction
func (t *txLookup) Count() int
func (t *txLookup) Add(tx *types.Transaction)
func (t *txLookup) Remove(hash common.Hash)
func (t *txLookup) Range(f func(hash common.Hash, tx *types.Transaction) bool)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### priceHeap

`priceHeap`实现heap.Interface，是一个小堆，堆顶是Gas单价最低的交易。

{% code-tabs %}
{% code-tabs-item title="core/tx\_list.go" %}
```go
type priceHeap []*types.Transaction

//实现heap.Interface
func (h priceHeap) Len() int
func (h priceHeap) Swap(i, j int)
func (h priceHeap) Less(i, j int) bool
func (h *priceHeap) Push(x interface{})
func (h *priceHeap) Pop() interface{}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### nonceHeap

`nonceHead`实现heap.Interface, 是一个小堆, 堆顶最小的nonce值

{% code-tabs %}
{% code-tabs-item title="core/tx\_list.go" %}
```go
type nonceHeap []uint64

//实现heap.Interface
func (h nonceHeap) Len() int
func (h nonceHeap) Less(i, j int) bool
func (h nonceHeap) Swap(i, j int)
func (h *nonceHeap) Push(x interface{})
func (h *nonceHeap) Pop() interface{}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### txSortedMap

`txSortedMap`是一个按nonce值排序的哈希表, 键是交易的nonce, 值是交易。

{% code-tabs %}
{% code-tabs-item title="core/tx\_list.go" %}
```go
type txSortedMap struct {
	items map[uint64]*types.Transaction // 哈希表
	index *nonceHeap                    // nonce值构成的小堆
	cache types.Transactions            // 已排序缓存
}
//根据nonce值获取交易
func (m *txSortedMap) Get(nonce uint64) *types.Transaction
//添加一个交易
func (m *txSortedMap) Put(tx *types.Transaction)
//移除nonce<threshold的交易
func (m *txSortedMap) Forward(threshold uint64) types.Transactions
//移除所有满足filter条件的交易
func (m *txSortedMap) Filter(filter func(*types.Transaction) bool) types.Transactions
//按nonce降序值删除交易，直到包含的交易数小于threshold
func (m *txSortedMap) Cap(threshold int) types.Transactions
//按nonce移除交易
func (m *txSortedMap) Remove(nonce uint64) bool
//返回一系列nonce连续递增的交易
func (m *txSortedMap) Ready(start uint64) types.Transactions
//返回包含的所有交易数
func (m *txSortedMap) Len() int
//返回nonce值递增的交易数组
func (m *txSortedMap) Flatten() types.Transactions
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### txNoncer

`txNoncer`用于获取帐户最新nonce值

{% code-tabs %}
{% code-tabs-item title="core/tx\_noncer.go" %}
```go
type txNoncer struct {
	fallback *state.StateDB  //状态数据库
	nonces   map[common.Address]uint64  //帐户nonce缓存
	lock     sync.Mutex
}
//获取帐户最新的nonce
func (txn *txNoncer) get(addr common.Address) uint64
//临时设置帐户最新nonce
func (txn *txNoncer) set(addr common.Address, nonce uint64)
//如果帐户nonce值小于新值，则设置新值
func (txn *txNoncer) setIfLower(addr common.Address, nonce uint64)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### txPricedList

`txPricedList`按Gas单价从低到高排列所有交易

{% code-tabs %}
{% code-tabs-item title="core/tx\_list.go" %}
```go
type txPricedList struct {
	all    *txLookup  // Pointer to the map of all transactions
	items  *priceHeap // Heap of prices of all the stored transactions
	stales int        // Number of stale price points to (re-heap trigger)
}

func (l *txPricedList) Put(tx *types.Transaction)
func (l *txPricedList) Removed(count int)
func (l *txPricedList) Cap(threshold *big.Int, local *accountSet) types.Transactions
func (l *txPricedList) Underpriced(tx *types.Transaction, local *accountSet) bool
func (l *txPricedList) Discard(count int, local *accountSet) types.Transactions
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 配置

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
type TxPoolConfig struct {
	//本地地址列表, 交易发起方地址属于这个列表的都被认为是本地提交的交易
	Locals    []common.Address
	//交易池是否接收本地提交的交易
	NoLocals  bool
	
	//交易池journal机制
	Journal   string 
	Rejournal time.Duration

	//接收的交易Gas单价下限
	PriceLimit uint64
	PriceBump  uint64

	//交易池交易数限制
	AccountSlots uint64
	GlobalSlots  uint64
	AccountQueue uint64
	GlobalQueue  uint64
	
	//交易滞留时间限制
	Lifetime time.Duration
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 收集交易

### 本地交易

以太坊客户端通过AP提交的交易作为_**本地交易**_被提交到交易池中

{% code-tabs %}
{% code-tabs-item title="eth/api\_backend.go" %}
```go
func (b *EthAPIBackend) SendTx(ctx context.Context, signedTx *types.Transaction) error {
	return b.eth.txPool.AddLocal(signedTx)
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 远端交易

从p2p网络监听到的交易作为远端交易提交到交易池中

{% code-tabs %}
{% code-tabs-item title="eth/handle.go" %}
```go
func (pm *ProtocolManager) handleMsg(p *peer) error {
      //省略一些代码
      case msg.Code == TxMsg:
		    // Transactions arrived, make sure we have a valid and fresh chain to handle them
			if atomic.LoadUint32(&pm.acceptTxs) == 0 {
			     break
			}
			// Transactions can be processed, parse all of them and deliver to the pool
			var txs []*types.Transaction
			if err := msg.Decode(&txs); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
			}
			for i, tx := range txs {
				// Validate and mark the remote transaction
				if tx == nil {
					return errResp(ErrDecode, "transaction %d is nil", i)
				}
				p.MarkTransaction(tx.Hash())
			}
			//提交远端交易
			pm.txpool.AddRemotes(txs)
		default:
			return errResp(ErrInvalidMsgCode, "%v", msg.Code)
	}
	return nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 核心逻辑



## 创建交易池

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain blockChain) *TxPool {
	config = (&config).sanitize()  //检查交易池配置参数，并用默认参数纠正非法值
	pool := &TxPool{
		config:          config,
		chainconfig:     chainconfig,
		chain:           chain,
		signer:          types.NewEIP155Signer(chainconfig.ChainID),
		pending:         make(map[common.Address]*txList),
		queue:           make(map[common.Address]*txList),
		beats:           make(map[common.Address]time.Time),
		all:             newTxLookup(),
		chainHeadCh:     make(chan ChainHeadEvent, chainHeadChanSize),
		reqResetCh:      make(chan *txpoolResetRequest),
		reqPromoteCh:    make(chan *accountSet),
		queueTxEventCh:  make(chan *types.Transaction),
		reorgDoneCh:     make(chan chan struct{}),
		reorgShutdownCh: make(chan struct{}),
		gasPrice:        new(big.Int).SetUint64(config.PriceLimit),
	}
	pool.locals = newAccountSet(pool.signer)  //加载本地帐户
	//添加配置中的本地帐户
	for _, addr := range config.Locals { 
		log.Info("Setting new local account", "address", addr)
		pool.locals.add(addr)
	}
	
	pool.priced = newTxPricedList(pool.all)
	pool.reset(nil, chain.CurrentBlock().Header())  //用当前最新块重置交易池

	pool.wg.Add(1)
	go pool.scheduleReorgLoop()

	// 加载持久化的事务
	if !config.NoLocals && config.Journal != "" {
		pool.journal = newTxJournal(config.Journal)

		if err := pool.journal.load(pool.AddLocals); err != nil {
			log.Warn("Failed to load transaction journal", "err", err)
		}
		if err := pool.journal.rotate(pool.local()); err != nil {
			log.Warn("Failed to rotate transaction journal", "err", err)
		}
	}

	// 订阅新区块事件
	pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)
	
	pool.wg.Add(1)
	go pool.loop()   //启动交易池

	return pool
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 交易生命周期



