## BLSApkRegistry

original doc: [BLSApkRegistry.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/registries/BLSApkRegistry.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`BLSApkRegistry.sol`](../../src/BLSApkRegistry.sol) | Singleton | Transparent proxy |

### BLS public key 注册

```solidity
function registerBLSPublicKey(
    address operator,
    PubkeyRegistrationParams calldata params,
    BN254.G1Point calldata pubkeyRegistrationMessageHash
) external onlyRegistryCoordinator returns (bytes32 operatorId) {
    // 1. 验证公钥
    bytes32 pubkeyHash = BN254.hashG1Point(params.pubkeyG1);
    require(pubkeyHash != ZERO_PK_HASH, "cannot register zero pubkey");
    
    // 2. 验证未重复注册
    require(operatorToPubkeyHash[operator] == bytes32(0), "operator already registered");
    require(pubkeyHashToOperator[pubkeyHash] == address(0), "pubkey already registered");
    
    // 3. 验证签名
    uint256 gamma = uint256(keccak256(abi.encodePacked(
        params.pubkeyRegistrationSignature.X,
        params.pubkeyRegistrationSignature.Y,
        // ... other params
    ))) % BN254.FR_MODULUS;
    
    // 4. 存储映射关系
    operatorToPubkey[operator] = params.pubkeyG1;
    operatorToPubkeyHash[operator] = pubkeyHash;
    pubkeyHashToOperator[pubkeyHash] = operator;
}
```

BLS 公钥: operator 用于签名的密钥
pubkeyHash: 公钥的唯一标识

### APK aggregate public key 

```solidity
function _processQuorumApkUpdate(
    bytes memory quorumNumbers, 
    BN254.G1Point memory point
) internal {
    for (uint256 i = 0; i < quorumNumbers.length; i++) {
        uint8 quorumNumber = uint8(quorumNumbers[i]);
        
        // 更新聚合公钥
        BN254.G1Point memory newApk = currentApk[quorumNumber].plus(point);
        currentApk[quorumNumber] = newApk;
        
        // 记录历史
        bytes24 newApkHash = bytes24(BN254.hashG1Point(newApk));
        _updateApkHistory(quorumNumber, newApkHash);
    }
}
```

- quorum 中所有 operator 公钥的总和
- APK 更新: 注册/注销时自动更新
- APK Hash: 用于验证签名的聚合公钥哈希

###  APK 历史记录

```solidity
struct ApkUpdate {
    bytes24 apkHash;           
    uint32 updateBlockNumber;     // 更新时的区块号
    uint32 nextUpdateBlockNumber; // 下次更新的区块号
}

mapping(uint8 => ApkUpdate[]) public apkHistory;  // quorum => APK更新历史
```

- 每个 quorum 的 APK 变更历史
- 用于验证 APK 的连续性和一致性

### 查询机制

```solidity
function getApkHashAtBlockNumberAndIndex(
    uint8 quorumNumber,
    uint32 blockNumber,
    uint256 index
) external view returns (bytes24) {
    ApkUpdate memory quorumApkUpdate = apkHistory[quorumNumber][index];
    
    // 验证更新记录的有效性
    require(
        blockNumber >= quorumApkUpdate.updateBlockNumber,
        "index too recent"
    );
    require(
        quorumApkUpdate.nextUpdateBlockNumber == 0 || 
        blockNumber < quorumApkUpdate.nextUpdateBlockNumber,
        "not latest apk update"
    );
    
    return quorumApkUpdate.apkHash;
}
```

- 历史查询: 获取特定区块的 APK
- 时间验证: 确保使用正确的历史记录
- 索引验证: 防止使用过期数据

### register/deregister 

```solidity
function registerOperator(
    address operator,
    bytes memory quorumNumbers
) public virtual onlyRegistryCoordinator {
    // 1. 获取 operator 公钥
    (BN254.G1Point memory pubkey, ) = getRegisteredPubkey(operator);
    
    // 2. 更新 APK
    _processQuorumApkUpdate(quorumNumbers, pubkey);
}

function deregisterOperator(
    address operator,
    bytes memory quorumNumbers
) public virtual onlyRegistryCoordinator {
    // 1. 获取 operator 公钥
    (BN254.G1Point memory pubkey, ) = getRegisteredPubkey(operator);
    
    // 2. 从 APK 中减去公钥
    _processQuorumApkUpdate(quorumNumbers, pubkey.negate());
}
```

- 注册: 将公钥加入 APK
- 注销: 从 APK 中移除公钥
- 原子性: 确保 APK 始终正确
