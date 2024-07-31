## Staking  

这里主要是记录staking reward的计算方式的

在之前的epoch 中提到过每个epoch结束的时候会同意进行reward/slash的计算

```rust
fn finalize_epoch(
        &mut self,
        store_update: &mut StoreUpdate,
        block_info: &BlockInfo,
        last_block_hash: &CryptoHash,
        rng_seed: RngSeed,
    ) -> Result<(), EpochError> {
      ... do something ...
      
      let (validator_reward, minted_amount) = {
            let last_epoch_last_block_hash =
                *self.get_block_info(block_info.epoch_first_block())?.prev_hash();
            let last_block_in_last_epoch = self.get_block_info(&last_epoch_last_block_hash)?;
            assert!(block_info.timestamp_nanosec() > last_block_in_last_epoch.timestamp_nanosec());
            let epoch_duration =
                block_info.timestamp_nanosec() - last_block_in_last_epoch.timestamp_nanosec();
            for (account_id, reason) in validator_kickout.iter() {
                if matches!(
                    reason,
                    ValidatorKickoutReason::NotEnoughBlocks { .. }
                        | ValidatorKickoutReason::NotEnoughChunks { .. }
                        | ValidatorKickoutReason::NotEnoughChunkEndorsements { .. }
                ) {
                    validator_block_chunk_stats.remove(account_id);
                }
            }
            self.reward_calculator.calculate_reward(
                validator_block_chunk_stats,
                &validator_stake,
                *block_info.total_supply(),
                epoch_protocol_version,
                self.genesis_protocol_version,
                epoch_duration,
            )
        };
      
      ... update epoch info ...
      
}
```

列出的这部分代码实际上是计算验证的奖励，对于在当前 epoch 中未能达到预期的验证者，将其从验证者集中移除。

