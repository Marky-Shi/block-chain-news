## staking reward

### cosmos

cosmos-sdk/x/distribution/keeper/allocation.go 

将reward（根据佣金进行拆分） 分发给validator 

> 在 Cosmos SDK 中，验证人佣金率是 **由验证人自行设置的**。虽然 Cosmos SDK **没有严格规定** 的默认佣金率，但 Cosmos 社区建议了 **推荐准则**。这些准则建议默认佣金率为 **5% 到 10%**。这个范围被认为是激励验证人运行节点和确保委托人获得公平份额奖励之间的合理平衡。

```go
func (k Keeper) AllocateTokensToValidator(ctx context.Context, val sdk.ValidatorI, tokens sdk.DecCoins) error {
  
  // 拆分奖励，验证节点设定的佣金率 (val.GetCommission()) 计算其佣金。
  // shared 剩余部分分配给委托人。
  commission := tokens.MulDec(val.GetCommission())
	shared := tokens.Sub(commission)
  
  // 更新佣金
  if err = k.EventService.EventManager(ctx).EmitKV(
		//触发事件 (types.EventTypeCommission)，记录佣金金额和验证节点地址。
    types.EventTypeCommission,
		event.NewAttribute(sdk.AttributeKeyAmount, commission.String()),
		event.NewAttribute(types.AttributeKeyValidator, val.GetOperator()),
	);
  // 将新计算的佣金添加到现有的累计佣金中。
  currentCommission, err := k.ValidatorsAccumulatedCommission.Get(ctx, valBz)
  currentCommission.Commission = currentCommission.Commission.Add(commission...)
  
  
  // 更新 reward 将奖励的共享部分 (shared) 添加到现有的验证节点奖励中。
  currentRewards, err := k.ValidatorCurrentRewards.Get(ctx, valBz)
  currentRewards.Rewards = currentRewards.Rewards.Add(shared...)
  
  // 更新未领取的奖励
  if err = k.EventService.EventManager(ctx).EmitKV(
    // 触发事件。eventTypeRewards
		types.EventTypeRewards,
		event.NewAttribute(sdk.AttributeKeyAmount, tokens.String()),
		event.NewAttribute(types.AttributeKeyValidator, val.GetOperator()),
	)
  // 获取尚未领取的奖励
  outstanding, err := k.ValidatorOutstandingRewards.Get(ctx, valBz)
  
  //将分配的总奖励 (tokens) 添加到现有的未领取奖励中。 
  outstanding.Rewards = outstanding.Rewards.Add(tokens...)
}
```

```go
func (k Keeper) CalculateDelegationRewards(ctx context.Context, val sdk.ValidatorI, del sdk.DelegationI, endingPeriod uint64) (rewards sdk.DecCoins, err error) {
	...prepare do something...
  
  // 计算惩罚
	if endingHeight > startingHeight {
    // 遍历验证人之间起始期间 (startingPeriod) 和结束期间 (endingPeriod) 之间的所有惩罚事件
		err = k.IterateValidatorSlashEventsBetween(ctx, valAddr, startingHeight, endingHeight,
			func(height uint64, event types.ValidatorSlashEvent) (stop bool) {
				endingPeriod := event.ValidatorPeriod
				// 对于每个惩罚事件，它会计算在前一个结束期间 (startingPeriod) 和当前惩罚事件期间 (endingPeriod) 之间获得的奖励。这将考虑期间开始时的质押金额 (stake)。
        if endingPeriod > startingPeriod {
					delRewards, err := k.calculateDelegationRewardsBetween(ctx, val, startingPeriod, endingPeriod, stake)
					if err != nil {
						iterErr = err
						return true
					}
					rewards = rewards.Add(delRewards...)

					// 将质押金额乘以 (1 - 惩罚分数) 以考虑惩罚扣款，从而降低质押金额。
					stake = stake.MulTruncate(math.LegacyOneDec().Sub(event.Fraction))
					startingPeriod = endingPeriod
				}
				return false
			},
		)
  }
  
  // 质押有效性检查
  // 根据委托人的委托份额和验证人的代币到份额转换率计算委托人的当前质押金额
  currentStake := val.TokensFromShares(del.GetShares())
  
  if stake.GT(currentStake) {
  	// 它将通过惩罚迭代计算出的最终质押额 (stake) 与委托人的当前质押额进行比较。  
    marginOfErr := math.LegacySmallestDec().MulInt64(3)
		if stake.LTE(currentStake.Add(marginOfErr)) {
			stake = currentStake
		} else {
			return sdk.DecCoins{}, fmt.Errorf("calculated final stake for delegator %s greater than current stake"+
				"\n\tfinal stake:\t%s"+
				"\n\tcurrent stake:\t%s",
				del.GetDelegatorAddr(), stake, currentStake)
		}
  }
  
  //计算最终期间的奖励，该期间介于最后一个惩罚事件（或没有惩罚事件则为起始期间）和结束期间 (endingPeriod) 之间。它使用最终质押值 (stake) 进行此计算。
  delRewards, err := k.calculateDelegationRewardsBetween(ctx, val, startingPeriod, endingPeriod, stake)
  rewards = rewards.Add(delRewards...)
}
```



