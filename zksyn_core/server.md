## server
地址:zksync_dev/core/server

```toml
[dependencies]
anyhow = "1.0"   #用于处理Rust中的错误

structopt = "0.3.20" #通过定义结构来解析命令行参数

ctrlc = { version = "3.1", features = ["termination"] } #ctrl C 信号包装器

#异步处理
futures = "0.3" 
tokio = { version = "0.2", features = ["full"] } 

#基于verbosity level执行stdout/stderr 日志宏
vlog = { path = "../../lib/vlog", version = "1.0" }


[dev-dependencies]
#Rust中的数字类型和trait集合
num = { version = "0.3.1", features = ["serde"] }

# 序列化/反序列化框架
serde = "1.0.90"
#json 序列化文件格式
serde_json = "1.0.0"
```

> 主要是进行 api actor 以及eth sender prover server 等的初始化

```rust
#[tokio::main]
async fn mai()->anyhow::Result<()>{
    
    
    // 服务端初始化
    if let ServerCommand::Genesis = server_mode {
        vlog::info!("Performing the server genesis initialization",);
        genesis_init(&config).await;
        return Ok(());
    }
    
    /*
    初始化各类参数
    */
    
    // 初始化 connectionpool
    Connection::new(None);
    
    // ctrl C
    // 异步缓冲channel
    let (stop_signal_sender, mut stop_signal_receiver) = mpsc::channel(256);
    
    {
        let stop_signal_sender = RefCell::new(stop_signal_sender.clone());
        ctrlc::set_handler(move || {
            let mut sender = stop_signal_sender.borrow_mut();
            block_on(sender.send(true)).expect("Ctrl+C signal send");
        })
        .expect("Error setting Ctrl+C handler");
    }
    
    // prometheus data exporter  数据导出
    let (prometheus_task_handle, counter_task_handle) =
        run_prometheus_exporter(connection_pool.clone(), config.api.prometheus.port, true);
    
    
    // api actor
    let api_task_handle = run_api(connection_pool.clone(), stop_signal_sender.clone(), &config);
    
    // eth sender actor
     let eth_sender_task_handle = run_eth_sender(connection_pool.clone(), config.clone());
    
    //prover server & witness generator
     let database = zksync_witness_generator::database::Database::new(connection_pool);
    run_prover_server(database, stop_signal_sender, ZkSyncConfig::from_env());
    
    tokio::select! {
        _ = async { wait_for_tasks(core_task_handles).await } => {
        
        },
        _ = async { api_task_handle.await } => {
            panic!("API server actors aren't supposed to finish their execution")
        },
        _ = async { eth_sender_task_handle.await } => {
            panic!("Ethereum Sender actors aren't supposed to finish their execution")
        },
        _ = async { prometheus_task_handle.await } => {
            panic!("Prometheus exporter actors aren't supposed to finish their execution")
        },
        _ = async { counter_task_handle.unwrap().await } => {
            panic!("Operation counting actor is not supposed to finish its execution")
        },
        _ = async { stop_signal_receiver.next().await } => {
            vlog::warn!("Stop signal received, shutting down");
        }
    };

    Ok(())    
}
```

