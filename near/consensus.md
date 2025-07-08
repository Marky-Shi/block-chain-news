## Consensus 

Nearcore. 9932e95c5c49262189bc4737366ce46bb2b953c8

NEAR 的共识算法 PoS

在之前已经研究过了 NEAR 一笔交易的生命周期，其中涉及到一点区块构建的内容，本节主要是研究 NEAR 的共识过程，展示这个系统是如何工作的，里边会涉及到分片管理的一些知识。

共识的过程其实就是全网参与选举出块节点、产生新区块，执行批交易、进行状态更改的过程。

NEAR 采用的是改进的 PoS 共识机制，结合了分片技术和快速终结性保证，主要特点包括：

* **Nightshade 共识**：NEAR 的分片共识协议
* **Doomslug**：快速终结性机制，提供 1-2 秒的区块确认时间
* **分片并行处理**：多个分片可以并行处理交易
* **跨分片通信**：通过 Receipt 机制实现分片间的异步通信

### 节点选举

```rust
fn select_validators(
    mut proposals: BinaryHeap<OrderedValidatorStake>,
    max_number_selected: usize,
    min_stake_ratio: Ratio<u128>,
    protocol_version: ProtocolVersion,
) -> (Vec<ValidatorStake>, BinaryHeap<OrderedValidatorStake>, Balance) {
    let mut total_stake = 0;
    let n = cmp::min(max_number_selected, proposals.len());
    let mut validators = Vec::with_capacity(n);
    for _ in 0..n {
        let p = proposals.pop().unwrap().0;
        let p_stake = p.stake();
        let total_stake_with_p = total_stake + p_stake;
        if Ratio::new(p_stake, total_stake_with_p) > min_stake_ratio {
            validators.push(p);
            total_stake = total_stake_with_p;
        } else {
            // p was not included, return it to the list of proposals
            proposals.push(OrderedValidatorStake(p));
            break;
        }
    }
    if validators.len() == max_number_selected {
        // all slots were filled, so the threshold stake is 1 more than the current
        // smallest stake
        let threshold = validators.last().unwrap().stake() + 1;
        (validators, proposals, threshold)
    } else {
        // the stake ratio condition prevented all slots from being filled,
        // or there were fewer proposals than available slots,
        // so the threshold stake is whatever amount pass the stake ratio condition
        let threshold = if checked_feature!(
            "protocol_feature_fix_staking_threshold",
            FixStakingThreshold,
            protocol_version
        ) {
            (min_stake_ratio * Ratio::from_integer(total_stake)
                / (Ratio::from_integer(1u128) - min_stake_ratio))
                .ceil()
                .to_integer()
        } else {
            (min_stake_ratio * Ratio::new(total_stake, 1)).ceil().to_integer()
        };
        (validators, proposals, threshold)
    }
}
```

根据质押比例和数量限制，从一组候选人中选取验证者的逻辑。通过检查质押比例和总质押量，它确保选中的验证者具有足够的质押量，同时返回未选中的提案和阈值质押金额。

而这个方法调用的地方有两个，第一个是epoch构建的时候调用这个方法用来确定这个epoch中的出块顺序，另一个就是Finalizes epoch epoch结束时。

epoch 结束时要做的事情：

1. 收集和处理区块和验证者信息。
2. 计算验证者奖励和铸币量。
3. 确定下一个和下下个 epoch 的分片布局和配置信息。
4. 生成和保存下下个 epoch 的信息。

### doomslug consens

Doomslug主要负责区块的快速确认，而分片则负责提升网络的可扩展性和性能

该模块的启动代码

```rust
pub fn start(&mut self, ctx: &mut dyn DelayedActionRunner<Self>) {
        self.start_flat_storage_creation(ctx);
        // Start syncing job.
        self.start_sync(ctx);

        // Start triggers
  			// 一些定任务负责提出区块提案并处理，确定每个epoch的出块顺序。
        self.schedule_triggers(ctx);

        // Start catchup job.
  			// 如果要是验证节点，则需要同步历史数据
        self.catchup(ctx);

        if let Err(err) = self.client.send_network_chain_info() {
            tracing::error!(target: "client", ?err, "Failed to update network chain info");
        }
    }
```

