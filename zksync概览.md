## zksync 概览
关键字：
- `Owner` L2中控制资产的用户
- `Operator` L2 操作
- `Assets in rollup` 由owner控制L2 智能合约中的资产
- `Rollup key` owner 的私钥
- `Rescue signature` 用owner私钥对其消息进行签名，该签名用于Rollup内部交易

### zksync operation 
由L1 发起，接收方则为L2部署在L1的智能合约
- `Deposit` 将资金由以太坊转移至zkSync中的指定账号。若zkSync中的该账号不存在，则会创建该账号，同时为该地址赋予一个numeric ID。 
- `FullExit` 在不与zkSync server交互的情况下，将资金由zkSync提取到以太坊。当检测到来自zkSync服务器节点的审查时 或 无法设置zkSync账户的signing key时（如，某地址对应的是智能合约），该操作可用于紧急退出。

L2 内部发起的交易
- `ChangePubKey` 设置或修改某账号的signing key。若无相应的signing key，该账号无法做出priority operation之外的任何操作。
- `Transfer` L2 内部账户之间的相互转账，若zkSync上的接收账号不存在，则会创建新账号并为其赋予一个numeric ID。
- `Withdraw` 将L2 的资金 ===> L1 。
- `ForcedExit` 从unonwed的账户中提取资金到对应的一个eth地址， unowned是**没有**设置`signing key`的账户。
   - 默认情况下，每个账户的`signing key` 为0，标记为`unowned`
   - 若要让账户能发起L2的交易，用户可以通过`ChangePubKey` 设置`signing key`
   - 在该类交易中，既不能指定以太坊地址，也不能指定交易金额。仅支持将target L2地址某种特定token的所有可用金额都提取至target L1 address。


