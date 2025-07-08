## Shard

NEAR 中分片的概念，在 NEAR 中整个网络被分为若干的分片，他们各自维护自己的状态。分片之间可以进行跨分片通信。

先说分片上验证者的分配，还是回顾一下之前提到的 EpochManager，在每个 epoch 开始/结束的时候都会重新做一些事情，其中就包含了验证者分配，分配到不同的分片上去

```rust
pub fn proposals_to_epoch_info(
    epoch_config: &EpochConfig,
    rng_seed: RngSeed,
    prev_epoch_info: &EpochInfo,
    proposals: Vec<ValidatorStake>,
    mut validator_kickout: HashMap<AccountId, ValidatorKickoutReason>,
    validator_reward: HashMap<AccountId, Balance>,
    minted_amount: Balance,
    protocol_version: ProtocolVersion,
    use_stable_shard_assignment: bool,
) -> Result<EpochInfo, EpochError> {
  
  ... do somethings ...
  
  // 验证者选举
 	let validator_roles = if ProtocolFeature::StatelessValidation.enabled(protocol_version) {
        select_validators_from_proposals(epoch_config, proposals, protocol_version)
    } else {
        old_validator_selection::select_validators_from_proposals(
            epoch_config,
            proposals,
            protocol_version,
        )
    };
  
  
  // 验证者分配至分片
  let ChunkProducersAssignment {
        all_validators,
        validator_to_index,
        mut chunk_producers_settlement,
    } = if ProtocolFeature::StatelessValidation.enabled(protocol_version) {
        get_chunk_producers_assignment(
            epoch_config,
            rng_seed,
            prev_epoch_info,
            &validator_roles,
            use_stable_shard_assignment,
        )?
    } else if checked_feature!("stable", ChunkOnlyProducers, protocol_version) {
        old_validator_selection::assign_chunk_producers_to_shards_chunk_only(
            epoch_config,
            validator_roles.chunk_producers,
            &validator_roles.block_producers,
        )?
    } else {
        old_validator_selection::assign_chunk_producers_to_shards(
            epoch_config,
            validator_roles.chunk_producers,
            &validator_roles.block_producers,
        )?
    };
	
  
  let validator_mandates = if ProtocolFeature::StatelessValidation.enabled(protocol_version) {
       
        let target_mandates_per_shard = epoch_config.target_validator_mandates_per_shard as usize;
        let num_shards = shard_ids.len();
        let validator_mandates_config =
        ValidatorMandatesConfig::new(target_mandates_per_shard, num_shards);
      
        ValidatorMandates::new(validator_mandates_config, &all_validators)
    } else {
        ValidatorMandates::default()
    };
}
```

```rust
fn get_chunk_producers_assignment(
    epoch_config: &EpochConfig,
    rng_seed: RngSeed,
    prev_epoch_info: &EpochInfo,
    validator_roles: &ValidatorRoles,
    use_stable_shard_assignment: bool,
) -> Result<ChunkProducersAssignment, EpochError> {
  ... do something
  
  
  let shard_assignment = assign_chunk_producers_to_shards(
        chunk_producers.clone(),
        shard_ids.len() as NumShards,
        minimum_validators_per_shard,
        epoch_config.validator_selection_config.chunk_producer_assignment_changes_limit as usize,
        rng_seed,
        prev_chunk_producers_assignment,
    )
    .map_err(|_| EpochError::NotEnoughValidators {
        num_validators: num_chunk_producers as u64,
        num_shards: shard_ids.len() as NumShards,
    })?;
  
  ...do something...
}
```

```rust
pub(crate) fn assign_chunk_producers_to_shards(
    chunk_producers: Vec<ValidatorStake>,
    num_shards: NumShards,
    min_validators_per_shard: usize,
    shard_assignment_changes_limit: usize,
    rng_seed: RngSeed,
    prev_chunk_producers_assignment: Option<Vec<Vec<ValidatorStake>>>,
) -> Result<Vec<Vec<ValidatorStake>>, NotEnoughValidators> {
    ... do some thing ...

    let result = if chunk_producers.len() < min_validators_per_shard * (num_shards as usize) {
        assign_to_satisfy_shards(chunk_producers, num_shards, min_validators_per_shard)
    } else {
        assign_to_balance_shards(
            chunk_producers,
            num_shards,
            min_validators_per_shard,
            shard_assignment_changes_limit,
            rng_seed,
            prev_chunk_producers_assignment,
        )
    };
    Ok(result)
}
```

### 分配方式

质押量

- **核心依据**: 质押量是验证者参与网络的“投入”，质押量越大，对网络的贡献就越大，因此在分配时往往会给予更高的优先级。
- **影响**: 质押量高的验证者更有可能被分配到更多的 Shard，或者在 Shard 内获得更高的权重。