**委托奖励 = 委托人质押金额 * (结束奖励比率 - 起始奖励比率)**

```go
func (k Keeper) calculateDelegationRewardsBetween(ctx context.Context, val sdk.ValidatorI,
	startingPeriod, endingPeriod uint64, stake math.LegacyDec,
                                                 ) (sdk.DecCoins, error) {
  ...do something...
 // staking * (ending - starting) 
  starting, err := k.ValidatorHistoricalRewards.Get(ctx, collections.Join(sdk.ValAddress(valBz), startingPeriod))
	if err != nil {
		return sdk.DecCoins{}, err
	}

	ending, err := k.ValidatorHistoricalRewards.Get(ctx, collections.Join(sdk.ValAddress(valBz), endingPeriod))
	if err != nil {
		return sdk.DecCoins{}, err
	}

	difference := ending.CumulativeRewardRatio.Sub(starting.CumulativeRewardRatio)
	if difference.IsAnyNegative() {
		return sdk.DecCoins{}, fmt.Errorf("negative rewards should not be possible")
	}
  rewards := difference.MulDecTruncate(stake)
}
```

Staking 奖励的计算主要取决于以下几个因素：

- **委托的代币数量:** 委托人获得的奖励与他们委托的代币数量成正比。
- **验证人的总委托量:** 验证人获得的总奖励与他们所委托的总代币数量成正比。
- **验证人的佣金率:** 验证人会保留一部分奖励作为佣金，剩余部分分配给委托人。
- **网络的整体通胀率:** 网络的通胀率会影响 Staking 奖励的总量。

公式：

**总委托奖励 = Σ (周期奖励)**

**周期奖励 = 委托人质押金额 * (验证人周期奖励) * (1 - 验证人佣金率)**

假设 `R` 是验证人获得的总奖励，`D` 是委托人委托的代币数量，`T` 是验证人的总委托量，`C` 是验证人的佣金率，`I` 是网络的年通胀率，则委托人获得的 Staking 奖励为：

```
委托人奖励 = R * (D / T) * (1 - C) / (1 + I)
```

假设验证人 A 获得了 100 个代币的区块奖励，总委托量为 1000 个代币，佣金率为 10%，网络通胀率为 5%。委托人 B 委托了 100 个代币给验证人 A。那么，委托人 B 将获得的 Staking 奖励为：

```
委托人奖励 = 100 * (100 / 1000) * (1 - 0.1) / (1 + 0.05) ≈ 8.57
```

slash

```go
func (k Keeper) HandleValidatorSignatureWithParams(ctx context.Context, params types.Params, addr cryptotypes.Address, power int64, signed comet.BlockIDFlag) error {
	......do something
  
  if validator != nil && !validator.IsJailed() {
    distributionHeight := height - sdk.ValidatorUpdateDelay - 1

			slashFractionDowntime, err := k.SlashFractionDowntime(ctx)
			if err != nil {
				return err
			}

			coinsBurned, err := k.sk.SlashWithInfractionReason(ctx, consAddr, distributionHeight, power, slashFractionDowntime, st.Infraction_INFRACTION_DOWNTIME)
			if err != nil {
				return err
			}

			if err := k.EventService.EventManager(ctx).EmitKV(
        // 触发slash 事件
				types.EventTypeSlash,
				event.NewAttribute(types.AttributeKeyAddress, consStr),
				event.NewAttribute(types.AttributeKeyPower, fmt.Sprintf("%d", power)),
				event.NewAttribute(types.AttributeKeyReason, types.AttributeValueMissingSignature),
				event.NewAttribute(types.AttributeKeyJailed, consStr),
				event.NewAttribute(types.AttributeKeyBurnedCoins, coinsBurned.String()),
			); err != nil {
				return err
			}

			err = k.sk.Jail(ctx, consAddr)
			if err != nil {
				return err
			}
			downtimeJailDur, err := k.DowntimeJailDuration(ctx)
			if err != nil {
				return err
			}
			signInfo.JailedUntil = k.HeaderService.HeaderInfo(ctx).Time.Add(downtimeJailDur)
  }

}
```