```rust
pub fn send_block_approval(
        &mut self,
        parent_hash: &CryptoHash,
        approval: Approval,
        signer: &Option<Arc<ValidatorSigner>>,
    ) -> Result<(), Error> {
        let next_epoch_id = self.epoch_manager.get_epoch_id_from_prev_block(parent_hash)?;
        let next_block_producer =
            self.epoch_manager.get_block_producer(&next_epoch_id, approval.target_height)?;
        let next_block_producer_id = signer.as_ref().map(|x| x.validator_id());
        if Some(&next_block_producer) == next_block_producer_id {
            self.collect_block_approval(&approval, ApprovalType::SelfApproval, signer);
        } else {
            debug!(target: "client",
                approval_inner = ?approval.inner,
                account_id = ?approval.account_id,
                next_bp = ?next_block_producer,
                target_height = approval.target_height,
                approval_type="PeerApproval",
                "send_block_approval");
            let approval_message = ApprovalMessage::new(approval, next_block_producer);
            self.network_adapter.send(PeerManagerMessageRequest::NetworkRequests(
                NetworkRequests::Approval { approval_message },
            ));
        }

        Ok(())
    }
```

这个方法区块链系统中负责传播区块批准信息的关键环节，它确保区块批准信息能够及时地传递给其他节点，从而参与共识过程。

### 产生新区块

首先需要明白near 区块的结构

```rust
#[derive(BorshSerialize, BorshDeserialize, Debug, Clone, Eq, PartialEq)]
pub struct BlockV1 {
    pub header: BlockHeader,
    pub chunks: Vec<ShardChunkHeaderV1>,
    pub challenges: Challenges,
    // Data to confirm the correctness of randomness beacon output
    pub vrf_value: near_crypto::vrf::Value,
    pub vrf_proof: near_crypto::vrf::Proof,
}

#[derive(BorshSerialize, BorshDeserialize, Debug, Clone, Eq, PartialEq)]
pub struct BlockV2 {
    pub header: BlockHeader,
    pub chunks: Vec<ShardChunkHeader>,
    pub challenges: Challenges,
    // Data to confirm the correctness of randomness beacon output
    pub vrf_value: near_crypto::vrf::Value,
    pub vrf_proof: near_crypto::vrf::Proof,
}

/// V2 -> V3: added BlockBodyV1
#[derive(BorshSerialize, BorshDeserialize, Debug, Clone, Eq, PartialEq)]
pub struct BlockV3 {
    pub header: BlockHeader,
    pub body: BlockBodyV1,
}

/// V3 -> V4: use versioned BlockBody
#[derive(BorshSerialize, BorshDeserialize, Debug, Clone, Eq, PartialEq)]
pub struct BlockV4 {
    pub header: BlockHeader,
    pub body: BlockBody,
}
```

从区块结构大致能看出来V1----> V2 中区块除了包含区块头，chunk的信息包含交易等重要的信息，challenges 则是包含对于该区块的挑战信息， vrf-vaule、vrf-proof 

从v3开始就把区块分成了 blockheader blackbody 

#### blockbody

```rust
pub struct BlockBodyV1 {
    pub chunks: Vec<ShardChunkHeader>,
    pub challenges: Challenges,

    // Data to confirm the correctness of randomness beacon output
    pub vrf_value: Value,
    pub vrf_proof: Proof,
}
```

而v3---》v4 blockbody的改变则是新增了一个属性

```rust
/*
区块认可
这些结构为来自 fn get_ordered_chunk_validators 获得的每个分片的所有有序区块验证器的签名向量。chunk_endorsements[shard_id][chunk_validator_index] 是签名（如果存在）。如果区块验证器未认可该区块，则签名为 None。
对于缺失区块的情况，会像对待区块一样，包括来自前一个区块的区块认可。
*/
pub chunk_endorsements: Vec<ChunkEndorsementSignatures>,
```

#### blockheader

```rust
pub struct BlockHeaderV1 {
    pub prev_hash: CryptoHash,

    // 返回给清客户端的信息
    pub inner_lite: BlockHeaderInnerLite,
    pub inner_rest: BlockHeaderInnerRest,

    /// Signature of the block producer.
    pub signature: Signature,

    /// Cached value of hash for this block.
    #[borsh(skip)]
    pub hash: CryptoHash,
}
```

```rust
pub struct BlockHeaderInnerLite {
    /// Height of this block.
    pub height: BlockHeight,
    /// Epoch start hash of this block's epoch.
    /// Used for retrieving validator information
    pub epoch_id: EpochId,
    pub next_epoch_id: EpochId,
    /// Root hash of the state at the previous block.
    pub prev_state_root: MerkleHash,
    /// Root of the outcomes of transactions and receipts from the previous chunks.
    pub prev_outcome_root: MerkleHash,
    /// Timestamp at which the block was built (number of non-leap-nanoseconds since January 1, 1970 0:00:00 UTC).
    pub timestamp: u64,
    /// Hash of the next epoch block producers set
    pub next_bp_hash: CryptoHash,
    /// Merkle root of block hashes up to the current block.
    pub block_merkle_root: CryptoHash,
}
```

