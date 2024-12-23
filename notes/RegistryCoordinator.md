## RegistryCoordinator

original doc: [RegistryCoordinator.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/RegistryCoordinator.md)

| File | Type | Proxy? |
| -------- | -------- | -------- |
| [`RegistryCoordinator.sol`](../src/RegistryCoordinator.sol) | Singleton | Transparent proxy |

### general 

1. 作为 operator 注册/注销的主入口:
- registerOperator
- registerOperatorWithChurn 
- deregisterOperator
- ejectOperator

2. 与三个 Registry 的交互:
- BLSApkRegistry: 管理 BLS 公钥
- StakeRegistry: 管理质押
- IndexRegistry: 管理 operator 列表

3. 与 EigenLayer Core 的集成:
- 通过 ServiceManager 连接
- 同步注册/注销状态

### roles

- Owner: a permissioned role that can create and configure quorums as well as manage other roles
- Ejector: a permissioned role that can forcibly eject an operator from a quorum via RegistryCoordinator.ejectOperator
- Churn Approver: a permissioned role that signs off on operator churn in RegistryCoordinator.registerOperatorWithChurn

### Registering and Deregistering

registryCoordinator 四个核心 register/deregister 函数：

1. registerOperator:
- 基本注册流程
- 需要 BLS 公钥
- 有容量限制

2. registerOperatorWithChurn:
- 带替换的注册流程
- 需要 churnApprover 签名
- 有质押要求

3. deregisterOperator:
- 主动注销流程
- 清理状态
- 同步更新

4. ejectOperator:
- 强制驱逐
- 只能由 ejector 调用
- 与 deregister 共用逻辑

### registerOperator

```solidity
function registerOperator(
    bytes calldata quorumNumbers,      // 要注册的 quorum 编号数组
    string calldata socket,            // operator 的网络地址
    IBLSApkRegistry.PubkeyRegistrationParams calldata params,  // BLS 公钥参数
    SignatureWithSaltAndExpiry memory operatorSignature       // operator 签名
) external onlyWhenNotPaused(PAUSED_REGISTER_OPERATOR) {
    // 1. 获取或创建 operatorId
    bytes32 operatorId = _getOrCreateOperatorId(msg.sender, params);
    
    // 2. 注册到各个 Registry
    RegisterResults memory results = _registerOperator({
        operator: msg.sender,
        operatorId: operatorId,
        quorumNumbers: quorumNumbers,
        socket: socket,
        operatorSignature: operatorSignature
    });
    
    // 3. 验证容量限制
    for (uint256 i = 0; i < quorumNumbers.length; i++) {
        require(
            results.numOperatorsPerQuorum[i] <= _quorumParams[uint8(quorumNumbers[i])].maxOperatorCount,
            "operator count exceeds maximum"
        );
    }
}
```

### registerOperatorWithChurn


```solidity
function registerOperatorWithChurn(
    bytes calldata quorumNumbers,
    string calldata socket, 
    IBLSApkRegistry.PubkeyRegistrationParams calldata params,
    OperatorKickParam[] calldata operatorKickParams,      // 要替换的 operator 信息
    SignatureWithSaltAndExpiry memory churnApproverSignature,  // churnApprover 签名
    SignatureWithSaltAndExpiry memory operatorSignature
) external onlyWhenNotPaused(PAUSED_REGISTER_OPERATOR) {
    // 1. 验证输入
    require(
        operatorKickParams.length == quorumNumbers.length,
        "input length mismatch"
    );
    
    // 2. 获取或创建 operatorId
    bytes32 operatorId = _getOrCreateOperatorId(msg.sender, params);
    
    // 3. 验证 churnApprover 签名
    _verifyChurnApproverSignature({
        registeringOperator: msg.sender,
        registeringOperatorId: operatorId,
        operatorKickParams: operatorKickParams,
        churnApproverSignature: churnApproverSignature
    });
    
    // 4. 注册新 operator
    RegisterResults memory results = _registerOperator(...);
    
    // 5. 处理替换逻辑
    for (uint256 i = 0; i < quorumNumbers.length; i++) {
        if (results.numOperatorsPerQuorum[i] > maxOperatorCount) {
            // 验证替换条件
            _validateChurn({
                quorumNumber: uint8(quorumNumbers[i]),
                totalQuorumStake: results.totalStakes[i],
                newOperator: msg.sender,
                newOperatorStake: results.operatorStakes[i],
                kickParams: operatorKickParams[i],
                setParams: operatorSetParams
            });
            
            // 注销旧 operator
            _deregisterOperator(
                operatorKickParams[i].operator,
                quorumNumbers[i:i+1]
            );
        }
    }
}
```