```rust
pub fn calculate_reward(
        &self,
        validator_block_chunk_stats: HashMap<AccountId, BlockChunkValidatorStats>,
        validator_stake: &HashMap<AccountId, Balance>,
        total_supply: Balance,
        protocol_version: ProtocolVersion,
        genesis_protocol_version: ProtocolVersion,
        epoch_duration: u64,
    ) -> (HashMap<AccountId, Balance>, Balance) {
        let mut res = HashMap::new();
        let num_validators = validator_block_chunk_stats.len();
        let use_hardcoded_value = genesis_protocol_version < protocol_version
            && protocol_version >= ENABLE_INFLATION_PROTOCOL_VERSION;
        let max_inflation_rate =
            if use_hardcoded_value { Rational32::new_raw(1, 20) } else { self.max_inflation_rate };
        let protocol_reward_rate = if use_hardcoded_value {
            Rational32::new_raw(1, 10)
        } else {
            self.protocol_reward_rate
        };
        let epoch_total_reward: u128 =
            if checked_feature!("stable", RectifyInflation, protocol_version) {
                (U256::from(*max_inflation_rate.numer() as u64)
                    * U256::from(total_supply)
                    * U256::from(epoch_duration)
                    / (U256::from(self.num_seconds_per_year)
                        * U256::from(*max_inflation_rate.denom() as u64)
                        * U256::from(NUM_NS_IN_SECOND)))
                .as_u128()
            } else {
                (U256::from(*max_inflation_rate.numer() as u64)
                    * U256::from(total_supply)
                    * U256::from(self.epoch_length)
                    / (U256::from(self.num_blocks_per_year)
                        * U256::from(*max_inflation_rate.denom() as u64)))
                .as_u128()
            };
        let epoch_protocol_treasury = (U256::from(epoch_total_reward)
            * U256::from(*protocol_reward_rate.numer() as u64)
            / U256::from(*protocol_reward_rate.denom() as u64))
        .as_u128();
        res.insert(self.protocol_treasury_account.clone(), epoch_protocol_treasury);
        if num_validators == 0 {
            return (res, 0);
        }
        let epoch_validator_reward = epoch_total_reward - epoch_protocol_treasury;
        let mut epoch_actual_reward = epoch_protocol_treasury;
        let total_stake: Balance = validator_stake.values().sum();
        for (account_id, stats) in validator_block_chunk_stats {
            // Uptime is an average of block produced / expected, chunk produced / expected,
            // and chunk endorsed produced / expected.

            let expected_blocks = stats.block_stats.expected;
            let expected_chunks = stats.chunk_stats.expected();
            let expected_endorsements = stats.chunk_stats.endorsement_stats().expected;

            let (average_produced_numer, average_produced_denom) =
                match (expected_blocks, expected_chunks, expected_endorsements) {
                    // Validator was not expected to do anything
                    (0, 0, 0) => (U256::from(0), U256::from(1)),
                    // Validator was a stateless validator only (not expected to produce anything)
                    (0, 0, expected_endorsements) => {
                        let endorsement_stats = stats.chunk_stats.endorsement_stats();
                        (U256::from(endorsement_stats.produced), U256::from(expected_endorsements))
                    }
                    // Validator was a chunk-only producer
                    (0, expected_chunks, 0) => {
                        (U256::from(stats.chunk_stats.produced()), U256::from(expected_chunks))
                    }
                    // Validator was only a block producer
                    (expected_blocks, 0, 0) => {
                        (U256::from(stats.block_stats.produced), U256::from(expected_blocks))
                    }
                    // Validator produced blocks and chunks, but not endorsements
                    (expected_blocks, expected_chunks, 0) => {
                        let numer = U256::from(
                            stats.block_stats.produced * expected_chunks
                                + stats.chunk_stats.produced() * expected_blocks,
                        );
                        let denom = U256::from(2 * expected_chunks * expected_blocks);
                        (numer, denom)
                    }
                    // Validator produced chunks and endorsements, but not blocks
                    (0, expected_chunks, expected_endorsements) => {
                        let endorsement_stats = stats.chunk_stats.endorsement_stats();
                        let numer = U256::from(
                            endorsement_stats.produced * expected_chunks
                                + stats.chunk_stats.produced() * expected_endorsements,
                        );
                        let denom = U256::from(2 * expected_chunks * expected_endorsements);
                        (numer, denom)
                    }
                    // Validator produced blocks and endorsements, but not chunks
                    (expected_blocks, 0, expected_endorsements) => {
                        let endorsement_stats = stats.chunk_stats.endorsement_stats();
                        let numer = U256::from(
                            endorsement_stats.produced * expected_blocks
                                + stats.block_stats.produced * expected_endorsements,
                        );
                        let denom = U256::from(2 * expected_blocks * expected_endorsements);
                        (numer, denom)
                    }
                    // Validator did all the things
                    (expected_blocks, expected_chunks, expected_endorsements) => {
                        let produced_blocks = stats.block_stats.produced;
                        let produced_chunks = stats.chunk_stats.produced();
                        let produced_endorsements = stats.chunk_stats.endorsement_stats().produced;

                        let numer = U256::from(
                            produced_blocks * expected_chunks * expected_endorsements
                                + produced_chunks * expected_blocks * expected_endorsements
                                + produced_endorsements * expected_blocks * expected_chunks,
                        );
                        let denom = U256::from(
                            3 * expected_chunks * expected_blocks * expected_endorsements,
                        );
                        (numer, denom)
                    }
                };

            let online_min_numer = U256::from(*self.online_min_threshold.numer() as u64);
            let online_min_denom = U256::from(*self.online_min_threshold.denom() as u64);
            // If average of produced blocks below online min threshold, validator gets 0 reward.
            let chunk_only_producers_enabled =
                checked_feature!("stable", ChunkOnlyProducers, protocol_version);
            let reward = if average_produced_numer * online_min_denom
                < online_min_numer * average_produced_denom
                || (chunk_only_producers_enabled
                    && expected_chunks == 0
                    && expected_blocks == 0
                    && expected_endorsements == 0)
                // This is for backwards compatibility. In 2021 December, after we changed to 4 shards,
                // mainnet was ran without SynchronizeBlockChunkProduction for some time and it's
                // possible that some validators have expected blocks or chunks to be zero.
                || (!chunk_only_producers_enabled
                    && (expected_chunks == 0 || expected_blocks == 0))
            {
                0
            } else {
                let stake = *validator_stake
                    .get(&account_id)
                    .unwrap_or_else(|| panic!("{} is not a validator", account_id));
                // Online reward multiplier is min(1., (uptime - online_threshold_min) / (online_threshold_max - online_threshold_min).
                let online_max_numer = U256::from(*self.online_max_threshold.numer() as u64);
                let online_max_denom = U256::from(*self.online_max_threshold.denom() as u64);
                let online_numer =
                    online_max_numer * online_min_denom - online_min_numer * online_max_denom;
                let mut uptime_numer = (average_produced_numer * online_min_denom
                    - online_min_numer * average_produced_denom)
                    * online_max_denom;
                let uptime_denum = online_numer * average_produced_denom;
                // Apply min between 1. and computed uptime.
                uptime_numer =
                    if uptime_numer > uptime_denum { uptime_denum } else { uptime_numer };
                (U512::from(epoch_validator_reward) * U512::from(uptime_numer) * U512::from(stake)
                    / U512::from(uptime_denum)
                    / U512::from(total_stake))
                .as_u128()
            };
            res.insert(account_id, reward);
            epoch_actual_reward += reward;
        }
        (res, epoch_actual_reward)
    }
```

