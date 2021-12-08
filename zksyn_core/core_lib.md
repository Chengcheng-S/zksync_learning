# core lib

### `run_core`

> 启动核心的程序，主要包含几个子模块：
>
> - eth watcher 用于链上监控的模块
> - zkSync  状态管理器，execute、seal block
> - mempool 组织传入交易的模块
> - block proposer 为状态保持这创建区块提议的模块
> - committer 将挂起和完成的块存储到数据里中的模块
> - 私有核心API服务

```rust
pub async fn run_core(
    connection_pool: ConnectionPool,
    panic_notify: mpsc::Sender<bool>,
    config: &ZkSyncConfig,
)->amyhow::Result<Vec<JoinHandle<()>>>{
    
    /*
    声明多个异步缓冲channel,容量为32_768
    主要包含 eth watch、statekeeper、 mempool、 block etc
    */
    let (proposed_blocks_sender, proposed_blocks_receiver)=mpsc::channel(xxx);
    
    let (state_keeper_req_sender, state_keeper_req_receiver) =mpsc::channel(xxx);
    
    let (eth_watch_req_sender, eth_watch_req_receiver) = mpsc::channel(xxx);
    
    let (mempool_tx_request_sender, mempool_tx_request_receiver) =mpsc::channel(xxx);
    
    let (mempool_block_request_sender, mempool_block_request_receiver) =mpsc::channel(xxx);
    
    
    //开始eth 监控
    let eth_watch_task = start_eth_watch(
    	&config,
        eth_watch_req_sender.clone(),
        eth_watch_req_receiver,
    );
 	   
    
    //将待处理的交易存入数据库
    let mut storage_processor = connection_pool.access_storage().await?;
    
    /*
    state keeper
    */
    //1、init
    ZkSyncStateInitParams::restore_from_db(&mut storage_processor).await?;
    
    //2、get pending block
    state_keeper_init
        .get_pending_block(&mut storage_processor)
        .await;
 	ZkSyncStateKeeper::new(xxx);
    
    // construct the state keeper task
    let state_keeper_task = start_state_keeper(state_keeper, pending_block);
    
    
    // committer  construct the committer task
    let committer_task = run_committer(
        proposed_blocks_receiver,
        mempool_block_request_sender.clone(),
        connection_pool.clone(),
        &config,
    );
    
    
    // construct mempool task
     let mempool_task = run_mempool_tasks(
        connection_pool.clone(),
        mempool_tx_request_receiver,
        mempool_block_request_receiver,
        eth_watch_req_sender.clone(),
        &config,
        4,
        DEFAULT_CHANNEL_CAPACITY,
    );
    
    
    // 启动拒绝交易清理任务
    let rejected_tx_cleaner_task = run_rejected_tx_cleaner(&config, connection_pool.clone());
    

    //contrauct block propose task
    let proposer_task = run_block_proposer_task(
        &config,
        mempool_block_request_sender.clone(),
        state_keeper_req_sender.clone(),
    );
    
    // private API
    
     start_private_core_api(
        panic_notify.clone(),
        mempool_tx_request_sender,
        eth_watch_req_sender,
        config.api.private.clone(),
    );
    
    
    // collect all task & return it
    let task_futures = vec![
        eth_watch_task,
        state_keeper_task,
        committer_task,
        mempool_task,
        proposer_task,
        rejected_tx_cleaner_task,
    ];
    
    Ok(task_futures)
    
}
```



### eth watch

```rust
#[must_use]
pub fn start_eth_watch(
	config_options: &ZkSyncConfig,
    eth_req_sender: mpsc::Sender<EthWatchRequest>,
    eth_req_receiver: mpsc::Receiver<EthWatchRequest>,
)->JoinHandle<()>{
    
	get eth client
    
    new eth watch
    
   	new a tikio async task
    
    poll interval
    
}
```

>  eth的数据源主要从infura.io (第三方)中获取，因此会对数据进行限流等操作。







