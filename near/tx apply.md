## TX lifetime

在之前已经介绍过了，NEAR 中的消息种类，以及 NEAR 是基于分片的 L1 层区块链平台。交易到底是如何执行，状态又是如何被更改的呢？从传统区块链的角度来看交易的生命周期无非就是：`客户签名` → `进入 mempool`（需要进行基础的校验，账户余额，nonce 等）→ `构建区块之后打包进区块` → `节点同步之后进行完整校验，执行通过之后则会进行状态的更改`。

NEAR 与传统区块链平台不同的地方则是分片的应用，所以交易生命周期部分还是类似的。

### 最开始的构建交易

通过 JS/Rust SDK 进行 dapp 或者其他的开发，发送一笔交易

```js
const account = await nearConnection.account("example-account.testnet");
const transactionOutcome = await account.deployContract(
  fs.readFileSync("example-file.wasm")
);
```

交易签名：ed25519/Secp256K1 这两个签名算法

### 监听

客户端监听到用户发送的消息，然后进行处理

```rust
fn handle(&mut self, msg: ProcessTxRequest) -> ProcessTxResponse {
        let ProcessTxRequest { transaction, is_forwarded, check_only } = msg;
        self.client.process_tx(transaction, is_forwarded, check_only)
    }
```

```rust
fn process_tx_internal(
        &mut self,
        tx: &SignedTransaction,
        is_forwarded: bool,
        check_only: bool,
        signer: &Option<Arc<ValidatorSigner>>,
    ) -> Result<ProcessTxResponse, Error> {
        // 1. 基础验证
        self.validate_transaction_basic(tx)?;
        
        // 2. 签名验证
        if !tx.signature.verify(tx.get_hash().as_ref(), &tx.transaction.public_key) {
            return Err(Error::InvalidTxSignature);
        }
        
        // 3. 检查nonce和账户状态
        let signer_account = self.get_account(&tx.transaction.signer_id)?;
        if tx.transaction.nonce <= signer_account.nonce {
            return Err(Error::InvalidNonce);
        }
        
        // 4. 检查余额是否足够支付gas费用
        let gas_cost = tx.transaction.gas_price * tx.transaction.gas;
        if signer_account.amount < gas_cost {
            return Err(Error::InsufficientBalance);
        }
        
        // 5. 路由到正确的分片
        let target_shard = self.get_target_shard(&tx.transaction.receiver_id);
        if target_shard != self.current_shard_id {
            return self.forward_transaction_to_shard(tx, target_shard);
        }
        
        // 6. 执行交易
        self.execute_transaction(tx, check_only)
      	//区块头校验
      // 检查交易是否过期：通过 cur_block_header 和 transaction_validity_period 确认交易是否在有效期内，避免处理过期的交易。
      if let Err(e) = self.chain.chain_store().check_transaction_validity_period(
            &cur_block_header,
            tx.transaction.block_hash(),
            transaction_validity_period,
        ) {
            debug!(target: "client", ?tx, "Invalid tx: expired or from a different fork");
            return Ok(ProcessTxResponse::InvalidTx(e));
        }
      
      // 交易签名& 格式校验
      // 第一次校验 (gas_price, None, tx, true): 不需要读取状态根，主要检查交易的签名、格式、以及 Gas 价格是否满足当前区块的最低要求。
      if let Some(err) = self
            .runtime_adapter
            .validate_tx(
                gas_price,
                None,
                tx,
                true,
                &epoch_id,
                protocol_version,
                receiver_congestion_info,
            )
            .expect("no storage errors")
        {
            debug!(target: "client", tx_hash = ?tx.get_hash(), ?err, "Invalid tx during basic validation");
            return Ok(ProcessTxResponse::InvalidTx(err));
        }
      
      ... get some args...
      
      // 分片层校验
      // 判断当前节点是否负责处理该分片 (care_about_shard, will_care_about_shard)。
      // 获取分片状态根 (state_root): 如果需要进行完整校验，则会尝试获取对应分片最新的状态根，用于后续的交易模拟执行。
      if care_about_shard || will_care_about_shard {
            let shard_uid = self.epoch_manager.shard_id_to_uid(shard_id, &epoch_id)?;
            let state_root = match self.chain.get_chunk_extra(&head.last_block_hash, &shard_uid) {
                Ok(chunk_extra) => *chunk_extra.state_root(),
                Err(_) => {
                    // Not being able to fetch a state root most likely implies that we haven't
                    //     caught up with the next epoch yet.
                    if is_forwarded {
                        return Err(Error::Other("Node has not caught up yet".to_string()));
                    } else {
                        self.forward_tx(&epoch_id, tx, signer)?;
                        return Ok(ProcessTxResponse::RequestRouted);
                    }
                }
            };
      
      // 第二次校验
      //第二次校验 (gas_price, Some(state_root), tx, false): 如果需要进行完整校验 (check_only 为 false)，则会读取对应分片的最新状态根 (state_root)，并进行更深入的校验，例如账户余额是否充足、交易是否符合智能合约的逻辑等。
       if let Some(err) = self
                .runtime_adapter
                .validate_tx(
                    gas_price,
                    Some(state_root),
                    tx,
                    false,
                    &epoch_id,
                    protocol_version,
                    receiver_congestion_info,
                )
                .expect("no storage errors")
            {
                debug!(target: "client", ?err, "Invalid tx");
                Ok(ProcessTxResponse::InvalidTx(err))
            } else if check_only {
                Ok(ProcessTxResponse::ValidTx)
            } else {
               
                if me.is_some() {
                  // 全部校验通过之后，则可以添加至tx pool, 交易池会进行重复交易检查 (Duplicate) 和容量限制检查 (NoSpaceLeft)。

                  match self.sharded_tx_pool.insert_transaction(shard_uid, tx.clone()){
                    ... do some thing ...
                  }
              }
      
       // 转发处理 (forward_tx)
       // 如果交易不是由当前节点负责处理的分片发出的，或者当前节点不是活跃的验证节点，则会将交易转发给其他节点进行处理。
       if self.active_validator(shard_id, signer)? {
                    trace!(target: "client", account = ?me, shard_id, tx_hash = ?tx.get_hash(), is_forwarded, "Recording a transaction.");
                    metrics::TRANSACTION_RECEIVED_VALIDATOR.inc();

                    if !is_forwarded {
                        self.possibly_forward_tx_to_next_epoch(tx, signer)?;
                    }
                    Ok(ProcessTxResponse::ValidTx)
                } else if !is_forwarded {
                    trace!(target: "client", shard_id, tx_hash = ?tx.get_hash(), "Forwarding a transaction.");
                    metrics::TRANSACTION_RECEIVED_NON_VALIDATOR.inc();
                    self.forward_tx(&epoch_id, tx, signer)?;
                    Ok(ProcessTxResponse::RequestRouted)
                } else {
                    trace!(target: "client", shard_id, tx_hash = ?tx.get_hash(), "Non-validator received a forwarded transaction, dropping it.");
                    metrics::TRANSACTION_RECEIVED_NON_VALIDATOR_FORWARDED.inc();
                    Ok(ProcessTxResponse::NoResponse)
                }
              
             ......
      
}
```

