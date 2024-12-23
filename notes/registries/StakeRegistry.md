## StakeRegistry

original doc: [StakeRegistry.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/registries/StakeRegistry.md)
| File | Type | Proxy |
| -------- | -------- | -------- |
| [`StakeRegistry.sol`](../src/StakeRegistry.sol) | Singleton | Transparent proxy |

作用：
1. 跟踪每个 operator 在不同 quorum 中的 stake 权重
2. 跟踪每个 quorum 的总 stake 权重
3. 管理每个 quorum 的最小 stake 要求
4. 提供 stake 历史记录查询功能
   
### variables

1. operatorStakeHistory - 记录每个 operator 在每个 quorum 中的 stake 历史
2. _totalStakeHistory - 记录每个 quorum 的总 stake 历史
3. strategyParams - 记录每个 quorum 的策略参数(包括策略合约和权重)
4. minimumStakeForQuorum - 记录每个 quorum 的最小 stake 要求

### workflow

1. register:
- 计算 operator 的加权 stake
- 验证是否满足最小 stake 要求
- 更新 operator 的 stake 历史
- 更新 quorum 的总 stake 历史

2. unregister:
- 将 operator 的 stake 设为 0
- 从 quorum 的总 stake 中减去对应值
- 更新历史记录

3. update:
- 重新计算 operator 的加权 stake
- 如果不满足最小要求则设为 0
- 更新历史记录
- 返回需要移除的 quorum bitmap

### registerOperator

```solidity
function registerOperator(
    address operator,
    bytes32 operatorId,
    bytes calldata quorumNumbers
)
```

- 功能:记录 operator 的 stake 权重
- flow:
  1. 计算 stake 权重
  2. 更新 operator stake 历史
  3. 更新 quorum 总 stake 历史

### updateOperatorStake

```solidity
function updateOperatorStake(
    address operator,
    bytes32 operatorId,
    bytes calldata quorumNumbers
)
```

- 功能:更新 operator 的 stake 记录
- flow:
  1. 重新计算 stake 权重
  2. 更新历史记录
  3. 返回需要注销的 quorum