## ServiceManagerBase

original doc: [ServiceManagerBase.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/ServiceManagerBase.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`ServiceManagerBase.sol`](../src/ServiceManagerBase.sol) | Singleton | Transparent proxy |

`AVSDirectory` 用这个contract 记录 operator registration and deregistration

### registerOperatorToAVS

```solidity
function registerOperatorToAVS(
    address operator,                   // 要注册的运营商地址
    ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature  // 运营商的签名数据
) 
    public 
    virtual 
    onlyRegistryCoordinator            // 只允许 RegistryCoordinator 调用
```

workflow:

```solidity
// 1. Operator 发起注册请求
Operator 
    -> RegistryCoordinator.registerOperator()
        -> ServiceManagerBase.registerOperatorToAVS()
            -> AVSDirectory.registerOperatorToAVS()
```

功能:
```solidity
// 转发调用到 AVSDirectory
_avsDirectory.registerOperatorToAVS(operator, operatorSignature);
```

- 确保 operator 在 EigenLayer 和 AVS 中的状态一致
- 记录 operator 的注册信息

### registerOperatorToAVS

```solidity
function deregisterOperatorFromAVS(
    address operator    // 要注销的operator address
) 
    public 
    virtual 
    onlyRegistryCoordinator    // 只允许 RegistryCoordinator 调用
{
    _avsDirectory.deregisterOperatorFromAVS(operator);
}
```

调用入口：
1. 替换注销

```solidity
function registerOperatorWithChurn(
    address newOperator,
    address operatorToRemove
) external {
    // ... 验证逻辑 ...
    deregisterOperatorFromAVS(operatorToRemove);
    registerOperatorToAVS(newOperator, signature);
}
```

2. 主动注销：

```solidity
function deregisterOperator() external {
    address operator = msg.sender;
    // ... 验证逻辑 ...
    deregisterOperatorFromAVS(operator);
}
```

1. 强制eject：
```solidity
function ejectOperator(address operator) external onlyOwner {
    // ... 验证逻辑 ...
    deregisterOperatorFromAVS(operator);
}
```

4. 批量更新:
```solidity
function updateOperators(address[] memory operators) external {
    for(uint i = 0; i < operators.length; i++) {
        if(needsDeregistration(operators[i])) {
            deregisterOperatorFromAVS(operators[i]);
        }
    }
}
```

workflow:
```solidity
// 1. 主动注销场景
function exampleDeregisterFlow() {
    // Operator 调用注销
    registryCoordinator.deregisterOperator();
    
    // RegistryCoordinator 处理
    function deregisterOperator() internal {
        // 验证 operator 状态
        require(isOperatorRegistered(msg.sender));
        
        // 调用 ServiceManager
        serviceManager.deregisterOperatorFromAVS(msg.sender);
        
        // 清理本地状态
        _cleanupOperatorState(msg.sender);
    }
    
    // ServiceManager 转发到 AVSDirectory
    function deregisterOperatorFromAVS(address operator) {
        avsDirectory.deregisterOperatorFromAVS(operator);
    }
}

// 2. 强制驱逐场景
function exampleEjectionFlow() {
    // 管理员调用驱逐
    registryCoordinator.ejectOperator(badOperator);
    
    // 验证和执行驱逐
    function ejectOperator(address operator) {
        require(msg.sender == owner);
        
        // 调用注销
        deregisterOperatorFromAVS(operator);
        
        // 额外的惩罚逻辑
        _punishOperator(operator);
    }
}
```

状态更新：
```solidity
function _cleanupOperatorState(address operator) internal {
    // 1. 清除 quorum 注册
    bytes32 operatorId = getOperatorId(operator);
    uint192 currentBitmap = getCurrentQuorumBitmap(operatorId);
    _removeOperatorFromQuorums(operatorId, currentBitmap);
    
    // 2. 清除质押记录
    _stakeRegistry.removeOperatorStake(operator);
    
    // 3. 更新索引
    _indexRegistry.removeOperator(operator);
    
    // 4. 发出事件
    emit OperatorDeregistered(operator);
}
```

- 处理 operator 从 AVS 的注销
- 确保状态一致性
- 支持多种注销场景