这才是第一步的校验，交易进入txpool，可以看到，这个检验还是相对严格的。

### 交易打包

在生成chunk的时候，去txpool打包消息，在打包完成之后又会放入到txpool中。是不是很疑惑。

如果消息直接被包含在了chunk中，但是由于网络或者共识等某些原因倒是chunk一致构建不成区块，那这些交易就无效了，放回至交易池也算是确保这批交易可以被最终确认。

* 等chunk构建成了区块，那批打包的交易会直接从txpool中移除。
* 在near的设计中 tx是包含在chunk里的，chunk最后会组成区块。

```rust
fn prepare_transactions(
        &mut self,
        shard_uid: ShardUId,
        prev_block: &Block,
        last_chunk: &ShardChunk,
        chunk_extra: &ChunkExtra,
    ) -> Result<PreparedTransactions, Error> {
      
      ... 做一些准备工作...
      
      let prepared_transactions = if let Some(mut iter) =
            sharded_tx_pool.get_pool_iterator(shard_uid)
        {
          ......
          runtime.prepare_transactions(
                storage_config,
                PrepareTransactionsChunkContext {
                    shard_id,
                    gas_limit: chunk_extra.gas_limit(),
                    last_chunk_transactions_size,
                },
                prev_block.into(),
                &mut iter,
                &mut chain.transaction_validity_check(prev_block.header().clone()),
                self.config.produce_chunk_add_transactions_time_limit.get(),
            )?
          
      }
      
       let reintroduced_count = sharded_tx_pool
            .reintroduce_transactions(shard_uid, &prepared_transactions.transactions);
        if reintroduced_count < prepared_transactions.transactions.len() {
            debug!(target: "client", reintroduced_count, num_tx = prepared_transactions.transactions.len(), "Reintroduced transactions");
        }
        Ok(prepared_transactions)
      
}
```

