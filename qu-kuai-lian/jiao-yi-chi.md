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

func (pool *TxPool) Stop()
func (pool *TxPool) SubscribeNewTxsEvent(ch chan<- NewTxsEvent) event.Subscription

func (pool *TxPool) GasPrice() *big.Int
func (pool *TxPool) SetGasPrice(price *big.Int)

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