最小验证者数量

- **保障网络安全**: 每个 Shard 必须保证有足够的验证者，以确保网络的安全性。
- **影响**: 在分配时，会优先考虑那些验证者数量较少的 Shard，以满足最小验证者数量的要求。

历史分配

- **稳定性**: 考虑上一个 epoch 的分配情况，可以减少验证者频繁切换 Shard，提高网络的稳定性。
- **影响**: 在新的分配中，会尽量保持验证者的 Shard 分配不变，除非有充分的理由进行调整。

随机性

- **公平性**: 引入随机性可以避免固定的分配模式，增加分配的公平性。
- **影响**: 在满足其他条件的情况下，随机因素会对最终的分配结果产生一定的影响。

负载均衡

- **网络性能**: 通过合理分配验证者，可以平衡各个 Shard 的负载，提高网络的整体性能。
- **影响**: 分配算法会尽量将验证者均匀地分配到不同的 Shard，避免出现负载不均衡的情况。

其他因素

- **网络拓扑**: 可能会考虑网络拓扑结构，将验证者分配到与其物理位置接近的 Shard，以降低网络延迟。
- **验证者行为**: 对于行为不良的验证者，可能会降低其分配优先级。

### 分片状态管理

每个分片维护自己独立的状态，包括账户余额、合约代码和存储：

```rust
pub struct ShardTrie {
    pub trie: Trie,
    pub shard_id: ShardId,
    pub state_root: StateRoot,
}

impl ShardTrie {
    pub fn get_account(&self, account_id: &AccountId) -> Result<Option<Account>, StorageError> {
        let key = TrieKey::Account { account_id: account_id.clone() };
        match self.trie.get(&key.to_vec())? {
            Some(data) => Ok(Some(Account::try_from_slice(&data)?)),
            None => Ok(None),
        }
    }
    
    pub fn set_account(&mut self, account_id: &AccountId, account: &Account) -> Result<(), StorageError> {
        let key = TrieKey::Account { account_id: account_id.clone() };
        self.trie.set(key.to_vec(), account.try_to_vec()?);
        Ok(())
    }
    
    pub fn apply_state_changes(&mut self, changes: Vec<StateChange>) -> Result<StateRoot, StorageError> {
        for change in changes {
            match change {
                StateChange::AccountUpdate { account_id, account } => {
                    self.set_account(&account_id, &account)?;
                }
                StateChange::AccountDeletion { account_id } => {
                    let key = TrieKey::Account { account_id };
                    self.trie.delete(key.to_vec());
                }
                StateChange::ContractCodeUpdate { account_id, code } => {
                    let key = TrieKey::ContractCode { account_id };
                    self.trie.set(key.to_vec(), code);
                }
                // 处理其他状态变更...
            }
        }
        
        // 计算新的状态根
        self.state_root = self.trie.get_root();
        Ok(self.state_root)
    }
}
```

### 跨分片交易处理

当交易涉及不同分片的账户时，需要通过Receipt机制进行跨分片通信：

```rust
pub struct CrossShardTransaction {
    pub from_shard: ShardId,
    pub to_shard: ShardId,
    pub receipt: Receipt,
}

impl ShardManager {
    pub fn process_cross_shard_transaction(
        &mut self,
        transaction: SignedTransaction,
    ) -> Result<Vec<Receipt>, Error> {
        let signer_shard = self.account_id_to_shard_id(&transaction.transaction.signer_id);
        let receiver_shard = self.account_id_to_shard_id(&transaction.transaction.receiver_id);
        
        if signer_shard == receiver_shard {
            // 同分片交易，直接处理
            self.process_local_transaction(transaction, signer_shard)
        } else {
            // 跨分片交易，生成Receipt
            let receipt = self.transaction_to_receipt(transaction)?;
            
            // 在发送方分片执行扣费等操作
            self.execute_sender_actions(&receipt, signer_shard)?;
            
            // 生成发送到接收方分片的Receipt
            Ok(vec![receipt])
        }
    }
    
    fn account_id_to_shard_id(&self, account_id: &AccountId) -> ShardId {
        // 使用一致性哈希将账户ID映射到分片
        let hash = CryptoHash::hash_bytes(account_id.as_bytes());
        let shard_index = u64::from_le_bytes(hash.as_ref()[0..8].try_into().unwrap()) 
            % self.num_shards as u64;
        shard_index as ShardId
    }
}
```

### 分片负载均衡

NEAR通过动态调整分片配置来实现负载均衡：