```rust
fn prepare_transactions(
        &self,
        storage_config: RuntimeStorageConfig,
        chunk: PrepareTransactionsChunkContext,
        prev_block: PrepareTransactionsBlockContext,
        transaction_groups: &mut dyn TransactionGroupIterator,
        chain_validate: &mut dyn FnMut(&SignedTransaction) -> bool,
        time_limit: Option<Duration>,
    ) -> Result<PreparedTransactions, Error> {
      // 初始化工作
     	... do initlize 
      
      //gas 等一些参数的计算，再次验证交易，确保区块中包含的是真实有效的交易
      let delayed_receipts_indices: DelayedReceiptIndices =
            near_store::get(&state_update, &TrieKey::DelayedReceiptIndices)?.unwrap_or_default();
      let min_fee = runtime_config.fees.fee(ActionCosts::new_action_receipt).exec_fee();
      let new_receipt_count_limit =
            get_new_receipt_count_limit(min_fee, gas_limit, delayed_receipts_indices);

      let size_limit: u64 = calculate_transactions_size_limit(
            protocol_version,
            &runtime_config,
            chunk.last_chunk_transactions_size,
            transactions_gas_limit,
       );
      
      ......
      
      // Verifying the transaction is on the same chain and hasn't expired yet.
      if !chain_validate(&tx) {
     	}
      // 根据当前状态验证交易的有效性。
      match verify_and_charge_transaction(
                    runtime_config,
                    &mut state_update,
                    prev_block.next_gas_price,
                    &tx,
                    false,
                    Some(next_block_height),
                    protocol_version,
                ) {
                    Ok(verification_result) => {
                        tracing::trace!(target: "runtime", tx=?tx.get_hash(), "including transaction that passed validation");
                        state_update.commit(StateChangeCause::NotWritableToDisk);
                        total_gas_burnt += verification_result.gas_burnt;
                        total_size += tx.get_size();
                        result.transactions.push(tx);
                        // Take one transaction from this group, no more.
                        break;
                    }        
    ......
}
```

### 构建区块

```rust
 pub fn produce(
        this_epoch_protocol_version: ProtocolVersion,
        next_epoch_protocol_version: ProtocolVersion,
        prev: &BlockHeader,
        height: BlockHeight,
        block_ordinal: crate::types::NumBlocks,
        chunks: Vec<ShardChunkHeader>,
        chunk_endorsements: Vec<ChunkEndorsementSignatures>,
        epoch_id: EpochId,
        next_epoch_id: EpochId,
        epoch_sync_data_hash: Option<CryptoHash>,
        approvals: Vec<Option<Box<near_crypto::Signature>>>,
        gas_price_adjustment_rate: Rational32,
        min_gas_price: Balance,
        max_gas_price: Balance,
        minted_amount: Option<Balance>,
        challenges_result: crate::challenge::ChallengesResult,
        challenges: Challenges,
        signer: &crate::validator_signer::ValidatorSigner,
        next_bp_hash: CryptoHash,
        block_merkle_root: CryptoHash,
        clock: near_time::Clock,
        sandbox_delta_time: Option<near_time::Duration>,
    ) -> Self {
      // 从chunk中计算 gasused,gaslimit,prev_validator_proposals等信息
      for chunk in chunks.iter() {
            if chunk.height_included() == height {
                prev_validator_proposals.extend(chunk.prev_validator_proposals());
                gas_used += chunk.prev_gas_used();
                gas_limit += chunk.gas_limit();
                balance_burnt += chunk.prev_balance_burnt();
                chunk_mask.push(true);
            } else {
                chunk_mask.push(false);
            }
        }
      // 计算 下一个块的 gas——price
        let next_gas_price = Self::compute_next_gas_price(
            prev.next_gas_price(),
            gas_used,
            gas_limit,
            gas_price_adjustment_rate,
            min_gas_price,
            max_gas_price,
        );
      // 总供应
      let new_total_supply = prev.total_supply() + minted_amount.unwrap_or(0) - balance_burnt;
      // 当时的时间戳
      let now = clock.now_utc().unix_timestamp_nanos() as u64;
      
      
      // 生成随机值
      let (vrf_value, vrf_proof) = signer.compute_vrf_with_proof(prev.random_value().as_ref());
      let random_value = hash(vrf_value.0.as_ref());
	
      //构建区块头，区块体
      let body = BlockBody::new(
            this_epoch_protocol_version,
            chunks,
            challenges,
            vrf_value,
            vrf_proof,
            chunk_endorsements,
        );
        let header = BlockHeader::new(
            this_epoch_protocol_version,
            next_epoch_protocol_version,
            height,
            *prev.hash(),
            body.compute_hash(),
            Block::compute_state_root(body.chunks()),
            Block::compute_chunk_prev_outgoing_receipts_root(body.chunks()),
            Block::compute_chunk_headers_root(body.chunks()).0,
            Block::compute_chunk_tx_root(body.chunks()),
            Block::compute_outcome_root(body.chunks()),
            time,
            Block::compute_challenges_root(body.challenges()),
            random_value,
            prev_validator_proposals,
            chunk_mask,
            block_ordinal,
            epoch_id,
            next_epoch_id,
            next_gas_price,
            new_total_supply,
            challenges_result,
            signer,
            *last_final_block,
            *last_ds_final_block,
            epoch_sync_data_hash,
            approvals,
            next_bp_hash,
            block_merkle_root,
            prev.height(),
            clock,
        );
      Self::block_from_protocol_version(
            this_epoch_protocol_version,
            next_epoch_protocol_version,
            header,
            body,
        )
      
}
```



