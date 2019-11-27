---
description: 网络运行过程中会持久化一些状态数据到数据库中
---

# 数据存储

## 本地信息

| 键 | 值 | 描述 |
| :--- | :--- | :--- |
| local:{NodeID}:seq | int64 | 节点enr序列号 |

## 节点交互

服务层会持久化记录一些节点的信息

| 键 | 值 | 描述 |
| :--- | :--- | :--- |
| n:${NodeID}:discover:lastpong | uint64 | NodeID最后一次回复pong的时间 |
| n:${NodeID}:discover:lastping | uint64 | 对NodeID最后一次ping的时间 |
| n:${NodeID}:discover:findfail | uint64 | findnode查询目标NodeID时的错误次数 |

