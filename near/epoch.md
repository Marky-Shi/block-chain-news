# Epoch

在NEAR Protocol中，epoch是一个基本的时间单位，理想情况下为12小时。Epoch作为网络运行的重要周期，承担着多种关键功能。

## Epoch的基本概念

- **定义**：Epoch是NEAR区块链中的固定时间段，通常设计为12小时
- **区块数量**：每个epoch包含约43,200个区块（假设区块生成时间为1秒）
- **目的**：提供网络状态更新、验证者轮换和奖励分配的时间边界

## Epoch的主要功能

### 验证器轮换
- 每个Epoch结束时，网络会重新选择和分配验证器
- 这是通过质押量和随机性机制决定的，以确保验证器的公平分配和网络的安全性
- 验证者选择过程在当前epoch结束前就已确定，为下一个epoch做准备

### 质押和奖励结算
- 在每个Epoch结束时，网络会结算验证器和质押者的奖励
- 根据验证器的表现（如成功打包区块的数量、在线时间）和质押者的质押量，分配奖励和罚款
- 奖励包括区块奖励和交易费用的分配

### 状态检查点
- Epoch作为时间边界，有助于网络进行状态检查点的创建和维护
- 状态检查点确保网络的状态可以在需要时恢复，提供数据完整性和安全性
- 这些检查点对于网络的长期稳定性和可靠性至关重要

### 治理和升级
- Epoch结束时也是网络进行治理和升级的机会
- 提案和变更可以在Epoch之间进行评估和部署，以确保网络的稳定和持续改进
- 协议升级通常在epoch边界执行，以减少网络中断

### Epoch结束时的处理流程

每个epoch结束时，NEAR协议会执行以下关键步骤：

1. **收集和处理信息**：
   * 收集当前epoch的区块生产数据
   * 处理验证者的表现数据（出块率、在线时间等）
   * 统计网络活动和资源使用情况

2. **奖励计算与分配**：
   * 计算验证者奖励和铸币量
   * 根据质押比例分配奖励给验证者和委托人
   * 处理可能的惩罚（如双签或长时间离线）

3. **分片配置更新**：
   * 确定下一个和下下个epoch的分片布局
   * 更新分片配置信息，包括验证者分配
   * 优化分片结构以平衡网络负载

4. **未来epoch准备**：
   * 生成和保存下下个epoch的信息
   * 准备随机种子用于验证者选择
   * 更新网络参数和协议状态

### Epoch的技术实现

#### EpochInfo结构
```rust
pub struct EpochInfo {
    pub epoch_height: EpochHeight,
    pub validators: Vec<ValidatorStake>,
    pub fishermen: Vec<ValidatorStake>,
    pub stake_change: BTreeMap<AccountId, Balance>,
    pub validator_kickout: HashMap<AccountId, ValidatorKickoutReason>,
    pub validator_reward: HashMap<AccountId, Balance>,
    pub minted_amount: Balance,
    pub seat_price: Balance,
    pub protocol_version: ProtocolVersion,
}
```

#### Epoch转换的核心函数
```rust
fn finalize_epoch(
    &mut self,
    store_update: &mut StoreUpdate,
    block_info: &BlockInfo,
    last_block_hash: &CryptoHash,
    rng_seed: RngSeed,
) -> Result<(), EpochError> {
    // 计算epoch持续时间
    let epoch_duration = block_info.timestamp_nanosec() - last_block_in_last_epoch.timestamp_nanosec();
    
    // 计算验证者奖励
    let (validator_reward, minted_amount) = self.reward_calculator.calculate_reward(
        validator_block_chunk_stats,
        &validator_stake,
        *block_info.total_supply(),
        epoch_protocol_version,
        self.genesis_protocol_version,
        epoch_duration,
    );
    
    // 处理验证者踢出
    for (account_id, reason) in validator_kickout.iter() {
        // 移除表现不佳的验证者
    }
    
    // 更新epoch信息
    // ...
}
```

#### Epoch边界的关键操作

1. **验证者选择算法**：
```rust
fn select_validators(
    mut proposals: BinaryHeap<OrderedValidatorStake>,
    max_number_selected: usize,
    min_stake_ratio: Ratio<u128>,
    protocol_version: ProtocolVersion,
) -> (Vec<ValidatorStake>, BinaryHeap<OrderedValidatorStake>, Balance) {
    // 基于质押量和随机性选择验证者
    // 确保最小质押比例要求
    // 返回选中的验证者列表
}
```

2. **分片重新分配**：
   * 每个epoch结束时重新计算分片分配
   * 基于验证者数量和网络负载动态调整分片数量
   * 确保分片间的负载均衡

3. **状态快照**：
   * 在epoch边界创建状态快照
   * 用于快速同步和状态恢复
   * 提供数据完整性验证

### Epoch参数配置

* **EPOCH_LENGTH**: 43,200个区块（约12小时）
* **MIN_VALIDATORS_PER_SHARD**: 每个分片最少验证者数量
* **MAX_VALIDATORS**: 网络最大验证者数量
* **SEAT_PRICE_UPDATE_RATIO**: 座位价格更新比例

### Epoch与分片的协调

每个epoch转换时，NEAR协议需要：

1. **重新分配验证者到分片**：
   * 确保每个分片有足够的验证者
   * 平衡分片间的质押分布
   * 维护网络的去中心化程度

2. **跨分片状态同步**：
   * 处理跨分片交易的最终确认
   * 更新分片间的状态根
   * 确保全局状态一致性

3. **分片配置更新**：
   * 根据网络活动调整分片数量
   * 优化分片大小和验证者分配
   * 预测下一个epoch的资源需求

### Epoch与网络安全

* **终结性保证**：Epoch边界提供了额外的区块终结性保证
* **攻击防御**：通过定期轮换验证者，减少长期控制攻击的风险
* **随机性引入**：每个epoch引入新的随机性，增强网络的不可预测性和安全性
* **长程攻击防护**：Epoch检查点防止历史重写攻击

### Epoch性能优化

* **预计算**：提前计算下一个epoch的验证者分配
* **并行处理**：epoch转换过程中的并行计算
* **缓存机制**：缓存频繁访问的epoch信息
* **增量更新**：只更新变化的部分，减少计算开销

