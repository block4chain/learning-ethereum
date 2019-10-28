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

### txList

`txList`用来存储一个帐户的交易列表

{% code-tabs %}
{% code-tabs-item title="core/tx\_list.go" %}
```go
type txList struct {
	strict bool         // nonce是否严格连续递增
	txs    *txSortedMap // 所有交易按nonce排序，并支持根据nonce索引查找

	costcap *big.Int // 
	gascap  uint64   // 
}
//是否已经存在一个nonce值tx相等的交易
func (l *txList) Overlaps(tx *types.Transaction) bool
//添加一个交易，如果交易nonce冲突，则新交易必须满足一定的条件才能替换
func (l *txList) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction)
//返回所有nonce小于threshold的交易
func (l *txList) Forward(threshold uint64) types.Transactions
//返回所有消耗/gas上耗超过costLimit或gasLimit的交易
func (l *txList) Filter(costLimit *big.Int, gasLimit uint64) (types.Transactions, types.Transactions)
//将列表大小限制到threshold
func (l *txList) Cap(threshold int) types.Transactions
//移除指定的交易
func (l *txList) Remove(tx *types.Transaction) (bool, types.Transactions)
//返回nonce<=start的交易或者nonce>=start的所有nonce连续的交易
func (l *txList) Ready(start uint64) types.Transactions
//返回交易列表长度
func (l *txList) Len() int 
//交易列表是否为空
func (l *txList) Empty() bool
//返回交易列表
func (l *txList) Flatten() types.Transactions
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
	
	PriceBump  uint64  //交易碰撞时，替换交易时新交易Gas单价的增长率

	//交易池交易数限制
	AccountSlots uint64
	GlobalSlots  uint64   //所有帐户可执行队列容量总限制
	AccountQueue uint64
	GlobalQueue  uint64  //所有帐户等待队列容量总限制
	
	//交易滞留时间限制
	Lifetime time.Duration
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 交易队列

### 全局队列

交易维护两个全局队列，用于记录跟踪所有交易:

* 查询队列: 利用哈希表记录所有交易，方便查询
* 竞价队列: 按Gas单价由低到高排列所有交易\(使用小堆实现\)

```go
type TxPool struct {
	//省略代码
	all     *txLookup               //查询队列
	priced  *txPricedList    // 竞价队列
	//省略代码
}
```

### 帐户队列

交易池为帐户维护着两类交易队列，用户记录跟踪帐户提交的交易:

* 等待队列
* 可执行队列

```go
type TxPool struct {
	//省略代码
	pending map[common.Address]*txList   // 可执行队列
	queue   map[common.Address]*txList   // 等待队列
	//省略代码
}
```

交易池队列会为每个交易帐户维护一个交易列表\(`txList`\)。



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

### 交易验证

交易池在收集交易前会对交易进行一系列验证，保证收集的交易满足要求

1. 交易大小不超32KB
2. 转帐金额不是负数
3. 交易的Gas上限不超过当前块的Gas上限
4. 交易的签名有效
5. 非本地交易的Gas单价满足最低要求
6. 交易nonce值合法\(大于当前记录的最大nonce\)
7. 交易消息的金额不超过帐户的余额
8. 创建合约的交易规则校验

```go
func (pool *TxPool) add(tx *types.Transaction, local bool) (replaced bool, err error) {
    //省略代码
    if err := pool.validateTx(tx, local); err != nil {
		log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
		invalidTxMeter.Mark(1)
		return false, err
	}
	//省略代码
}
```

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
	// 交易大小
	if tx.Size() > 32*1024 {
		return ErrOversizedData
	}
	// 转帐金额
	if tx.Value().Sign() < 0 {
		return ErrNegativeValue
	}
	// gas上限
	if pool.currentMaxGas < tx.Gas() {
		return ErrGasLimit
	}
	// 交易签名
	from, err := types.Sender(pool.signer, tx)
	if err != nil {
		return ErrInvalidSender
	}
	// 非本地交易的gas单价限制
	local = local || pool.locals.contains(from) // account may be local even if the transaction arrived from the network
	if !local && pool.gasPrice.Cmp(tx.GasPrice()) > 0 {
		return ErrUnderpriced
	}
	//交易nonce
	if pool.currentState.GetNonce(from) > tx.Nonce() {
		return ErrNonceTooLow
	}
	//交易最大消耗的balance不超过帐户的余额
	if pool.currentState.GetBalance(from).Cmp(tx.Cost()) < 0 {
		return ErrInsufficientFunds
	}
	// 创建合约交易gas限制
	intrGas, err := IntrinsicGas(tx.Data(), tx.To() == nil, true, pool.istanbul)
	if err != nil {
		return err
	}
	if tx.Gas() < intrGas {
		return ErrIntrinsicGas
	}
	return nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 交易碰撞

当同一个帐户提交了两个不同\(交易hash不同\)的交易，但交易Nonce相同，此时会发生交易碰撞。

当Gas单价满足一定条件时以太坊才会用新交易替换旧交易:

$$
{newGasPrice \over oldGasPrice} > 1 + {priceBump\over100}
$$

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
func (pool *TxPool) add(tx *types.Transaction, local bool) (replaced bool, err error) {
    //省略部分代码
    if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
		// Nonce already pending, check if required price bump is met
		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted {
			pendingDiscardMeter.Mark(1)
			return false, ErrReplaceUnderpriced
		}
		// New transaction is better, replace old one
		if old != nil {
			pool.all.Remove(old.Hash())
			pool.priced.Removed(1)
			pendingReplaceMeter.Mark(1)
		}
		pool.all.Add(tx)
		pool.priced.Put(tx)
		pool.journalTx(from, tx)
		pool.queueTxEvent(tx)
		log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())
		return old != nil, nil
	}
    //省略部分代码
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="core/tx\_list.go" %}
```go
//priceBump由节点自由设置
func (l *txList) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
	// If there's an older better transaction, abort
	old := l.txs.Get(tx.Nonce())
	if old != nil {
		threshold := new(big.Int).Div(new(big.Int).Mul(old.GasPrice(), big.NewInt(100+int64(priceBump))), big.NewInt(100))
		if old.GasPrice().Cmp(tx.GasPrice()) >= 0 || threshold.Cmp(tx.GasPrice()) > 0 {
			return false, nil
		}
	}

	l.txs.Put(tx)
	if cost := tx.Cost(); l.costcap.Cmp(cost) < 0 {
		l.costcap = cost
	}
	if gas := tx.Gas(); l.gascap < gas {
		l.gascap = gas
	}
	return true, old
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