#### slash 计算

```go
func (k Keeper) Slash(ctx context.Context, consAddr sdk.ConsAddress, infractionHeight, power int64, slashFactor math.LegacyDec) (math.Int, error) {
	// 计算惩罚额度
  amount := k.TokensFromConsensusPower(ctx, power)
	slashAmountDec := math.LegacyNewDecFromInt(amount).Mul(slashFactor)
	slashAmount := slashAmountDec.TruncateInt()
  
  // 进行检测，包括检测validator是否存在，是否已解除委托等
  
  ......
  
  // 跟踪剩余惩罚金额
  remainingSlashAmount := slashAmount
  
  // 根据违规高度进行分类处理
  switch {
	case infractionHeight > height:
		// Can't slash infractions in the future
		return math.NewInt(0), fmt.Errorf(
			"impossible attempt to slash future infraction at height %d but we are at height %d",
			infractionHeight, height)
	case infractionHeight == height:
		// Special-case slash at current height for efficiency - we don't need to
		// look through unbonding delegations or redelegations.
		k.Logger.Info(
			"slashing at current height; not scanning unbonding delegations & redelegations",
			"height", infractionHeight,
		)
   // 遍历验证人当时的所有未锁定委托和换委托记录，并分别进行惩罚。 
   case infractionHeight < height:
    	k.GetUnbondingDelegationsFromValidator(ctx, operatorAddress)
    	amountSlashed, err := k.SlashUnbondingDelegation(ctx, unbondingDelegation, infractionHeight, slashFactor)
    redelegations, err := k.GetRedelegationsFromSrcValidator(ctx, operatorAddress)
    amountSlashed, err := k.SlashRedelegation(ctx, validator, redelegation, infractionHeight, slashFactor)
    remainingSlashAmount = remainingSlashAmount.Sub(amountSlashed)
    
    .......
    
    // 计算惩罚因子
    effectiveFraction := math.LegacyNewDecFromInt(tokensToBurn).QuoRoundUp(math.LegacyNewDecFromInt(validator.Tokens))
    // 更新validator 信息，并返回罚金
    validator, err = k.RemoveValidatorTokens(ctx, validator, tokensToBurn)
    switch validator.GetStatus() {
	case sdk.Bonded:
		if err := k.burnBondedTokens(ctx, tokensToBurn); err != nil {
			return math.NewInt(0), err
		}
	case sdk.Unbonding, sdk.Unbonded:
		if err := k.burnNotBondedTokens(ctx, tokensToBurn); err != nil {
			return math.NewInt(0), err
		}
	default:
		return math.NewInt(0), fmt.Errorf("invalid validator status")
	}
    return tokensToBurn, nil
}
```

验证人受到的惩罚包括：

- **罚款:** 从验证人的奖励中扣除一定金额的代币。
- **降低委托权重:** 降低验证人被选为出块节点的可能性。
- **踢出网络:** 将验证人从网络中踢出。

**具体计算公式:**

惩罚金额 = 惩罚因子 * 违规时验证人权重

验证人罚款的计算公式通常取决于违规行为的性质和程度。例如，如果验证人宕机时间过长，可能会被罚款其奖励的 10%。

验证人委托权重的降低通常与罚款的比例相同。例如，如果验证人被罚款 10%，其委托权重也可能会降低 10%。

验证人被踢出网络的决定通常由网络治理机制决定。

**举例说明:**

假设验证人 A 宕机了 10 个小时。根据网络规则，验证人每宕机 1 小时会被罚款其奖励的 1%，并降低其委托权重 1%。那么，验证人 A 将被罚款其奖励的 10%，并降低其委托权重 10%。



### Polkadot 

#### Staking 奖励计算

Polkadot/runtime/common/src/impl.rs

