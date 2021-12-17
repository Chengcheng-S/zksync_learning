## zksync gasfee calculate way
L2主要的目的就是减少L1的gascost，众所周知在L2中需要进行一定量的证明计算，也会涉及到各种的资源消耗，那么问题来了，L2的gascost是如何计算的呢？

从源码中来看zksync的源码计算方式还是有点繁琐，先从最基本的计算公式看起：
```shell
(zkp cost of chunk * number of chunks + gas price of transaction)*token risk factor /cost of token is usd
```
乍一看公式，chunk? chun number? risk factor? 以及token？ 咱一项一项的来看

- Chunk：就是zksync中op拆分的最小单位
- risk factor：以常数系数增加gasprice，由于gasprice 是不稳定的，因此在计算时将这部分风险包含在内

扫盲完毕之后，开始进入正题 **zkcync gasfee cosy 如何计算**？
关于gascost的这部分源码则是在`zksync_api` 中所定义的，相关连的服务为`ticker_task` (path `zksync/core/bin/zksync_api/src/lib.rs`)。在其内部又根据不同的tickerrequest换分为多种类型，每种类型也分别对应不同的处理逻辑。

### 单笔交易的费用 
源码中负责这部分的计算方法为`get_fee_from_ticker_in_wei`，感兴趣的同学可以去研究源码。

在进行计算gasfeecost之前，需要收集的几个关键变量：
- `zkp_cost_chunk`  这个op使用了多少个`chunk` (从配置中加载) 
- `scale_gas_prices ` 评估的gasprice 即 gas_price * (130/100)
- `wei_price_usd`    从`CoinMarketCap`  中加载的eth当前的价格   
- `token_usd_risk`   从config中加载的 `token_risk_factors` 然后这部分再与`token_price_usd` 相除 
- `fee_type` `gas_tx_amount` `op_chunks`  这些内容是需要从消息中加载，因为不同类型的消息在底层电路中定义的时候是不同的，剔除常规交易之外，存在`FastWithdraw` 以及`FastWithdrawNFT` 这两种交易的`gas_tx_amount` 计算方式又不同于其他的消息,即
  - commit_cost = calculate_cost(450_000,max_block_to_aggregate,block_to_commit);
  - execute_cost = calculate_cost(450_000,max_block_to_aggregate,block_to_commit);
  - proof_cost = calculate_cost(1_500_000,max_block_to_aggregate,block_to_prove);
  - cost_result = commit_cost + execute_cost + proof_cost 
  - 附：calculate_cost = base_cost - (base_cost/max_block)*(future_blocks % max_block)  
    以 commit_cost 为例: calculate_cost = 450_000 - (450_000/max_block_to_aggregate)*(block_to_commit % max_block_to_aggregate) 

- `zkp_fee`  (zkp_cost_chunk * op_chunks) * token_usd_risk
- `normal_gas_fee` =(wei_price_usd * gas_tx_amount * scale_gas_price) * token_usd_risk 
   针对L2的部分操作 normal_gas_fee *=scale_fee_coefficient(这个需要从config中加载)，部分操作则是:`TransferNew` `Tranfer` `MintNFT` `Swap`

最后给段源码，这就是最后传到channel中 返还给请求端的`normal_fee`
```rust 
let normal_fee = Fee::new(
            fee_type,
            zkp_fee,
            normal_gas_fee,
            gas_tx_amount,
            gas_price_wei.clone(),
        );
```

### 多笔消息聚合的gasfee cost 计算方式
`get_batch_from_ticker_in_wei` 
先前准备和单笔消息大同小异，差距则是在内部回迭代消息列表，然后将所有的`op_chunks` 以及`gas_tx_amount` 累加，然后聚合到一个BatchFee的结构体中，最后通过channel传递出去
```rust 
let normal_fee = BatchFee::new(total_zkp_fee, total_normal_gas_fee);
```

