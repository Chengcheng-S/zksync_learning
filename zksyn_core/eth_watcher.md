# `eth-watch` 

这部分主要是从`infura.io`获取数据， `eth watch`  轮询一台节点以获取新的event，一旦足够多的量级的事件被确认，新事件就会被`zksync`网络接受。



## state

```rust
pub stcurt ETHStat{
    
    // 从eth watch监听到eth 最新的区块
    last_ethereum_block :u64,
	
    /*
    以太坊网络接受的优先操作队列，但还没有足够的确认来由 zkSync 处理。请注意，由于这些操作没有足够的确认，因此将来可能不会执行它们，因此此列表是近似值。
    */
    unconfirmed_queuq : Vec<PriorityOp>,
    
    
    /*
    键为具有PriorityOp的块数，超过确认阈值并等待执行的优先级操作队列
    */
    priority_queue:HashMap<u64, ReceivedPriorityOp>,
    
}
```



`zksync/core/bin/zksync_core/src/bin/eth_watcher.rs`

>  这部分程序主要是阐述eth watch 启动时需要做的事情，则是将eth_watch  作为一个task 异步的进行处理。

```rust
fn main(){
    
    let mut main_runtime = Runtime::new().expect("main runtime start");

    vlog::init();
    vlog::info!("ETH watcher started");

    let config = ZkSyncConfig::from_env();
    let client = EthereumGateway::from_config(&config);

    let (eth_req_sender, eth_req_receiver) = mpsc::channel(256);

    let eth_client = EthHttpClient::new(client, config.contracts.contract_addr);
    let watcher = EthWatch::new(eth_client, 0);

    main_runtime.spawn(watcher.run(eth_req_receiver));
    main_runtime.block_on(async move {
        let mut timer = time::interval(Duration::from_secs(1));

        loop {
            timer.tick().await;
            eth_req_sender
                .clone()
                .send(EthWatchRequest::PollETHNode)
                .await
                .expect("ETH watch receiver dropped");
        }
    });
    
}
```

处理的核心逻辑则是在`watcher.run` 中

### run

```rust
pub async fn run(mut self, mut eth_watch_req: mpsc::Receiver<EthWatchRequest>) {
	//针对data suorce 需要进行单独的处理，
    /*
    do something 
    */
    
    //restore eth watcher
    self.restore_state_from_eth(block).await.expect("Unable to restore ETHWatcher state");
    
    // 轮询eth节点，然后处理轮询的结果
    
    /*
    do some thing
    ...
    */
    
    // 同时对于轮询的结果进行处理
    let poll_result = self.poll_eth_node().await;
    
    //进行错误分析，检测是否被限制，若是则进行back off 模式，否则则处理新块的时候失败，
    
    /*
    do something
    */
    
    
   
   /*
   
   之后则是获取优先处理的队列
   获取未经确认的押金
   获取未经证实的操作
   通过哈希获取未经确认的操作
   以及是否授权公钥更改
   */ 
}
```



#### `poll_eth_node`

> 整体逻辑则是对比是否有新块，有的话则进行处理新块。

```rust
    async fn poll_eth_node(&mut self) -> anyhow::Result<()> {
        let start = Instant::now();
        let last_block_number = self.client.block_number().await?;

        if last_block_number > self.eth_state.last_ethereum_block() {
            self.process_new_blocks(last_block_number).await?;
        }

        metrics::histogram!("eth_watcher.poll_eth_node", start.elapsed());
        Ok(())
    
}
```

#### `process_new_blocks`

```rust
async fn process_new_block(&mut self, last_ethereum_block: u64)->anyhow::Result<()>{
    debug_assert!(self.eth_state.last_ethereum_block() < last_ethereum_block);
    // 此处则是处理处于back off 所遗漏的块
    let block_difference =
            last_ethereum_block.saturating_sub(self.eth_state.last_ethereum_block());
	// 主要是获取优先级队列以及未处理的队列
    let (unconfirmed_queue, received_priority_queue) = self
            .update_eth_state(last_ethereum_block, block_difference)
            .await?;
    
    let mut priority_queue = sift_outdated_ops(self.eth_state.priority_queue());
        for (serial_id, op) in received_priority_queue {
            priority_queue.insert(serial_id, op);
        }
		
	// 更新状态    
     let new_state = ETHState::new(last_ethereum_block, unconfirmed_queue, priority_queue);
     self.set_new_state(new_state);
        
    Ok(())
}
```



