This method performs similar steps to registerOperator above, except that for each quorum where the new Operator total exceeds the maxOperatorCount, the operatorKickParams are used to deregister a current Operator to make room for the new one.

### deregisterOperator

```solidity
function deregisterOperator(
    bytes calldata quorumNumbers  // 要注销的 quorum 编号数组
) external onlyWhenNotPaused(PAUSED_DEREGISTER_OPERATOR) {
    _deregisterOperator({
        operator: msg.sender,
        quorumNumbers: quorumNumbers
    });
}

function _deregisterOperator(
    address operator,
    bytes memory quorumNumbers
) internal virtual {
    // 1. 验证 operator 状态
    OperatorInfo storage operatorInfo = _operatorInfo[operator];
    require(
        operatorInfo.status == OperatorStatus.REGISTERED,
        "operator is not registered"
    );
    
    // 2. 更新 bitmap
    uint192 quorumsToRemove = BitmapUtils.orderedBytesArrayToBitmap(
        quorumNumbers,
        quorumCount
    );
    uint192 newBitmap = currentBitmap.minus(quorumsToRemove);
    
    // 3. 如果完全注销,更新状态
    if (newBitmap.isEmpty()) {
        operatorInfo.status = OperatorStatus.DEREGISTERED;
        serviceManager.deregisterOperatorFromAVS(operator);
    }
    
    // 4. 从各个 Registry 注销
    blsApkRegistry.deregisterOperator(operator, quorumNumbers);
    stakeRegistry.deregisterOperator(operatorId, quorumNumbers);
    indexRegistry.deregisterOperator(operatorId, quorumNumbers);
}
```


### ejectOperator

```solidity
function ejectOperator(
    address operator,
    bytes calldata quorumNumbers
) external onlyEjector {
    _deregisterOperator({
        operator: operator,
        quorumNumbers: quorumNumbers
    });
}
```

### Updating Registered Operators

#### updateOperators

更新每一个 operator 的contract view
Allows anyone to update the contracts' view of one or more Operators' stakes. For each currently-registered operator, this method calls StakeRegistry.updateOperatorStake, triggering an update of that Operator's stake.

`StakeRegistry.updateOperatorStake` 触发 operator 的 stake 更新. return bitmap of quorums 返回operator不符合minimum requirement的quorum bitmap
```solidity
function updateOperators(
    address[] calldata operators  // 要更新的 operator 地址数组
) external onlyWhenNotPaused(PAUSED_UPDATE_OPERATOR) {
    // 遍历每个 operator
    for (uint256 i = 0; i < operators.length; i++) {
        address operator = operators[i];
        OperatorInfo memory operatorInfo = _operatorInfo[operator];
        bytes32 operatorId = operatorInfo.operatorId;

        // 获取当前注册的 quorum bitmap
        uint192 currentBitmap = _currentOperatorBitmap(operatorId);
        
        // 更新每个 quorum 中的质押状态
        bytes memory quorumsToUpdate = BitmapUtils.bitmapToBytesArray(currentBitmap);
        _updateOperator(operator, operatorInfo, quorumsToUpdate);
    }
}
```

#### updateOperatorsForQuorum

```solidity
function updateOperatorsForQuorum(
    address[][] calldata operatorsPerQuorum,  // 每个 quorum 的 operator 列表
    bytes calldata quorumNumbers             // quorum 编号数组
) external onlyWhenNotPaused(PAUSED_UPDATE_OPERATOR) {
    // 1. 验证输入
    uint192 quorumBitmap = BitmapUtils.orderedBytesArrayToBitmap(quorumNumbers, quorumCount);
    require(
        operatorsPerQuorum.length == quorumNumbers.length,
        "input length mismatch"
    );

    // 2. 遍历每个 quorum
    for (uint256 i = 0; i < quorumNumbers.length; ++i) {
        uint8 quorumNumber = uint8(quorumNumbers[i]);
        address[] calldata currQuorumOperators = operatorsPerQuorum[i];

        // 3. 验证 operator 数量
        require(
            currQuorumOperators.length == indexRegistry.totalOperatorsForQuorum(quorumNumber),
            "operator count mismatch"
        );

        // 4. 遍历并更新每个 operator
        address prevOperatorAddress = address(0);
        for (uint256 j = 0; j < currQuorumOperators.length; ++j) {
            address operator = currQuorumOperators[j];
            
            // 验证 operator 状态和排序
            OperatorInfo memory operatorInfo = _operatorInfo[operator];
            require(operator > prevOperatorAddress, "operators must be sorted");
            
            // 更新质押状态
            _updateOperator(operator, operatorInfo, quorumNumbers[i:i+1]);
            prevOperatorAddress = operator;
        }

        // 5. 更新 quorum 时间戳
        quorumUpdateBlockNumber[quorumNumber] = block.number;
        emit QuorumBlockNumberUpdated(quorumNumber, block.number);
    }
}
```

