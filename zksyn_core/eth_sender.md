## Eth_sender

> eth_sender 主要是L2与 L1 通信所用  FIFO进行操作，确保所有的zksync  op转换为eth 可以接受的block
>
> 

入口：

```rust
#[must_use]
pub fn run_eth_sender(pool:ConnectionPool,eth_gateway:EthereumGateway,options:ZkSyncConfig)->JoinHandle<()>{
    let db =Database::new(pool);
    
    tokio::spawn(async move{
        let eth_sender =ETHSender::new(options.eth_sender,db,eth_gateway).await;
        
        eth_sender.run().await
    })
}
```

核心方法 `run`

> eth_sender  每个块更改必须执行一次某些活动
>
> `0` 作为初始值是为了确保再第一次迭代时，运行所有的活动

```rust
pub async fn run(mut self){
    let mut last_used_block=0;
    
    loop{
        // 每间隔X 秒执行一次加载程序
        tokio::time::sleep(self.options.sender.tx_poll_period()).await;
        if let Err(error) = self.load_new_operations().await {
                vlog::error!("Unable to restore operations from the database: {}", error);
                panic!("Unable to restore operations from the database: {}", error);
        }
        
        if self.options.sender.is_enabled {
                // 继续处理
                last_used_block = self.proceed_next_operations(last_used_block).await;
                // 更新gas 以保持最新的 最大的gaslimit
                self.gas_adjuster
                    .keep_updated(&self.ethereum, &self.db)
                    .await;
        }
        
    }
}
```

#### `proceed_next_operations`

> 该方法主要做两件事情：
>
> - 从`TXQueue` 弹出所有可用的交易并发送他们
> - 筛选所有正在进行的操作，过滤已经完成的操作并管理其余操作 (如：通过为卡住的操作发送 txs)
>
> 返回这些操作的 eth block number

```rust
async fn proceed_next_operations(&mut self,last_used_block:u64)->u64{
    // get current_block
    let current_block = match.self.ethereum.block_number().await{
        Ok(current_block)=> current_block.as_u64().
        Err(e)=>{
            Self::process_error(e).await;
            return last_used_block;
        }
    };
    
    while let Some(tx) = self.tx_queue.pop_front() {
          if let Err(e) = self.initialize_operation(tx.clone(), current_block).await {
                Self::process_error(e).await;
                //将未执行的操作返回到队列中，因为操作初始化失败因为着其没有存储在数据库中
                if let Err(err_message) = self.tx_queue.return_popped(tx) {
                    panic!(
                        "Failed return previous sent operation to the queue: {}",
                        err_message
                    );
                }
            }
        }
    
    	//
        if last_used_block != current_block {
            // Queue for storing all the operations that were not finished at this iteration.
            let mut new_ongoing_ops = VecDeque::new();

            // Commit the next operations (if any).
            while let Some(mut current_op) = self.ongoing_ops.pop_front() {
                
                let commitment = match self
                    .perform_commitment_step(&mut current_op, current_block)
                    .await
                {
                    Ok(commitment) => commitment,
                    Err(e) => {
                        Self::process_error(e).await;
                        OperationCommitment::Pending
                    }
                };

                match commitment {
                    OperationCommitment::Committed => {
                        // Free a slot for the next tx in the queue.
                        self.tx_queue.report_commitment();
                    }
                    OperationCommitment::Pending => {
                        // Poll this operation on the next iteration.
                        new_ongoing_ops.push_back(current_op);
                    }
                }
            }
            assert!(
                self.ongoing_ops.is_empty(),
                "Ongoing ops queue should be empty after draining"
            );
            // Store the ongoing operations for the next round.
            self.ongoing_ops = new_ongoing_ops;
        }

        metrics::histogram!("eth_sender.proceed_next_operations", start.elapsed());
        current_block

}
```