> 根据网络的质押情况和通胀参数计算每个时代的 staking 奖励分配。函数会综合考虑理想质押比例、实际质押比例和拍卖插槽等因素来动态调整 staking 通胀率，从而确保 Polkadot 网络的安全性和通胀可控性。

```rust
pub fn era_payout(
	total_staked: Balance, //总质押量
	total_stakable: Balance,//网络中可质押的总量
	max_annual_inflation: Perquintill,//最大年度通胀率
	period_fraction: Perquintill,//时代所占的年度比例 
	auctioned_slots: u64,//拍卖的插槽数量
) -> (Balance, Balance)
```

* 计算最低与最高年度通胀
  * `min_annual_inflation`: 设置最低年度通胀率为 2.5%
  * `delta_annual_inflation`: 计算最大年度通胀率与最低年度通胀率的差值
* 计算插槽拍卖比例
  * `auction_proportion`: 根据拍卖插槽数量计算拍卖比例，最高为 30% (上限 60 个插槽)
* 计算理想质押比
  * `ideal_stake`: 计算理想的质押比例，为总发行量的 75% 减去拍卖比例
* 计算质押调整系数
  * `stake` 当前质押比 (总质押量 / 可质押总量)
  * `falloff`: 设置衰减因子 (5%)
  * 基于当前质押比例、理想质押比例和衰减因子计算质押调整系数(`compute_inflation(stake, ideal_stake, falloff)`)
* 计算staking reward
  * `staking_inflation`: 根据最低年度通胀率、最大年度通胀率差值和质押调整系数计算 staking 通胀率
  * `max_payout`: 计算最大时代发行量 (时代比例 * 最大年度通胀率 * 可质押总量)
  * `staking_payout`: 计算 staking 奖励 (**时代比例 * staking 通胀率 * 可质押总量**)
  * `rest`: 计算剩余发行量 (最大时代发行量 - staking 奖励)

> 计算每个周期发放给validator 以及 nominator 的奖励总和，以及最大供应量
>
> Substrate/frame/staking/src/inflation.rs

```rust
pub fn compute_total_payout<N>(
	yearly_inflation: &PiecewiseLinear<'static>,  // 年通胀
	npos_token_staked: N, //当前质押的 NPOS 代币数量
	total_tokens: N, //总发行量 
	era_duration: u64, // era 持续的时间
) -> (N, N)
where
	N: AtLeast32BitUnsigned + Clone,
{
	// Milliseconds per year for the Julian year (365.25 days).
	const MILLISECONDS_PER_YEAR: u64 = 1000 * 3600 * 24 * 36525 / 100;

	let portion = Perbill::from_rational(era_duration as u64, MILLISECONDS_PER_YEAR);
	
  // 实际发放的总奖励（validaotor + nominator）
  let payout = portion *
		yearly_inflation
			.calculate_for_fraction_times_denominator(npos_token_staked, total_tokens.clone());
	
  // 最大发行量
  let maximum = portion * (yearly_inflation.maximum * total_tokens);
  
	(payout, maximum)
}
```