```rust
pub fn chunks(&self) -> ChunksCollection {
        match self {
            Block::BlockV1(block) => ChunksCollection::V1(
                block.chunks.iter().map(|h| ShardChunkHeader::V1(h.clone())).collect(),
            ),
            Block::BlockV2(block) => ChunksCollection::V2(&block.chunks),
            Block::BlockV3(block) => ChunksCollection::V2(&block.body.chunks),
            Block::BlockV4(block) => ChunksCollection::V2(&block.body.chunks()),
        }
    }
```

### 执行交易

Apply-block----> apply-chunk----> apply



```rust
//runtime/runtime/src/lib.rs/apply
pub fn apply(
        &self,
        trie: Trie,
        validator_accounts_update: &Option<ValidatorAccountsUpdate>,
        apply_state: &ApplyState,
        incoming_receipts: &[Receipt],
        transactions: &[SignedTransaction],
        epoch_info_provider: &(dyn EpochInfoProvider),
        state_patch: SandboxStatePatch,
    ) -> Result<ApplyResult, RuntimeError> {
      // 更新验证者账户
       if let Some(validator_accounts_update) = validator_accounts_update {
            self.update_validator_accounts(
                &mut processing_state.state_update,
                validator_accounts_update,
                &mut processing_state.stats,
            )?;
        }
      
      // 处理迁移
      ... 准备工作...
      receipt_sink.forward_from_buffer(&mut processing_state.state_update, apply_state)?;
      
      
      // 处理交易
      self.process_transactions(&mut processing_state, &mut receipt_sink)?;
      
      // 处理回执
      let process_receipts_result =
            self.process_receipts(&mut processing_state, &mut receipt_sink)?;

      // 验证并且更新状态
      self.validate_apply_state_update(
            processing_state,
            process_receipts_result,
            own_congestion_info,
            validator_accounts_update,
            state_patch,
            outgoing_receipts,
        )
}

```

这里补充一下：

`process_transactions` 这个方法只是处理单个签名交易，并将其转换为交易收据。 而实际上的执行合约的消息/转账/部署合约/质押等一系列的操作处理，都是在`process_receipts` 里，也涉及了near 的vm（wasm）的执行合约。感兴趣的朋友可深入研究。

总结，从使用 SDK 签发交易开始，一直到交易打包成区块，交易一共被**验证了两次**，在构建区块的时候，有需要计算 gas used，下一个区块的 gas-price 等一系列的多工作，值得注意的是 tx 是包含在 chunk 中的，而多个 chunk 组成了一个区块。实际上执行消息是从 block → chunk → tx 的层层递进的。由于关注点并不在于 NEAR 的经济模型，感兴趣的朋友可以自行去研究，这部分设计的还是蛮有意思的。
