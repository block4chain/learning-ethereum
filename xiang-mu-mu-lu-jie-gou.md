# 项目目录结构

```bash
.
├── accounts   #实现了一个高等级的以太坊账户管理
├── build #主要是编译和构建的一些脚本和配置
├── cmd #以太坊主要命令工作
               ├── abigen #Source code generator to convert Ethereum contract definitions into easy to use, compile-time type-safe Go packages
               ├── bootnode # 启动一个仅仅实现网络发现的节点
               ├── checkpoint-admin# 
               ├── clef
               ├── devp2p
               ├── ethkey
               ├── evm
               ├── faucet
               ├── geth #以太坊客户端
               ├── p2psim
               ├── puppeth
               ├── rlpdump
               ├── utils
               └── wnode
├── common #公用包
├── consensus #提供了以太坊的一些共识算法，比如ethhash, clique(proof-of-authority)
├── console
├── contracts # 
├── core
├── crypto
├── dashboard
├── docs
├── eth # 实现了以太坊的协议
├── ethclient # 提供了以太坊的RPC客户端
├── ethdb # eth的数据库(包括实际使用的leveldb和供测试使用的内存数据库)
├── ethstats
├── event
├── graphql
├── internal
├── les #实现了以太坊的轻量级协议子集
├── light #实现为以太坊轻量级客户端提供按需检索的功能
├── log #提供对人机都友好的日志信息
├── metrics
├── miner #提供以太坊的区块创建和挖矿
├── mobile
├── node #以太坊的多种类型的节点
├── p2p #以太坊p2p网络协议
├── params
├── rlp #以太坊序列化处理
├── rpc #
├── signer
├── swarm
├── tests
├── trie #以太坊重要的数据结构Package trie implements Merkle Patricia Tries
├── vendor
└── whisper #提供了whisper节点的协议
```

