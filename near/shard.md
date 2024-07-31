## Shard

near中分片的概念，在near 中整个网络被分为若干的分片，他们各自维护自己的状态。分片之间可以进行跨分片通信。

先说分片上验证者的分配，还是回顾一下之前提到的epochmanager，在每个epoch开始/结束的时候都会重新做一些事情，其中就包含了验证者分配，分配到不同的分片上去

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

