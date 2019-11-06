---
description: 以太坊在底层KV存储中会记录一些区块链的全局信息，方便跟踪区块链的最新状态
---

# 数据存储

## 区块链配置

| Key | Value |
| :--- | :--- |
| ethereum-config-${genesis\_hash} | 网络配置, ChainConfig数据 |

## 区块数据

| Key | Value |
| :--- | :--- |
| h${bigendian\_block\_num}n | 块号对应的块哈希 |
| h${bigendian\_block\_num}${block\_hash}t | 块号对应的难度目标 |
| b${bigendian\_block\_num}${block\_hash} | 块号对应的块Body RLP编码数据 |
| H${block\_hash} | 块号的big endian编码 |
| h${bigendian\_block\_num}${block\_hash} | 块号对应的块Header RLP编码数据 |
| r${bigendian\_block\_num}${block\_hash} | 块号对应的块Receipts RLP编码数据 |
| LastBlock | 最新的块hash |
| LastFast | fast sync最新的块hash |
| LastHeader | 最新的块hash |