交易处于等待队列和待执行队列情况一样。

### 交易拥堵

交易池的队列有容量限制，容量大小定义在交易池配置中:

```go
type TxPoolConfig struct {
	//省略代码
	GlobalSlots  uint64   //所有帐户可执行队列容量总限制
	GlobalQueue  uint64  //所有帐户等待队列容量总限制
	//省略代码
}
```

当交易池队列出现拥堵时，节点会按照一定的规则丢弃部分交易:

```go
func (pool *TxPool) add(tx *types.Transaction, local bool) (replaced bool, err error) {
	//省略代码
    //交易池出现拥堵
	if uint64(pool.all.Count()) >= pool.config.GlobalSlots+pool.config.GlobalQueue {
		//交易的gas单价小于所有交易的最小值，则丢弃
		if !local && pool.priced.Underpriced(tx, pool.locals) {
			return false, ErrUnderpriced
		}
		// 按gas单价从低到高丢弃远程交易
		drop := pool.priced.Discard(pool.all.Count()-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
		for _, tx := range drop {
			pool.removeTx(tx.Hash(), false) 
		}
	}
	//省略代码
}
```

### 交易排队

当交易通过验证，并且解决碰撞、拥堵等问题，节点会把交易放入等待队列:

{% code-tabs %}
{% code-tabs-item title="core/tx\_pool.go" %}
```go
func (pool *TxPool) enqueueTx(hash common.Hash, tx *types.Transaction) (bool, error) {
	// Try to insert the transaction into the future queue
	from, _ := types.Sender(pool.signer, tx) // already validated
	if pool.queue[from] == nil {
		pool.queue[from] = newTxList(false)  //一个新的帐户提交了交易
	}
	inserted, old := pool.queue[from].Add(tx, pool.config.PriceBump)  //将交易放入帐户等待队列
	if !inserted {
		//交易碰撞，并且不满足替换规则
		return false, ErrReplaceUnderpriced
	}
	if old != nil {
		//旧交易被替换，从全局队列中移除
		pool.all.Remove(old.Hash())
		pool.priced.Removed(1)
	} else {
		queuedCounter.Inc(1)
	}
	if pool.all.Get(hash) == nil {
		//将交易放入全局队列
		pool.all.Add(tx)
		pool.priced.Put(tx)
	}
	return old != nil, nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}



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



