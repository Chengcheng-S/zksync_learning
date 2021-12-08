# Deposit

入金操作，将ERC20 代币存入 L2

L1 层面使用zksync.sol 中的 `depositERC20`或`depositEth` (存入以太坊)方法进行入金操作，（调用该方法需要指定 token _address、token amount、zksync_address）， 该方法内部调用`registerDeposit`方法(注册deposit request，添加优先级请求并触发OnchainDeposit事件)

L2 方面通过eth watch 监听其变化，捕获到相应的交易，然后将其放到mempool中进行处理(这部分逻辑则是在mempool中)，然后将消息池中预先打包的块，交给block_proposer 处理(这部分逻辑则是在\zksync\core\bin\zksync_core\src\block_preposer.rs)

block_proposer 然后将block 移交到state_keeper 板块进行处理。