```rust
pub struct BlockHeaderInnerRestV4 {
    /// Hash of block body
    pub block_body_hash: CryptoHash,
    /// Root hash of the previous chunks' outgoing receipts in the given block.
    pub prev_chunk_outgoing_receipts_root: MerkleHash,
    /// Root hash of the chunk headers in the given block.
    pub chunk_headers_root: MerkleHash,
    /// Root hash of the chunk transactions in the given block.
    pub chunk_tx_root: MerkleHash,
    /// Root hash of the challenges in the given block.
    pub challenges_root: MerkleHash,
    /// The output of the randomness beacon
    pub random_value: CryptoHash,
    /// Validator proposals from the previous chunks.
    pub prev_validator_proposals: Vec<ValidatorStake>,
    /// Mask for new chunks included in the block
    pub chunk_mask: Vec<bool>,
    /// Gas price for chunks in the next block.
    pub next_gas_price: Balance,
    /// Total supply of tokens in the system
    pub total_supply: Balance,
    /// List of challenges result from previous block.
    pub challenges_result: ChallengesResult,

    /// Last block that has full BFT finality
    pub last_final_block: CryptoHash,
    /// Last block that has doomslug finality
    pub last_ds_final_block: CryptoHash,

    /// The ordinal of the Block on the Canonical Chain
    pub block_ordinal: NumBlocks,

    pub prev_height: BlockHeight,

    pub epoch_sync_data_hash: Option<CryptoHash>,

    /// All the approvals included in this block
    pub approvals: Vec<Option<Box<Signature>>>,

    /// Latest protocol version that this block producer has.
    pub latest_protocol_version: ProtocolVersion,
}
```

也可以看到区块头中包含的信息也是十分丰富的，从gas-price 到merekle-root-hash，同时也包含了共识投票的信息

最关键的自然是epoch info了。

