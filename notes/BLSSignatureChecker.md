## BLSSignatureChecker

original doc: [BLSSignatureChecker.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/BLSSignatureChecker.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`BLSSignatureChecker.sol`](../src/BLSSignatureChecker.sol) | Singleton | Transparent proxy |
| [`OperatorStateRetriever.sol`](../src/OperatorStateRetriever.sol) | Singleton | None |

`BLSSignatureChecker` and `OperatorStateRetriever` 分别用于onchain and offchain BLS signature validation for the aggregate of a quorum's registered Operators.


### BLSSignatureChecker

1. checkSignatures 函数:
- 验证多个 quorum 的聚合签名
- 处理流程:
  1) 计算非签名者的总公钥(total nonsigner apk)
  2) 对每个 quorum:
     - 获取该 quorum 的总公钥
     - 减去非签名者的公钥
     - 计算签名者的总 stake
  3) 验证最终的 BLS 签名

2. trySignatureAndApkVerification 函数:
- 验证单个 BLS 签名
- 使用配对运算验证签名有效性
- 检查公钥和签名的一致性

1. 
- 防止过期状态(staleStakesForbidden)
- 权限控制(onlyCoordinatorOwner)

### OperatorStateRetriever

1. getOperatorState 函数:
- 查询指定 operator 在某个区块的状态
- 返回:
  - operator 所在的 quorum bitmap
  - 每个 quorum 中所有 operator 的信息

2. getCheckSignaturesIndices 函数:
- 获取验证签名所需的所有indices
- 包括:
  - 非签名者的 quorum bitmap 索引
  - quorum 的 APK 索引
  - 总 stake 索引
  - 非签名者的 stake 索引

3. 数据结构:
- Operator: 记录 operator 的基本信息
- CheckSignaturesIndices: 记录所有需要的索引



