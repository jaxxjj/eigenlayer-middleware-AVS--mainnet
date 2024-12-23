## IndexRegistry

original doc: [IndexRegistry.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/registries/IndexRegistry.md)

| File | Type | Proxy? |
| -------- | -------- | -------- |
| [`IndexRegistry.sol`](../../src/IndexRegistry.sol) | Singleton | Transparent proxy |

IndexRegistry 为每个 quorum 中的 operator 提供唯一index

### variables

```solidity
// 1. 当前indec映射
mapping(uint8 => mapping(bytes32 => uint32)) public currentOperatorIndex;

// 2. 历史index记录
mapping(uint8 => mapping(uint32 => OperatorUpdate[])) internal _operatorIndexHistory;

// 3. 历史数量记录
mapping(uint8 => QuorumUpdate[]) internal _operatorCountHistory;

// 4. operator 更新记录
struct OperatorUpdate {
    uint32 fromBlockNumber;  // 更新时的区块号
    bytes32 operatorId;      // operator ID
}

// 5. quorum更新记录
struct QuorumUpdate {
    uint32 fromBlockNumber;  // 更新时的区块号
    uint32 numOperators;     // operator数量
}
```

2. Register workflow

```solidity
function registerOperator(
    bytes32 operatorId,
    bytes calldata quorumNumbers
) public virtual onlyRegistryCoordinator returns(uint32[] memory) {
    uint32[] memory numOperatorsPerQuorum = new uint32[](quorumNumbers.length);

    for (uint256 i = 0; i < quorumNumbers.length; i++) {
        // 1. 验证 quorum 存在
        uint8 quorumNumber = uint8(quorumNumbers[i]);
        require(_operatorCountHistory[quorumNumber].length != 0, "quorum does not exist");

        // 2. 增加操作者数量
        uint32 newOperatorCount = _increaseOperatorCount(quorumNumber);
        
        // 3. 分配索引
        _assignOperatorToIndex({
            operatorId: operatorId,
            quorumNumber: quorumNumber,
            operatorIndex: newOperatorCount - 1
        });

        numOperatorsPerQuorum[i] = newOperatorCount;
    }

    return numOperatorsPerQuorum;
}
```

3. deregister

```solidity
function deregisterOperator(
    bytes32 operatorId,
    bytes calldata quorumNumbers
) public virtual onlyRegistryCoordinator {
    for (uint256 i = 0; i < quorumNumbers.length; i++) {
        uint8 quorumNumber = uint8(quorumNumbers[i]);
        
        // 1. 获取要移除的操作者索引
        uint32 operatorIndexToRemove = currentOperatorIndex[quorumNumber][operatorId];

        // 2. 减少操作者数量
        uint32 newOperatorCount = _decreaseOperatorCount(quorumNumber);
        
        // 3. 移除最后一个操作者
        bytes32 lastOperatorId = _popLastOperator(quorumNumber, newOperatorCount);
        
        // 4. 如果不是同一个操作者,重新分配索引
        if (operatorId != lastOperatorId) {
            _assignOperatorToIndex({
                operatorId: lastOperatorId,
                quorumNumber: quorumNumber,
                operatorIndex: operatorIndexToRemove
            });
        }
    }
}
```

4. 历史记录

```solidity
// 1. 更新操作者数量历史
function _updateOperatorCountHistory(
    uint8 quorumNumber,
    QuorumUpdate storage lastUpdate,
    uint32 newOperatorCount
) internal {
    if (lastUpdate.fromBlockNumber == uint32(block.number)) {
        // 同一区块内更新
        lastUpdate.numOperators = newOperatorCount;
    } else {
        // 新区块推入历史
        _operatorCountHistory[quorumNumber].push(QuorumUpdate({
            numOperators: newOperatorCount,
            fromBlockNumber: uint32(block.number)
        }));
    }
}

// 2. 更新操作者索引历史
function _updateOperatorIndexHistory(
    uint8 quorumNumber,
    uint32 operatorIndex,
    OperatorUpdate storage lastUpdate,
    bytes32 newOperatorId
) internal {
    if (lastUpdate.fromBlockNumber == uint32(block.number)) {
        // 同一区块内更新
        lastUpdate.operatorId = newOperatorId;
    } else {
        // 新区块推入历史
        _operatorIndexHistory[quorumNumber][operatorIndex].push(OperatorUpdate({
            operatorId: newOperatorId,
            fromBlockNumber: uint32(block.number)
        }));
    }
}
```

5. 查询

```solidity
// 1. 获取特定区块的操作者列表
function getOperatorListAtBlockNumber(
    uint8 quorumNumber,
    uint32 blockNumber
) external view returns (bytes32[] memory) {
    // 获取当时的操作者数量
    uint32 operatorCount = _operatorCountAtBlockNumber(quorumNumber, blockNumber);
    bytes32[] memory operatorList = new bytes32[](operatorCount);
    
    // 遍历获取每个索引对应的操作者
    for (uint256 i = 0; i < operatorCount; i++) {
        operatorList[i] = _operatorIdForIndexAtBlockNumber(
            quorumNumber,
            uint32(i),
            blockNumber
        );
        require(
            operatorList[i] != OPERATOR_DOES_NOT_EXIST_ID,
            "operator does not exist at the given block number"
        );
    }
    return operatorList;
}

// 2. 获取特定区块的操作者数量
function _operatorCountAtBlockNumber(
    uint8 quorumNumber,
    uint32 blockNumber
) internal view returns (uint32) {
    uint256 historyLength = _operatorCountHistory[quorumNumber].length;
    
    // 从后向前查找适用的记录
    for (uint256 i = historyLength; i > 0; i--) {
        QuorumUpdate memory quorumUpdate = _operatorCountHistory[quorumNumber][i - 1];
        if (quorumUpdate.fromBlockNumber <= blockNumber) {
            return quorumUpdate.numOperators;
        }
    }
    
    revert("quorum did not exist at given block number");
}
```
