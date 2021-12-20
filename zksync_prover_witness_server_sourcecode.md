## zksync prover witness server
这部分是zksync 验证部分的服务端，源码路径:`zksync/core/bin/zksync_witness_generator/src/lib.rs` `run_prover_server`
主要包含三个服务：`pool maintainer threads` `http server` `actix runtime`

### `http server`
构建http server，注册多个路由，并绑定到相应的地址上
注册的路由：

- `status` 
- `get_job`
- `working_on`
- `publish`
- `stopped`
- `api/internal/prover/replicas`

### `actix runtime`
用途：
- 更新prover queue 
- load last verified block 

#### `update prover job queue`
更新时间间隔： `5s`
核心threads方法：`update_prover_job_queue`

```rust
async fn update_prover_job_queue<DB:DatabaseInterface>(database:DB)->anyhow::Result<()>{
  
  let connection = database.connection;
  
  {
    // load block provider job queue from db
    let next_single_block_to_add = load_last_block_provider_job_queue.+1;  
    
    //load witness for next single block from db
    let witness_for_next_single_block  = db.load_witness;

    if let Some(witness) = witness_for_next_single_block{
    
      let provider_data = from_value(witness);
      let block_size = provider_data.operations.len();

      let job_data = serde_json::to_value(JobRequestData::BlockProof(prover_data, block_size));

      db.add_provider_job_to_job_queue();
    
    }
  
    {
    
      let next_aggregated_proof_block = db.load_last_block_provider_job_queue().+1;

      let create_block_proof_action = db.load_aggregated_op_that_affects_block();
           
      if let Some((
            _,
            AggregatedOperation::CreateProofBlocks(BlocksCreateProofOperation { blocks, .. }),      
            
            
            )) = create_block_proof_action
      {
       
        let first_block = blocks.first().map().expect("should have 1 block");

        let last_block = blocks.last().map().expect("should have 1 block");

        for block in blocks{
          
          let proof = db.load_proof();

          let block_size = block.block_size;
        
        }

        // serialize the aggregated proof 
        let job_data = serde_json::to_value(JobRequestData::AggregatedBlockProof(data))
        
        db.add_provider_job_to_job_queue(first_block,last_block,job_data,AGGREGATED_PROOF_JOB_PRIORITY,
                    ProverJobType::AggregatedProof,);
    
      }
      db.mark_stale_jobs_as_idle();
        
    }
  
  
  }

}
```

### `pool maintainer threads`
启动需要做的事情：
- 计算偏移量，然后添加至 `last_verified_block`
- 实例化` pool_maintainer`
- 启动服务

#### start pool maintainer
- thread name --->`prover_server_pool`
- build runtime for witness generator
- start maintain

##### start maintain
职能: `update witness data in db`
```rust

async fn maintain(self){

  
  get current block
  
  loop{
  
      let should_worl = should_work_on_block(current_block)  
  
      let next_block  = Self::next_witness_block(current_block,self.block_step,should_work); 
      
      if let BlockInfo::Nowitness(block) = should_work{
        
        let block_number = block.block_number;

        self.perpare_witness_and_save_it(block);
      
      }
      current_block = next_block;
  }
  
}
```

##### `should_work_on_block`
path `zksync/core/bin/zksync_witness_generator/src/witness_generator.rs`

> 返回给定区块的witness 状态
>

- load block from db
- load witness from db
- return block info 

##### `prepare_witness_and_save_it` 
- connect db
- load account tree
- build block witness 
- store witenss

###### `load account tree`

- connection db
- load account tree cache













