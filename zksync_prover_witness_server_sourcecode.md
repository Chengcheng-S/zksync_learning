## zksync prover witness server
这部分是zksync 验证部分的服务端，源码路径:`zksync/core/bin/zksync_witness_generator/src/lib.rs` `run_prover_server`
主要包含三个服务：`pool maintainer threads` `http server` `actix runtime`

### `http server`











### `actix runtime`
用途：
- 更新prover queue 
- load last verified block 

#### `update prover job queue`
更新时间间隔： `5s`
核心threads方法：`update_prover_job_queue`
