根据near [共识文档](https://nomicon.io/ChainSpec/Consensus)的描述，咱们可以这么理解，实际上的区块产生是提前根据各个分片的活跃验证者选举出的一批，在一个epoch中（12小时）到了指定高度，就有对应的候选人去出块，等区块提案通过了2/3 那么这个区块就是通过了共识。那么问题来了，1.到了指定高度了，但是没出块，掉线了如何处理？2. 到了指定高度了，候选人作恶了，又该如何处理呢？

问题1：到了指定高度，但是没出块，掉线了如何处理？

**NEAR 的应对机制：**

- **备用验证者接替：** 每个分片都会有多个验证者，当主验证者掉线时，系统会自动选择下一个排名靠前的验证者来接替出块任务。这个备用验证者会立即开始生成新的区块，并广播到网络中。
- **超时机制：** 如果主验证者长时间没有出块，系统会触发超时机制，直接让下一个备用验证者接替。
- **惩罚机制：** 掉线的验证者会受到惩罚，例如质押的代币会被扣除一部分，以激励验证者保持在线。

问题2：到了指定高度，候选人作恶了，又该如何处理呢？

**NEAR 的应对机制：**

- 共识机制的自我修复：
  - **2/3 投票机制：** 如果一个验证者生成的区块是无效的（例如包含双重支付、交易顺序错误等），其他验证者会拒绝接受它。只有当超过 2/3 的验证者同意一个区块时，这个区块才会被添加到主链上。
- 惩罚机制：
  - **质押金扣除：** 如果一个验证者被证明作恶，其质押的代币会受到更严厉的惩罚，甚至会被全部扣除。
  - **声誉损失：** 作恶的验证者会失去社区的信任，未来很难再次成为验证者。
- **Slashing:** NEAR 引入了 slashing 机制，即如果多个验证者共同作恶，他们会受到更严重的惩罚。
- **除名（Ejection）**：恶意行为严重的验证者可能会被从验证者列表中除名，失去未来的验证资格。

### Doomslug快速终结性机制详解

Doomslug是NEAR独特的快速终结性协议，确保区块能够在1-2秒内达到终结性：

```rust
pub struct DoomslugThresholdMode {
    pub threshold: Rational32,
    pub height_diff: BlockHeight,
}

impl Doomslug {
    pub fn process_block_approval(
        &mut self,
        approval: &Approval,
        signer: &ValidatorSigner,
    ) -> Result<(), Error> {
        // 处理区块批准
        // 检查是否达到2/3+1的阈值
        if self.has_threshold_approval(approval.inner.target_height) {
            // 触发快速终结性
            self.finalize_block(approval.inner.target_height)?;
        }
        Ok(())
    }
    
    fn has_threshold_approval(&self, height: BlockHeight) -> bool {
        let total_stake = self.get_total_stake_at_height(height);
        let approval_stake = self.get_approval_stake_at_height(height);
        approval_stake * 3 > total_stake * 2  // 2/3+1 阈值
    }
}
```

**Doomslug的关键特性**：
* **快速确认**：不需要等待完整的BFT轮次
* **安全保证**：仍然保持拜占庭容错的安全性
* **活性保证**：即使在网络分区情况下也能保持进展

### Nightshade分片共识详解

Nightshade是NEAR的分片共识协议，允许多个分片并行处理：

```rust
pub struct ChunkHeader {
    pub chunk_hash: ChunkHash,
    pub prev_block_hash: CryptoHash,
    pub outcome_root: CryptoHash,
    pub prev_state_root: StateRoot,
    pub encoded_merkle_root: CryptoHash,
    pub encoded_length: u64,
    pub height_created: BlockHeight,
    pub height_included: BlockHeight,
    pub shard_id: ShardId,
    pub gas_used: Gas,
    pub gas_limit: Gas,
    pub rent_paid: Balance,
    pub validator_reward: Balance,
    pub balance_burnt: Balance,
    pub outgoing_receipts_root: CryptoHash,
    pub tx_root: CryptoHash,
    pub validator_proposals: Vec<ValidatorStake>,
    pub signature: Signature,
}

impl ChunkProducer {
    pub fn produce_chunk(
        &mut self,
        prev_block_hash: CryptoHash,
        epoch_id: &EpochId,
        last_header: ShardBlockHeader,
        height: BlockHeight,
        shard_id: ShardId,
    ) -> Result<(EncodedShardChunk, Vec<MerklePathItem>, Vec<Receipt>), Error> {
        // 1. 收集待处理的交易和receipts
        let (transactions, receipts) = self.collect_transactions_and_receipts(shard_id)?;
        
        // 2. 执行交易和receipts
        let execution_outcomes = self.apply_transactions_and_receipts(
            &transactions, 
            &receipts, 
            &last_header.prev_state_root()
        )?;
        
        // 3. 生成新的状态根
        let new_state_root = self.compute_state_root(&execution_outcomes)?;
        
        // 4. 创建chunk header
        let chunk_header = ChunkHeader {
            prev_state_root: last_header.prev_state_root(),
            shard_id,
            height_created: height,
            // ... 其他字段
        };
        
        // 5. 对chunk进行编码和签名
        let encoded_chunk = self.encode_and_sign_chunk(chunk_header, transactions)?;
        
        Ok((encoded_chunk, merkle_paths, outgoing_receipts))
    }
}
```

### 跨分片通信机制

NEAR通过Receipt系统实现分片间的异步通信：

```rust
pub enum ReceiptEnum {
    Action(ActionReceipt),
    Data(DataReceipt),
}

pub struct ActionReceipt {
    pub signer_id: AccountId,
    pub signer_public_key: PublicKey,
    pub gas_price: Balance,
    pub output_data_receivers: Vec<DataReceiver>,
    pub input_data_ids: Vec<CryptoHash>,
    pub actions: Vec<Action>,
}

impl CrossShardCommunication {
    pub fn process_cross_shard_receipts(
        &mut self,
        receipts: Vec<Receipt>,
        shard_id: ShardId,
    ) -> Result<Vec<Receipt>, Error> {
        let mut outgoing_receipts = Vec::new();
        
        for receipt in receipts {
            match receipt.receiver_id.get_shard_id() {
                target_shard if target_shard == shard_id => {
                    // 本分片内处理
                    self.apply_receipt_locally(receipt)?;
                }
                target_shard => {
                    // 跨分片发送
                    outgoing_receipts.push(receipt);
                }
            }
        }
        
        Ok(outgoing_receipts)
    }
}
```

### 共识性能优化策略

NEAR共识机制的性能优化策略：

1. **并行chunk生产**：多个分片可以同时生产chunks
2. **异步跨分片通信**：通过Receipt实现非阻塞的分片间通信
3. **预执行机制**：在区块确认前预先执行交易
4. **状态见证**：减少验证者需要存储的状态数据
5. **动态分片**：根据网络负载动态调整分片数量

### 共识安全性保证

* **拜占庭容错**：能够容忍最多1/3的恶意验证者
* **终结性保证**：通过Doomslug提供快速且安全的终结性
* **分片安全**：每个分片都有独立的安全保证
* **跨分片一致性**：通过Receipt机制保证全局状态一致性