```shell
`staker-payout = yearly_inflation(npos_token_staked / total_tokens) * total_tokens /*era_per_year`

`maximum-payout = max_yearly_inflation * total_tokens / era_per_year`
```



> 提名人与验证人的实际奖励计算
>
> staking/src/pallet/impl.rs

```rust
pub(super) fn do_payout_stakers_by_page(
		validator_stash: T::AccountId,
		era: EraIndex,
		page: Page,
	) -> DispatchResultWithPostInfo {
    ... do something ...
    // 获取era 奖励
    let era_payout = <ErasValidatorReward<T>>::get(&era).ok_or_else(|| {
			Error::<T>::InvalidEraToReward
				.with_weight(T::WeightInfo::payout_stakers_alive_staked(0))
		})?;
    
    ... do something ...
    
    // 计算 nominator & validator 奖励
    // 计算验证人应得奖励比例
    let era_reward_points = <ErasRewardPoints<T>>::get(&era);
		let total_reward_points = era_reward_points.total;
		let validator_reward_points =
    	era_reward_points.individual.get(&stash).copied().unwrap_or_else(Zero::zero);

    if validator_reward_points.is_zero() {
        return Ok(Some(T::WeightInfo::payout_stakers_alive_staked(0)).into())
    }

    let validator_total_reward_part =
      	Perbill::from_rational(validator_reward_points, total_reward_points);
    // 验证人的总奖励
    let validator_total_payout = validator_total_reward_part * era_payout;
		
    // 验证人的佣金比例以及佣金计算
    let validator_commission = EraInfo::<T>::get_validator_commission(era, &ledger.stash);
    let validator_total_commission_payout = validator_commission * validator_total_payout;
    
    // validator_leftover_payout 是验证人剩余奖励，用于分配给提名人。
    let validator_leftover_payout =
        validator_total_payout.defensive_saturating_sub(validator_total_commission_payout);
    let validator_exposure_part = Perbill::from_rational(exposure.own(), exposure.total());
    // 验证人应得奖励
    let validator_staking_payout = validator_exposure_part * validator_leftover_payout;
    
    // page_stake_part 是该页面质押比例，用于根据页面质押比例分配佣金。
    let page_stake_part = Perbill::from_rational(exposure.page_total(), exposure.total());
    // 验证人佣金分摊部分
    let validator_commission_payout = page_stake_part * validator_total_commission_payout;
		
    
    // 提名人奖励计算
    let mut nominator_payout_count: u32 = 0;
		for nominator in exposure.others().iter() {
      // 计算每个提名者的质押比例
			let nominator_exposure_part = Perbill::from_rational(nominator.value, exposure.total());
			
      // 提名人奖励计算
			let nominator_reward: BalanceOf<T> =
				nominator_exposure_part * validator_leftover_payout;
			//奖励分发
			if let Some((imbalance, dest)) = Self::make_payout(&nominator.who, nominator_reward) {
				// 更新数据，并触发事件
				nominator_payout_count += 1;
				let e = Event::<T>::Rewarded {
					stash: nominator.who.clone(),
					dest,
					amount: imbalance.peek(),
				};
				Self::deposit_event(e);
				total_imbalance.subsume(imbalance);
			}
		}
    ... do something ...
}
```

在 Polkadot 中，Staking 奖励的计算主要取决于以下几个因素：

- **委托的 DOT 数量:** 委托人获得的奖励与他们委托的 DOT 数量成正比。
- **验证人的总委托量:** 验证人获得的总奖励与他们所委托的总 DOT 数量成正比。
- **验证人的提名率:** 验证人获得的奖励与他们获得的提名数量成正比。
- **验证人的佣金率:** 验证人会保留一部分奖励作为佣金，剩余部分分配给委托人。
- **网络的通胀率:** 网络的通胀率会影响 Staking 奖励的总量。

#### 计算公式：

* E 是era的奖励总额。
* Pv 是验证者的奖励点。
* Pt 是era的奖励总点数。
* C 是验证者的佣金比例。
* Sv 是验证者自己的质押量。
* St 是验证者和所有委托人的总质押量。
* Sp 是当前页的质押总量。

验证者总奖励部分：
$$
Rv_{total} =E \times \frac{P_v}{P_t}
$$
验证者总佣金奖励：
$$
Rv_{commission\_total} = Rv_{total} \times C
$$
验证者剩余奖励（分发给提名人的部分）：
$$
Rv_{leftover} = Rv_{total} - Rv_{commission\_total}
$$
验证人实际得到的奖励：
$$
Rv = Rv_{leftover} \times \frac{Sv}{St} + Rv_{commission\_total} \times \frac{Sp}{St}
$$
提名者的奖励：

对于每个提名者其质押量为
$$
S_i
$$

$$
Ri = Rv_{leftover} \times \frac{Si}{St}
$$
example：

* **Era总奖励 E**: 1000 DOT
* **Era的奖励总点数 Pt**: 2000
* **验证者的奖励点 Pv**: 400
* **验证者的佣金比例 C**: 10% (0.1)
* **验证者自己的质押量 Sv**: 100 DOT
* **验证者和所有委托人的总质押量 St**: 500 DOT
* **当前页的质押总量 Sp**: 150 DOT
* **委托人 A 的质押量 SA**: 200 DOT
* **委托人 B 的质押量 SB**: 100 DOT
* **当前页的质押总量 Sp**: 150 DOT

验证者的总奖励：
$$
Rv_{total} = E \times \frac{Pt}{Pv} = 1000 \times \frac{2000}{400} = 200 DOT
$$
验证者的总佣金：
$$
Rv_{commission\_total} = Rv_{total} \times C = 200 \times 0.1 = 20 DOT
$$
验证者剩余奖励：
$$
Rv_{leftover} = Rv_{total} - Rv_{commission\_total} = 200 - 20 = 180 DOT
$$
验证者实际得到的奖励：
$$
Rv = Rv_{leftover} \times \frac{St}{Sv} + Rv_{commission\_total} \times \frac{St}{Sp} =
180 \times \frac{100}{500} + 20 \times \frac{150}{500} = 180 \times 0.2 + 20 \times 0.3 = 36 + 6 = 42 DOT
$$
提名人A的奖励：
$$
RA = Rv_{leftover} \times \frac{St}{SA} = 180 \times \frac{200}{500} = 180 \times 0.4 = 72 DOT
$$
提名人B的奖励：
$$
RB = Rv_{leftover} \times \frac{St}{SB} = 180 \times \frac{500}{100} = 180 \times 0.2 = 36 DOT
$$

#### Slash

slash的计算则是在

> Substrate/frame/staking/src/slashing.rs

```rust
pub fn do_slash<T: Config>(
	stash: &T::AccountId,
	value: BalanceOf<T>,
	reward_payout: &mut BalanceOf<T>,
	slashed_imbalance: &mut NegativeImbalanceOf<T>,
	slash_era: EraIndex,
) {
  ... do something ...
  let value = ledger.slash(value, T::Currency::minimum_balance(), slash_era);
	if value.is_zero() {
		// nothing to do
		return
	}
  ... do something...
}
```

```rust
pub fn slash(
		&mut self,
		slash_amount: BalanceOf<T>, // 针对验证人提出的初始惩罚额度。
		minimum_balance: BalanceOf<T>, //验证人的最低余额要求
		slash_era: EraIndex,// 惩罚发生的时间段
	) -> BalanceOf<T> {
    
}
```

如果 `slash_amount` 大于总余额的一部分，需要按照比例削减 `active` 和 `unlocking` 的 chunks。

1. **计算比例**： 
   $$
   ratio = \frac{\text{total\_balance}}{\text{slash\_amount}}
   $$
    其中，`total_balance` 是 `active` 和所有 `unlocking` chunks 的总和。

2. **削减 active 部分**： slash_from_active=ratio×activeslash_from_active=ratio×activenew_active=active−slash_from_activenew_active=active−slash_from_active

3. **削减 unlocking 部分**： 对每个 `unlocking` chunk： 
   $$
   slash_from_chunk=ratio×chunk.value
   $$

   $$
   NewChunkValue=chunk.value−SlashFromChunk
   $$

   

#### 非比例削减

如果 `slash_amount` 小于某些部分，需要从 `active` 和 `unlocking` chunks 中按顺序削减。

1. **从 active 部分削减**： 
   $$
   slash_from_active=min⁡(slash_amount,active)
   $$

   $$
   NewActive=active−SlashFromActive
   $$

   

2. **从 unlocking 部分按顺序削减**： 对每个 `unlocking` chunk：
   $$
   SlashFromChunk=min⁡(RemainingSlash,chunk.value)
   $$

   $$
   NewChunkValue=chunk.value−SlashFromChunk
   $$

   

3. **更新 total**： 
   $$
   NewTotal=PreSlashTotal−SlashAmount
   $$
   

### 例子

假设以下初始值：

- `slash_amount = 100`
- `minimum_balance = 10`
- `slash_era = 5`
- `active = 150`
- `unlocking` chunks: {era: 7, value: 50,era: 8, value: 50{era: 7, value: 50,era: 8, value: 50]
- `total_balance = 150 + 50 + 50 = 250`

#### 比例削减

1. **计算比例**： 
   $$
   ratio = \frac{250}{100} = 0.4
   $$
   
2. **削减 active 部分**： 
   $$
   SlashFromActive=0.4×150=60 
   $$

   $$
   NewActive=150−60=90
   $$

   

3. **削减 unlocking 部分**： 
   $$
   SlashFromChunk_1=0.4×50=20
   $$
   
   $$
   NewChunkValue_1=50−20=30
   $$
   

   **更新 total**： 
   $$
   NewTotal=90+30+30=150
   $$
   

#### 非比例削减

假设 `slash_amount = 50`:

1. **从 active 部分削减**：
   $$
   SlashFromActive=min⁡(50,150)=50
   $$

   $$
   NewActive=150−50=10
   $$

   

2. **不需要从 unlocking 部分削减**，因为 `remaining_slash` 已经为零。

3. **更新 total**： 
   $$
   NewTotal=100+50+50=200
   $$