```rust
pub struct ShardLoadBalancer {
    pub shard_metrics: HashMap<ShardId, ShardMetrics>,
    pub target_tps_per_shard: u64,
    pub max_gas_per_shard: Gas,
}

#[derive(Debug, Clone)]
pub struct ShardMetrics {
    pub transactions_per_second: f64,
    pub gas_usage: Gas,
    pub storage_usage: u64,
    pub validator_count: usize,
    pub average_latency: Duration,
}

impl ShardLoadBalancer {
    pub fn should_split_shard(&self, shard_id: ShardId) -> bool {
        if let Some(metrics) = self.shard_metrics.get(&shard_id) {
            metrics.transactions_per_second > self.target_tps_per_shard as f64 * 1.5
                || metrics.gas_usage > self.max_gas_per_shard
        } else {
            false
        }
    }
    
    pub fn should_merge_shards(&self, shard1: ShardId, shard2: ShardId) -> bool {
        let metrics1 = self.shard_metrics.get(&shard1);
        let metrics2 = self.shard_metrics.get(&shard2);
        
        if let (Some(m1), Some(m2)) = (metrics1, metrics2) {
            let combined_tps = m1.transactions_per_second + m2.transactions_per_second;
            let combined_gas = m1.gas_usage + m2.gas_usage;
            
            combined_tps < self.target_tps_per_shard as f64 * 0.5
                && combined_gas < self.max_gas_per_shard
        } else {
            false
        }
    }
    
    pub fn rebalance_shards(&mut self, epoch_config: &EpochConfig) -> Vec<ShardRebalanceAction> {
        let mut actions = Vec::new();
        
        // 检查是否需要分片拆分
        for (&shard_id, _) in &self.shard_metrics {
            if self.should_split_shard(shard_id) {
                actions.push(ShardRebalanceAction::Split { shard_id });
            }
        }
        
        // 检查是否需要分片合并
        let shard_ids: Vec<_> = self.shard_metrics.keys().cloned().collect();
        for i in 0..shard_ids.len() {
            for j in i+1..shard_ids.len() {
                if self.should_merge_shards(shard_ids[i], shard_ids[j]) {
                    actions.push(ShardRebalanceAction::Merge { 
                        shard1: shard_ids[i], 
                        shard2: shard_ids[j] 
                    });
                }
            }
        }
        
        actions
    }
}

#[derive(Debug, Clone)]
pub enum ShardRebalanceAction {
    Split { shard_id: ShardId },
    Merge { shard1: ShardId, shard2: ShardId },
    Redistribute { from_shard: ShardId, to_shard: ShardId, accounts: Vec<AccountId> },
}
```

### 分片同步机制

分片之间需要保持状态同步，特别是在处理跨分片交易时：

```rust
pub struct ShardSynchronizer {
    pub shard_id: ShardId,
    pub peer_shards: HashMap<ShardId, ShardPeerInfo>,
    pub pending_receipts: HashMap<ShardId, Vec<Receipt>>,
}

impl ShardSynchronizer {
    pub fn sync_with_peer_shards(&mut self) -> Result<(), Error> {
        for (&peer_shard_id, peer_info) in &self.peer_shards {
            // 同步待处理的Receipts
            if let Some(receipts) = self.pending_receipts.get(&peer_shard_id) {
                for receipt in receipts {
                    self.send_receipt_to_shard(receipt.clone(), peer_shard_id)?;
                }
            }
            
            // 请求对方分片的状态更新
            self.request_state_update_from_shard(peer_shard_id)?;
        }
        
        Ok(())
    }
    
    fn send_receipt_to_shard(&self, receipt: Receipt, target_shard: ShardId) -> Result<(), Error> {
        // 通过网络发送Receipt到目标分片
        // 实际实现会通过P2P网络进行通信
        Ok(())
    }
    
    fn request_state_update_from_shard(&self, shard_id: ShardId) -> Result<(), Error> {
        // 请求特定分片的状态更新
        Ok(())
    }
}
```

### 分片安全性保证

每个分片都有独立的安全机制：

```rust
pub struct ShardSecurity {
    pub validators: Vec<ValidatorStake>,
    pub min_validators: usize,
    pub byzantine_threshold: f64, // 通常是 1/3
}

impl ShardSecurity {
    pub fn is_secure(&self) -> bool {
        self.validators.len() >= self.min_validators
            && self.has_sufficient_stake_distribution()
    }
    
    fn has_sufficient_stake_distribution(&self) -> bool {
        let total_stake: Balance = self.validators.iter().map(|v| v.stake()).sum();
        let max_validator_stake = self.validators.iter().map(|v| v.stake()).max().unwrap_or(0);
        
        // 确保没有单个验证者控制超过1/3的质押
        (max_validator_stake as f64) < (total_stake as f64 * self.byzantine_threshold)
    }
    
    pub fn can_tolerate_failures(&self, failed_validators: usize) -> bool {
        let remaining_validators = self.validators.len() - failed_validators;
        remaining_validators >= self.min_validators
    }
}

