# state_keeper

从server中一路查询下来 statekeeper 逻辑则是在

`zksync/core/bin/zksync_core/src/state_keeper/mod.rs`

```rust
async fn run((mut self, pending_block: Option<SendablePendingBlock>) {
	    
    self.initialize(pending_block).await;
    while let Some(req) = self.rx_for_blocks.next().await {
            match req {
                StateKeeperRequest::GetAccount(addr, sender) => {
                    sender.send(self.account(&addr)).unwrap_or_default();
                }
                StateKeeperRequest::GetPendingBlockTimestamp(sender) => {
                    sender
                        .send(self.pending_block.timestamp)
                        .unwrap_or_default();
                }
                StateKeeperRequest::GetLastUnprocessedPriorityOp(sender) => {
                    sender
                        .send(self.current_unprocessed_priority_op)
                        .unwrap_or_default();
                }
                StateKeeperRequest::ExecuteMiniBlock(proposed_block) => {
                    self.execute_proposed_block(proposed_block).await;
                }
                StateKeeperRequest::SealBlock => {
                    self.seal_pending_block().await;
                }
                StateKeeperRequest::GetCurrentState(sender) => {
                    sender.send(self.get_current_state()).unwrap_or_default();
                }
            }
        }
    
}
```

#### initialize

> 将已经执行的操作转换成未执行的操作，方便其再次执行。由于它是一个挂起的块，状态更新实际上并未应用到数据库中(**因为其仅在提交完整块时发生**)。使用 `apply_tx` 和 `apply_priority_op` 方法来保存原始执行顺序。

```rust
pub async fn initialize(&mut self, pending_block: Option<SendablePendingBlock>) {
        let start = Instant::now();
        if let Some(pending_block) = pending_block {
        	
        	let mut txs_count = 0;
            let mut priority_op_count = 0;
            for operation in pending_block.success_operations {
                match operation {
                    ExecutedOperations::Tx(tx) => {
                        self.apply_tx(&tx.signed_tx)
                            .expect("Tx from the restored pending block was not executed");
                        txs_count += 1;
                    }
                    ExecutedOperations::PriorityOp(op) => {
                        self.apply_priority_op(op.priority_op)
                            .expect("Priority op from the restored pending block was not executed");
                        priority_op_count += 1;
                    }
                }
            }
            self.pending_block.stored_account_updates = self.pending_block.account_updates.len();

            vlog::info!(
                "Executed restored proposed block: {} transactions, {} priority operations, {} failed transactions",
                txs_count,
                priority_op_count,
                pending_block.failed_txs.len()
            );
            self.pending_block.failed_txs = pending_block.failed_txs;
            self.pending_block.timestamp = pending_block.timestamp;
        } else {
            vlog::info!("There is no pending block to restore");
            
        }

		metrics::histogram!("state_keeper.initialize", start.elapsed());
}
```

#### apply_tx

```rust
fn apply_tx(&mut self, tx: &SignedZkSyncTx) -> Result<ExecutedOperations, ()> {
    
    let start = Instant::now();
    let chunks_needed = self.state.chunks_for_tx(&tx);
    
    
    /*
    若由于大小限制无法将tx添加至block中，将返回tx，seal block并再次执行
    */
    
    if self.pending_block.chunks_left < chunks_needed {
       return Err(());
    }
    
    
    //检查将此交易添加到区块是否使得合约操作成本过高
     let non_executed_op = self.state.zksync_tx_to_zksync_op(tx.tx.clone());
     if let Ok(non_executed_op) = non_executed_op {
     	
        if self
                .pending_block
                .gas_counter
                .add_op(&non_executed_op)
                .is_err()
            {
         		// 此时已经达到了gas上限，seal block，这笔交易将进入到下一笔
                return  Err(());
         	}

      }
 	   
      if let ZkSyncTx::Withdraw(tx) = &tx.tx {
          //检查是否应该将此块标记为需要快速处理
            if tx.fast {
                self.pending_block.fast_processing_required = true;
            }
        }
    
 	   // 执行消息
       let tx_updates = self.execute_tx(tx.tx.clone(), self.pending_block.timestamp);
    	
       /*
       处理执行的结果
       */

}
```



>  zksync 具体消息执行的逻辑，不同类型的apply_tx 被集成在execute_tx 中。不同的消息会走不同的处理分支。 

```rust
pub fn execute_tx(&mut self, tx: ZkSyncTx) -> Result<OpSuccess, Error> {
        match tx {
            ZkSyncTx::Transfer(tx) => self.apply_tx(*tx),
            ZkSyncTx::Withdraw(tx) => self.apply_tx(*tx),
            ZkSyncTx::Close(tx) => self.apply_tx(*tx),
            ZkSyncTx::ChangePubKey(tx) => self.apply_tx(*tx),
            ZkSyncTx::ForcedExit(tx) => self.apply_tx(*tx),
        }
    }
```



























