---
description: 矿工执行以太坊的挖矿流程
---

# 矿工

## Miner结构

Miner结构定义了一个矿工，在创建以太坊实例时会创建一个Miner实例

{% tabs %}
{% tab title="eth/backend.go" %}
```go
func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
    //省略代码
    eth.miner = miner.New(eth, &config.Miner, chainConfig, eth.EventMux(), eth.engine, eth.isLocalBlock)
	eth.miner.SetExtra(makeExtraData(config.Miner.ExtraData))
	//省略代码
}
```
{% endtab %}

{% tab title="miner/miner.go" %}
```go
type Miner struct {
	mux      *event.TypeMux
	worker   *worker  //矿工工作线程
	coinbase common.Address  //矿工地址
	eth      Backend
	engine   consensus.Engine  //共识引擎
	exitCh   chan struct{}

	canStart    int32 // 允许启动开关
	shouldStart int32 // 启动开关
}
//启动矿工
func (self *Miner) Start(coinbase common.Address)
//停止矿工
func (self *Miner) Stop()
//关闭矿工
func (self *Miner) Close()
//是否开启挖矿
func (self *Miner) Mining() bool
//节点哈希率
func (self *Miner) HashRate() uint64
//设置块附加数据
func (self *Miner) SetExtra(extra []byte) error
//设置重新提交间隔
func (self *Miner) SetRecommitInterval(interval time.Duration)
//返回挂起的块，以及对应的statedb
func (self *Miner) Pending() (*types.Block, *state.StateDB)
//返回挂起的块
func (self *Miner) PendingBlock() *types.Block
//设置挖矿地址
func (self *Miner) SetEtherbase(addr common.Address)
```
{% endtab %}
{% endtabs %}

## worker结构

miner有一个worker实例，worker实例执行区块的打包、共识、打包等流程。

{% tabs %}
{% tab title="miner/worker.go" %}
```go
type worker struct {
	config      *Config
	chainConfig *params.ChainConfig
	engine      consensus.Engine
	eth         Backend
	chain       *core.BlockChain

	// Subscriptions
	mux          *event.TypeMux
	txsCh        chan core.NewTxsEvent
	txsSub       event.Subscription
	chainHeadCh  chan core.ChainHeadEvent
	chainHeadSub event.Subscription
	chainSideCh  chan core.ChainSideEvent
	chainSideSub event.Subscription

	// Channels
	newWorkCh          chan *newWorkReq
	taskCh             chan *task
	resultCh           chan *types.Block
	startCh            chan struct{}
	exitCh             chan struct{}
	resubmitIntervalCh chan time.Duration
	resubmitAdjustCh   chan *intervalAdjust

	current      *environment                 // An environment for current running cycle.
	localUncles  map[common.Hash]*types.Block // A set of side blocks generated locally as the possible uncle blocks.
	remoteUncles map[common.Hash]*types.Block // A set of side blocks as the possible uncle blocks.
	unconfirmed  *unconfirmedBlocks           // A set of locally mined blocks pending canonicalness confirmations.

	mu       sync.RWMutex // The lock used to protect the coinbase and extra fields
	coinbase common.Address
	extra    []byte

	pendingMu    sync.RWMutex
	pendingTasks map[common.Hash]*task

	snapshotMu    sync.RWMutex // The lock used to protect the block snapshot and state snapshot
	snapshotBlock *types.Block
	snapshotState *state.StateDB

	// atomic status counters
	running int32 // The indicator whether the consensus engine is running or not.
	newTxs  int32 // New arrival transaction count since last sealing work submitting.

	// External functions
	isLocalBlock func(block *types.Block) bool // Function used to determine whether the specified block is mined by local miner.

	// Test hooks
	newTaskHook  func(*task)                        // Method to call upon receiving a new sealing task.
	skipSealHook func(*task) bool                   // Method to decide whether skipping the sealing.
	fullTaskHook func()                             // Method to call before pushing the full sealing task.
	resubmitHook func(time.Duration, time.Duration) // Method to call upon updating resubmitting interval.
}
```
{% endtab %}
{% endtabs %}

### 新交易事件

交易池有新的交易待执行时会将这些交易广播出来，woker会订阅新交易事件:

{% tabs %}
{% tab title="miner/worker.go" %}
```go
func newWorker(config *Config, chainConfig *params.ChainConfig, engine consensus.Engine, eth Backend, mux *event.TypeMux, isLocalBlock func(*types.Block) bool) *worker {
    //省略代码
    // Subscribe NewTxsEvent for tx pool
	worker.txsSub = eth.TxPool().SubscribeNewTxsEvent(worker.txsCh)
    //省略代码
    go worker.mainLoop()  //事件处理goroutine
    //省略代码
}
```
{% endtab %}
{% endtabs %}

worker接收到这些事件后会交给事件处理线程:

{% tabs %}
{% tab title="miner/worker.go" %}
```go
func (w *worker) mainLoop() {
	defer w.txsSub.Unsubscribe()
	defer w.chainHeadSub.Unsubscribe()
	defer w.chainSideSub.Unsubscribe()
	for {
		select {
			case ev := <-w.txsCh:
			if !w.isRunning() && w.current != nil {t
				if gp := w.current.gasPool; gp != nil && gp.Gas() < params.TxGas {
					continue
				}
				w.mu.RLock()
				coinbase := w.coinbase
				w.mu.RUnlock()

				txs := make(map[common.Address]types.Transactions)
				for _, tx := range ev.Txs {
					acc, _ := types.Sender(w.current.signer, tx)
					txs[acc] = append(txs[acc], tx)
				}
				txset := types.NewTransactionsByPriceAndNonce(w.current.signer, txs)
				tcount := w.current.tcount
				w.commitTransactions(txset, coinbase, nil)  //提交事件
				if tcount != w.current.tcount {
					w.updateSnapshot()
				}
			} else {
				if w.chainConfig.Clique != nil && w.chainConfig.Clique.Period == 0 {
					w.commitNewWork(nil, true, time.Now().Unix())
				}
			}
			atomic.AddInt32(&w.newTxs, int32(len(ev.Txs)))
			//省略代码
		}
	}
}
```
{% endtab %}
{% endtabs %}