从这段代码中可以总结出以下公式

### epoch_total_reward

分为两个不同的计算方式，取决于`RectifyInflation` (和通胀有关)：

1. 启用 `RectifyInflation` 
   $$
   {epoch\_total\_reward} = \frac{\text{max\_inflation\_rate.numer} \times \text{total\_supply} \times \text{epoch\_duration}}{\text{num\_seconds\_per\_year} \times \text{max\_inflation\_rate.denom} \times \text{NUM\_NS\_IN\_SECOND}}
   $$
   

2. 未启用 
   $$
   {epoch\_total\_reward} = \frac{\text{max\_inflation\_rate.numer} \times \text{total\_supply} \times \text{epoch\_length}}{\text{num\_blocks\_per\_year} \times \text{max\_inflation\_rate.denom}}
   $$

   1. **max_inflation_rate.numer** 与 **max_inflation_rate.denom** 最大通胀率的分子和分母。通胀率决定了每年新生成的代币数量。
      1. 如果最大通胀率是 5%，那么 `max_inflation_rate.numer` 是 5，`max_inflation_rate.denom` 是 100。
   2. **total_supply** 当前的 NEAR 代币总供应量。它表示所有已发行的代币总数。 
   3. **epoch_duration** 和 **epoch_length**
      1. 这是一个 epoch 的持续时间。`epoch_duration` 通常以秒为单位，而 `epoch_length` 以区块数为单位。
   4.  **num_seconds_per_year** 和 **num_blocks_per_year**：
      1. `num_seconds_per_year` 是一年中的总秒数（通常是 31,536,000 秒），而 `num_blocks_per_year` 是一年中生成的区块总数。
   5. **NUM_NS_IN_SECOND**: 这是每秒的纳秒数，通常是 1,000,000,000 纳秒。



### epoch_protocol_treasury

**协议奖励**,在每个 epoch 期间分配给协议本身的奖励部分。具体来说，它是从 `epoch_total_reward` 中提取的一部分，用于支持协议的持续发展和维护。
$$
{epoch\_protocol\_treasury} = \frac{\text{epoch\_total\_reward} \times \text{protocol\_reward\_rate.numer}}{\text{protocol\_reward\_rate.denom}}
$$

1. **protocol_reward_rate.numer** 和 **protocol_reward_rate.denom**：
   - 这是协议奖励率的分子和分母。它决定了总奖励中有多少分配给协议本身。

### 验证者奖励计算公式

验证者总奖励
$$
epoch\_validator\_reward=epoch\_total\_reward−epoch\_protocol\_treasury
$$
对于每个验证者计算其实际的奖励

计算uptime，实际上就是验证者在epoch期间的活跃度，如果验证者在预期的时间内成功生产了区块、分片或背书，那么其 `uptime` 就会较高。
$$
uptime= 
\frac {
average\_produced\_numer×online\_min\_denom−online\_min\_numer×average\_produced\_denom} {\text online\_time}
$$
其中
$$
average\_produced\_numer=produced\_blocks×expected\_chunks×expected\_endorsements+produced\_chunks×expected\_blocks×expected\_endorsements+produced\_endorsements×expected\_blocks×expected\_chunks
$$

$$
average\_produced\_denom=3×expected\_chunks×expected\_blocks×expected\_endorsements
$$



计算验证者的奖励
$$
{reward} = \min\left(1, \frac{\text{uptime\_numer}}{\text{uptime\_denom}}\right) \times \frac{\text{epoch\_validator\_reward} \times \text{stake}}{\text{total\_stake}}
$$
其中：

`uptime_numer` 和 `uptime_denom` 分别表示验证者的实际生产率和期望生产率。公式前半段可以看做是计算奖励系数。
$$
{uptime\_numer} = (\text{average\_produced\_numer} \times \text{online\_min\_denom} - \text{online\_min\_numer} \times \text{average\_produced\_denom}) \times \text{online\_max\_denom}
$$

$$
{uptime\_denum} = (\text{online\_max\_numer} \times \text{online\_min\_denom} - \text{online\_min\_numer} \times \text{online\_max\_denom}) \times \text{average\_produced\_denom}
$$













