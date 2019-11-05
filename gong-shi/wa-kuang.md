---
description: 矿工执行以太坊的挖矿流程
---

# 矿工

## Miner结构

Miner结构定义了一个矿工，在创建以太坊实例时会创建一个Miner实例

{% code-tabs %}
{% code-tabs-item title="eth/backend.go" %}
```go
func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
    //省略代码
    eth.miner = miner.New(eth, &config.Miner, chainConfig, eth.EventMux(), eth.engine, eth.isLocalBlock)
	eth.miner.SetExtra(makeExtraData(config.Miner.ExtraData))
	//省略代码
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="miner/miner.go" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}



