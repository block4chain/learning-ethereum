# 区块链

{% code-tabs %}
{% code-tabs-item title="core/blockchain.go" %}
```go
type BlockChain struct {
	chainConfig *params.ChainConfig // Chain & network configuration
	cacheConfig *CacheConfig        // Cache configuration for pruning

	db     ethdb.Database // Low level persistent database to store final content in
	triegc *prque.Prque   // Priority queue mapping block numbers to tries to gc
	gcproc time.Duration  // Accumulates canonical block processing for trie dumping

	hc            *HeaderChain
	rmLogsFeed    event.Feed
	chainFeed     event.Feed
	chainSideFeed event.Feed
	chainHeadFeed event.Feed
	logsFeed      event.Feed
	blockProcFeed event.Feed
	scope         event.SubscriptionScope
	genesisBlock  *types.Block

	chainmu sync.RWMutex // blockchain insertion lock

	currentBlock     atomic.Value // Current head of the block chain
	currentFastBlock atomic.Value // Current head of the fast-sync chain (may be above the block chain!)

	stateCache    state.Database // State database to reuse between imports (contains state cache)
	bodyCache     *lru.Cache     // Cache for the most recent block bodies
	bodyRLPCache  *lru.Cache     // Cache for the most recent block bodies in RLP encoded format
	receiptsCache *lru.Cache     // Cache for the most recent receipts per block
	blockCache    *lru.Cache     // Cache for the most recent entire blocks
	txLookupCache *lru.Cache     // Cache for the most recent transaction lookup data.
	futureBlocks  *lru.Cache     // future blocks are blocks added for later processing

	quit    chan struct{} // blockchain quit channel
	running int32         // running must be called atomically
	// procInterrupt must be atomically called
	procInterrupt int32          // interrupt signaler for block processing
	wg            sync.WaitGroup // chain processing wait group for shutting down

	engine     consensus.Engine
	validator  Validator  // Block and state validator interface
	prefetcher Prefetcher // Block state prefetcher interface
	processor  Processor  // Block transaction processor interface
	vmConfig   vm.Config

	badBlocks       *lru.Cache                     // Bad block cache
	shouldPreserve  func(*types.Block) bool        // Function used to determine whether should preserve the given block.
	terminateInsert func(common.Hash, uint64) bool // Testing hook used to terminate ancient receipt chain insertion.
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 配置

{% code-tabs %}
{% code-tabs-item title="params/config.go" %}
```go
type ChainConfig struct {
	ChainID *big.Int `json:"chainId"` // 链ID

	DAOForkBlock   *big.Int `json:"daoForkBlock,omitempty"`   // TheDAO硬分叉切换块
	DAOForkSupport bool     `json:"daoForkSupport,omitempty"` // 是否启用TheDAO硬分叉

	// EIP150 implements the Gas price changes (https://github.com/ethereum/EIPs/issues/150)
	EIP150Block *big.Int    `json:"eip150Block,omitempty"` // EIP150 HF block (nil = no fork)
	EIP150Hash  common.Hash `json:"eip150Hash,omitempty"`  // EIP150 HF hash (needed for header only clients as only gas pricing changed)

	EIP155Block *big.Int `json:"eip155Block,omitempty"` // EIP155 HF block
	EIP158Block *big.Int `json:"eip158Block,omitempty"` // EIP158 HF block

	HomesteadBlock *big.Int `json:"homesteadBlock,omitempty"` // Homestead版本切换块
	ByzantiumBlock      *big.Int `json:"byzantiumBlock,omitempty"`      // Byzantium switch block (nil = no fork, 0 = already on byzantium)
	ConstantinopleBlock *big.Int `json:"constantinopleBlock,omitempty"` // Constantinople switch block (nil = no fork, 0 = already activated)
	PetersburgBlock     *big.Int `json:"petersburgBlock,omitempty"`     // Petersburg switch block (nil = same as Constantinople)
	IstanbulBlock       *big.Int `json:"istanbulBlock,omitempty"`       // Istanbul switch block (nil = no fork, 0 = already on istanbul)
	EWASMBlock          *big.Int `json:"ewasmBlock,omitempty"`          // EWASM switch block (nil = no fork, 0 = already activated)

	// Various consensus engines
	Ethash *EthashConfig `json:"ethash,omitempty"`
	Clique *CliqueConfig `json:"clique,omitempty"`
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

`ChainConfig`包含区块链的一些核心配置，这些配置被存储创世块中，网络可以自定义参数。

### 主网

{% code-tabs %}
{% code-tabs-item title="params/config.go" %}
```go
MainnetChainConfig = &ChainConfig{
		ChainID:             big.NewInt(1),
		HomesteadBlock:      big.NewInt(1150000),
		DAOForkBlock:        big.NewInt(1920000),
		DAOForkSupport:      true,
		EIP150Block:         big.NewInt(2463000),
		EIP150Hash:          common.HexToHash("0x2086799aeebeae135c246c65021c82b4e15a2c451340993aacfd2751886514f0"),
		EIP155Block:         big.NewInt(2675000),
		EIP158Block:         big.NewInt(2675000),
		ByzantiumBlock:      big.NewInt(4370000),
		ConstantinopleBlock: big.NewInt(7280000),
		PetersburgBlock:     big.NewInt(7280000),
		IstanbulBlock:       nil,
		Ethash:              new(EthashConfig),
	}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 测试网

{% code-tabs %}
{% code-tabs-item title="params/config.go" %}
```go
TestnetChainConfig = &ChainConfig{
		ChainID:             big.NewInt(3),
		HomesteadBlock:      big.NewInt(0),
		DAOForkBlock:        nil,
		DAOForkSupport:      true,
		EIP150Block:         big.NewInt(0),
		EIP150Hash:          common.HexToHash("0x41941023680923e0fe4d74a34bdac8141f2540e3ae90623718e47d66d1ca4a2d"),
		EIP155Block:         big.NewInt(10),
		EIP158Block:         big.NewInt(10),
		ByzantiumBlock:      big.NewInt(1700000),
		ConstantinopleBlock: big.NewInt(4230000),
		PetersburgBlock:     big.NewInt(4939394),
		IstanbulBlock:       nil,
		Ethash:              new(EthashConfig),
	}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 其它测试网

* Rinke
* Goerli

## 创世块

{% code-tabs %}
{% code-tabs-item title="core/genesis.go" %}
```go
type Genesis struct {
	Config     *params.ChainConfig `json:"config"`
	Nonce      uint64              `json:"nonce"`
	Timestamp  uint64              `json:"timestamp"`
	ExtraData  []byte              `json:"extraData"`
	GasLimit   uint64              `json:"gasLimit"   gencodec:"required"`
	Difficulty *big.Int            `json:"difficulty" gencodec:"required"`
	Mixhash    common.Hash         `json:"mixHash"`
	Coinbase   common.Address      `json:"coinbase"`
	Alloc      GenesisAlloc        `json:"alloc"      gencodec:"required"`

	// These fields are used for consensus tests. Please don't use them
	// in actual genesis blocks.
	Number     uint64      `json:"number"`
	GasUsed    uint64      `json:"gasUsed"`
	ParentHash common.Hash `json:"parentHash"`
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

创世块定义区块链网络的核心配置，结构体`Genesis`定义了创世块的生成模板.

### 主网创世块

{% code-tabs %}
{% code-tabs-item title="core/genesis.go" %}
```go
func DefaultGenesisBlock() *Genesis {
	return &Genesis{
		Config:     params.MainnetChainConfig,
		Nonce:      66,
		ExtraData:  hexutil.MustDecode("0x11bbe8db4e347b4e8c937c1c8370e4b5ed33adb3db69cbdb7a38e1e50b1b82fa"),
		GasLimit:   5000,
		Difficulty: big.NewInt(17179869184),
		Alloc:      decodePrealloc(mainnetAllocData),
	}
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 设置并加载网络配置

{% code-tabs %}
{% code-tabs-item title="core/genesis.go" %}
```go
func SetupGenesisBlockWithOverride(db ethdb.Database, genesis *Genesis, overrideIstanbul *big.Int) (*params.ChainConfig, common.Hash, error) {
	//自定义创世块配置
	if genesis != nil && genesis.Config == nil {
		return params.AllEthashProtocolChanges, common.Hash{}, errGenesisNoConfig
	}
	// 查看是否创世块已经添加
	stored := rawdb.ReadCanonicalHash(db, 0)
	if (stored == common.Hash{}) {
		if genesis == nil {
			genesis = DefaultGenesisBlock()  //默认使用主网
		} 
		block, err := genesis.Commit(db)  //提交创世块
		if err != nil {
			return genesis.Config, common.Hash{}, err
		}
		return genesis.Config, block.Hash(), nil
	}
	//创世块已经建立
	header := rawdb.ReadHeader(db, stored, 0)
	if _, err := state.New(header.Root, state.NewDatabaseWithCache(db, 0)); err != nil {
		//创世块statedb有问题，重新初始化
		if genesis == nil {
			genesis = DefaultGenesisBlock()
		}
		// Ensure the stored genesis matches with the given one.
		hash := genesis.ToBlock(nil).Hash()
		if hash != stored {
			return genesis.Config, hash, &GenesisMismatchError{stored, hash}
		}
		block, err := genesis.Commit(db)
		if err != nil {
			return genesis.Config, hash, err
		}
		return genesis.Config, block.Hash(), nil
	}
	//检查创世块没有被篡改
	if genesis != nil {
		hash := genesis.ToBlock(nil).Hash()
		if hash != stored {
			return genesis.Config, hash, &GenesisMismatchError{stored, hash}
		}
	}
	newcfg := genesis.configOrDefault(stored)  //最新区块链配置
	if overrideIstanbul != nil {
		newcfg.IstanbulBlock = overrideIstanbul
	}
	storedcfg := rawdb.ReadChainConfig(db, stored)   //已经存储的区块链配置
	if storedcfg == nil {
		log.Warn("Found genesis block without chain config")
		rawdb.WriteChainConfig(db, stored, newcfg)
		return newcfg, stored, nil
	}
	if genesis == nil && stored != params.MainnetGenesisHash {
		return storedcfg, stored, nil
	}

	height := rawdb.ReadHeaderNumber(db, rawdb.ReadHeadHeaderHash(db))
	if height == nil {
		return newcfg, stored, fmt.Errorf("missing block number for head header hash")
	}
	compatErr := storedcfg.CheckCompatible(newcfg, *height)  //检查兼容性
	if compatErr != nil && *height != 0 && compatErr.RewindTo != 0 {
		return newcfg, stored, compatErr
	}
	rawdb.WriteChainConfig(db, stored, newcfg)  //写入新配置
	return newcfg, stored, nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 生成创世块

根据Genesis实例生成一个创世块:

{% code-tabs %}
{% code-tabs-item title="core/genesis.go" %}
```go
func (g *Genesis) ToBlock(db ethdb.Database) *types.Block {
	if db == nil {
		db = rawdb.NewMemoryDatabase()
	}
	statedb, _ := state.New(common.Hash{}, state.NewDatabase(db))
	for addr, account := range g.Alloc {
		statedb.AddBalance(addr, account.Balance)
		statedb.SetCode(addr, account.Code)
		statedb.SetNonce(addr, account.Nonce)
		for key, value := range account.Storage {
			statedb.SetState(addr, key, value)
		}
	}
	root := statedb.IntermediateRoot(false)
	head := &types.Header{
		Number:     new(big.Int).SetUint64(g.Number),
		Nonce:      types.EncodeNonce(g.Nonce),
		Time:       g.Timestamp,
		ParentHash: g.ParentHash,
		Extra:      g.ExtraData,
		GasLimit:   g.GasLimit,
		GasUsed:    g.GasUsed,
		Difficulty: g.Difficulty,
		MixDigest:  g.Mixhash,
		Coinbase:   g.Coinbase,
		Root:       root,
	}
	if g.GasLimit == 0 {
		head.GasLimit = params.GenesisGasLimit
	}
	if g.Difficulty == nil {
		head.Difficulty = params.GenesisDifficulty
	}
	statedb.Commit(false)
	statedb.Database().TrieDB().Commit(root, true)

	return types.NewBlock(head, nil, nil, nil)
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}



