## zksync gas cost the source code
### txfee
```rust
async fn gas_fee_from_ticker_in_wei(
  &mut self,
  tx_type:TxFeeType,
  token:TokenLike,
  recipent:Address,
)->Result<ReponseFee,anyhow::Error>{

  zkp_cost_chunk
  token
  gas_price_wei
  risk_gas_price
  wei_price_used  
  token_usd_tisk
  
  /*
   针对不同类型的交易，分别获取他的fee_type,op_chunks,gas_tx_amount
   */
  get fee_typ,op_chunks,gas_tx_amount

  zkp_fee = (zkp_cost_chunk*op_chunks) * token_usd_risk
  
  normal_gas_fee  = wei_price_usd * gas_tx_amount * risk_gas_price * token_usd_risk

  // 若交易类型为 transfernew,transfer,mintnft,swap 则
  normal_gas_fee *= config.scale_fee_coefficient
  
  // 若为起来类型的交易，则不进行处理
  
  normal_fee={
    fee_type,
    zkp_fee,
    normal_gas_fee,
    gas_tx_amount,
  }
  OK(Reposnse{normal_fee})
}
```
#### `risk_gas_price` 计算方式
```shell
gas_price * 130/100
```
source code:
```rust
fn risk_gas_price_estimate(gas_price: BigUint) -> BigUint {
    gas_price * BigUint::from(130u32) / BigUint::from(100u32)
}
```
#### 快速提币的gas cost 计算方式 

```rust
async fn calcul_fast_withdrawal_gas_cost(&mut self,chunk_size:usize)->BigUint{

  future_blocks
  remining_pendinig_chunks
  additional_cost {
    if chunk_size>chunk{0}else{chunks*200}
  }
  // 计算op的基本price已经以块为单位支付了多少，便将剩余的添加至fast_withdrawal中
  commit_cost = calculate_cost(450_000,max_block_to_aggregate,block_to_commit);

  execute_cost = calculate_cost(450_000,max_block_to_aggregate,block_to_commit);

  proof_cost = calculate_cost(1_500_000,max_block_to_aggregate,block_to_prove);
  
  res_cost = commit_cost + execute_cost + proof_cost;
}
```

`calculate_cost`

```rust
fn calculate_cost(base_cost,max_block,future_blocks)->usize{
  base_cost - (base_cost/max_block)*(future_blocks % max_block)
}
```



### batch tx fee
聚合消息的gascost计算方式与单条消息的计算方式大同小异，也存在部分的差异,多了一层的消息迭代，以此来获取单消息的`fee_type`，`gas_tx_amount`,`op_chunks`,此处的处理方式则与单消息的处理方式一致，最后将`op_chunks` 以及`gas_tx_amount` 累加即可，源码：


```rust
async fn get_batch_from_ticker_in_wei(
        &mut self,
        token: TokenLike,
        txs: Vec<(TxFeeTypes, Address)>,
    ) -> anyhow::Result<ResponseBatchFee> {
    zkp_cost_chunk
    token
    gas_price_wei
    risk_gas_price
    wei_price_used  
    token_usd_tisk
    
    txs.into_itter().map(|tx|self.gas_tx_amount(tx_type,recipient).await;).fltter_map(|tx| if match!(Transfer|TransfertoNew|Swap|MintNFT){config.scale_fee_coefficient.clone()*gas_tx_amount}else{gas_tx_amount.into()});
    
    total_amount += gas_tx_amount;
    tocal_op +=op_chunk; 

    // 之后的处理方式则与单消息的一致
    let total_zkp_fee = (zkp_cost_chunk * total_op_chunks) * token_usd_risk.clone();
    let total_normal_gas_fee =
            (&wei_price_usd * total_normal_gas_tx_amount * &scale_gas_price) * &token_usd_risk;
    let normal_fee = BatchFee::new(total_zkp_fee, total_normal_gas_fee);
    Ok(ReponseBtchFee(normal_fee))
}

```

### get token price
```rust

async fn get_token_price(&self,tokne:TokenLike,request_type:TokenPriceRequestType,)->Result<BigDecimal,PriceError>{

   let facotr= match request_type{
    
    USDForOneWei=> 10^18
    USDForOneToken=>1,
   };

   get_last_price_from_api
   
   price = price.usd_price/factor 

}

```


zksync中估算L2发送至L1的交易，不会消耗gas，`zksync/core/lib/types/src/gas_counter.rs`