#### updateSocket

socket：
```solidity
// 更新 socket 信息
function updateSocket(
    string memory socket  // 如 "ip:port"
) external {
    // 验证 operator 状态
    require(
        _operatorInfo[msg.sender].status == OperatorStatus.REGISTERED,
        "operator is not registered"
    );
    
    // event
    emit OperatorSocketUpdate(
        _operatorInfo[msg.sender].operatorId,
        socket
    );
}
```

```solidity
function updateSocket(string memory socket) external
```

operator emit event, updating their socket

### system config

#### createQuorum

```solidity
function createQuorum(
    OperatorSetParam memory operatorSetParams,  // quorum 的运营参数
    uint96 minimumStake,                       // 最低质押要求
    IStakeRegistry.StrategyParams[] memory strategyParams  // 质押策略参数
) external virtual onlyOwner {
    _createQuorum(operatorSetParams, minimumStake, strategyParams);
}

function _createQuorum(
    OperatorSetParam memory operatorSetParams,
    uint96 minimumStake,
    IStakeRegistry.StrategyParams[] memory strategyParams
) internal {
    // 1. 检查 quorum 数量上限
    uint8 prevQuorumCount = quorumCount;
    require(prevQuorumCount < MAX_QUORUM_COUNT, "max quorums reached");
    quorumCount = prevQuorumCount + 1;
    
    // 2. 初始化新的 quorum
    uint8 quorumNumber = prevQuorumCount;
    _setOperatorSetParams(quorumNumber, operatorSetParams);
    
    // 3. 在各个 Registry 中初始化
    stakeRegistry.initializeQuorum(quorumNumber, minimumStake, strategyParams);
    indexRegistry.initializeQuorum(quorumNumber);
    blsApkRegistry.initializeQuorum(quorumNumber);
}
```

- 创建新的 quorum
- 设置运营参数
- 在所有 Registry 中初始化

#### setOperatorSetParams

```solidity
function setOperatorSetParams(
    uint8 quorumNumber,                    // quorum 编号
    OperatorSetParam memory operatorSetParams  // 新的运营参数
) external onlyOwner quorumExists(quorumNumber) {
    _setOperatorSetParams(quorumNumber, operatorSetParams);
}
```

- maxOperatorCount: quorum 最大 operator 数量
- kickBIPsOfOperatorStake: 替换 operator 所需的额外质押比例
- kickBIPsOfTotalStake: 被替换 operator 的最低质押比例

#### setChurnApprover

```solidity
function setChurnApprover(
    address _churnApprover  // 新的审批者地址
) external onlyOwner {
    _setChurnApprover(_churnApprover);
}

// 验证审批者签名
function _verifyChurnApproverSignature(
    address registeringOperator,
    bytes32 registeringOperatorId, 
    OperatorKickParam[] memory operatorKickParams, 
    SignatureWithSaltAndExpiry memory churnApproverSignature
) internal {
    // 1. 验证签名未过期且未使用
    require(
        !isChurnApproverSaltUsed[churnApproverSignature.salt],
        "salt already used"
    );
    require(
        churnApproverSignature.expiry >= block.timestamp,
        "signature expired"
    );   

    // 2. 标记 salt 已使用
    isChurnApproverSaltUsed[churnApproverSignature.salt] = true;    

    // 3. 验证签名
    EIP1271SignatureUtils.checkSignature_EIP1271(
        churnApprover,
        calculateOperatorChurnApprovalDigestHash(
            registeringOperator,
            registeringOperatorId,
            operatorKickParams,
            churnApproverSignature.salt,
            churnApproverSignature.expiry
        ),
        churnApproverSignature.signature
    );
}
```

- 设置替换approver地址
- 验证替换approver签名
- 防止signature replay

#### setEjector

```solidity
function setEjector(
    address _ejector  // 新的ejector地址
) external onlyOwner {
    _setEjector(_ejector);
}

// 驱逐 operator
function ejectOperator(
    address operator,
    bytes calldata quorumNumbers
) external onlyEjector {
    _deregisterOperator({
        operator: operator,
        quorumNumbers: quorumNumbers
    });
}
```

- 设置ejector地址