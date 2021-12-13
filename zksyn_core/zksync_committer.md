# committer

>  模块入口，这部分是zksync中关于区块提交处理以及验证。 涉及两个核心的任务
>
> - handle_new_commit_task
> - poll_for_new_proofs_task

## Run_committer

```rust
#[must_use]
pub fn run_committer(
    rx_for_ops: Receiver<CommitRequest>,
    mempool_req_sender: Sender<MempoolBlocksRequest>,
    pool: ConnectionPool,
    config: &ZkSyncConfig,
) -> JoinHandle<()> {
    tokio::spawn(handle_new_commit_task(
        rx_for_ops,
        mempool_req_sender,
        pool.clone(),
    ));
    tokio::spawn(poll_for_new_proofs_task(pool, config.clone()))
}
```



### handle_new_commit_task

> 核心的方法则是 commit_block 
>
> 以及save_pending_block

```rust
async fn handle_new_commit_task(mut rx_for_ops: Receiver<CommitRequest>,
    mut mempool_req_sender: Sender<MempoolBlocksRequest>,
    pool: ConnectionPool,
) {
    while let Some(request) = rx_for_ops.next().await {
        match request {
            CommitRequest::Block((block_commit_request, applied_updates_req)) => {
                commit_block(
                    block_commit_request,
                    applied_updates_req,
                    &pool,
                    &mut mempool_req_sender,
                )
                    .await;
            }
            CommitRequest::PendingBlock((pending_block, applied_updates_req)) => {
                let mut operations = pending_block.success_operations.clone();
                operations.extend(
                    pending_block
                        .failed_txs
                        .clone()
                        .into_iter()
                        .map(|tx| ExecutedOperations::Tx(Box::new(tx))),
                );
                save_pending_block(pending_block, applied_updates_req, &pool).await;
            }
        }
    }
}
```



#### save_pending_block

> 在持久化pending_block 之后进行状态更改。

```rust
async fn asve_pending_block(pending_block:PendingBlock,applied_updates_request:AppliedUpdateRequest,pool:&ConnectionPool){
    
     // ....
    
        transaction
        .chain()
        .block_schema()
        .save_pending_block(pending_block)
        .await
        .expect("committer must commit the pending block into db");

    transaction
        .chain()
        .state_schema()
        .commit_state_update(
            block_number,
            &applied_updates_request.account_updates,
            applied_updates_request.first_update_order_id,
        )
        .await
        .expect("committer must commit the pending block into db");

    transaction
        .commit()
        .await
        .expect("Unable to commit DB transaction");
	
    // ...
}
```

#####  save_pending_block 

>  这部分就是真正存储pending block的逻辑，可以看出是使用原生的sql语句直接进行处理的。

```rust
async fn save_pending_block(&mut self,pending_block:PendingBlock)->QueryResult<()>{
    
    // do something
     // Store the pending block header.
        sqlx::query!("
            INSERT INTO pending_block (number, chunks_left, unprocessed_priority_op_before, pending_block_iteration, previous_root_hash, timestamp)
            VALUES ($1, $2, $3, $4, $5, $6)
            ON CONFLICT (number)
            DO UPDATE
              SET chunks_left = $2, unprocessed_priority_op_before = $3, pending_block_iteration = $4, previous_root_hash = $5, timestamp = $6
            ",
            storage_block.number, storage_block.chunks_left, storage_block.unprocessed_priority_op_before, storage_block.pending_block_iteration, storage_block.previous_root_hash,
            storage_block.timestamp
        ).execute(transaction.conn())
        .await?;

        // Store the transactions from the block.
        let executed_transactions = pending_block
            .success_operations
            .into_iter()
            .chain(
                pending_block
                    .failed_txs
                    .into_iter()
                    .map(|tx| ExecutedOperations::Tx(Box::new(tx))),
            )
            .collect();
        BlockSchema(&mut transaction)
            .save_block_transactions(pending_block.number, executed_transactions)
            .await?;
	// do somehing 
}
```



> 顺带一提，zksync中关于nft 的部分底层操作。源码位置:zksync/core/lib/storage/src/chain/state/mod.rs
>
> ![image-20211206102018683](C:\Users\35730\AppData\Roaming\Typora\typora-user-images\image-20211206102018683.png)



### poll_for_new_proofs_task

```rust
async fn poll_for_new_proofs_task(pool: ConnectionPool, config: ZkSyncConfig) {
    let mut timer = time::interval(PROOF_POLL_INTERVAL);
    loop {
        timer.tick().await;

        let mut storage = pool
            .access_storage()
            .await
            .expect("db connection failed for committer");

        aggregated_committer::create_aggregated_operations_storage(&mut storage, &config)
            .await
            .map_err(|e| vlog::error!("Failed to create aggregated operation: {}", e))
            .unwrap_or_default();
    }
}
```



#### create_aggregated_operations_storage

```rust
pub async fn create_aggregated_operations_storage(
    storage: &mut StorageProcessor<'_>,
    config: &ZkSyncConfig,
) -> anyhow::Result<()> {
    while create_aggregated_commits_storage(storage, config).await? {}
    while create_aggregated_prover_task_storage(storage, config).await? {}
    while create_aggregated_publish_proof_operation_storage(storage).await? {}
    while create_aggregated_execute_operation_storage(storage, config).await? {}

    Ok(())
}
```



































