---
description: 以太坊中的一些交易会固定消耗一定的Gas。
---

# 固定消耗

| 类型 | 消耗 | 用途 |
| :--- | :--- | :--- |
| TxGasContractCreation | 53000 | 创建合约 |
| TxGas | 21000 | 所有非创建合约的交易 |
| TxDataNonZeroGasFrontier | 68 | Frontier版本。交易数据字段中一个非0字节的Gas固定消耗 |
| TxDataNonZeroGasEIP2028 | 16 | EIP2028后。交易数据字段中一个非0字节的Gas固定消耗 |
| TxDataZeroGas | 4 | 交易数据字段中一个0字节的Gas固定消耗 |

