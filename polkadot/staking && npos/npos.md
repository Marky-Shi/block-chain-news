## npos source code

核心选举算法：Phragmen

Path: polkadot-sdk/substrate/frame/elections-phragmen

Name : pallet-elections-phragmen



```rust
fn do_phragmen() -> Weight {
  
  // 获取候选人信息
  
		let desired_seats = T::DesiredMembers::get() as usize;
		let desired_runners_up = T::DesiredRunnersUp::get() as usize;
		let num_to_elect = desired_runners_up + desired_seats;

		let mut candidates_and_deposit = Candidates::<T>::get();
		// add all the previous members and runners-up as candidates as well.
		candidates_and_deposit.append(&mut Self::implicit_candidates_with_deposit());
  
  	// 根据总发行量在股权和投票权之间进行转换。
  	let total_issuance = T::Currency::total_issuance();
		let to_votes = |b: BalanceOf<T>| T::CurrencyToVote::to_vote(b, total_issuance);
		let to_balance = |e: ExtendedBalance| T::CurrencyToVote::to_currency(e, total_issuance);
  	
  	// 处理投票者生成一个列表，用于 Phragmen 算法，将投票者股权转换为投票权
  	let voters_and_votes = voters_and_stakes
			.iter()
			.cloned()
			.map(|(voter, stake, votes)| {
				num_edges = num_edges.saturating_add(votes.len() as u32);
				(voter, to_votes(stake), votes)
			})
			.collect::<Vec<_>>();
  
  
  	
  	// 执行phragmen 算法,此函数执行 Phragmen 选举并返回当选的验证节点及其分配的投票者。
  	let _ =
			sp_npos_elections::seq_phragmen(num_to_elect, candidate_ids, voters_and_votes, None)
				.map(|ElectionResult::<T::AccountId, Perbill> { winners, assignments: _ }| {
         	 // 获取旧成员和候补列表
          	old_members_ids_sorted
          	old_runners_up_ids_sorted
          
          // 筛选出有支持的新的winner
          let mut new_set_with_stake = winners
						.into_iter()
						.filter_map(
							|(m, b)| if b.is_zero() { None } else { Some((m, to_balance(b))) },
						)
						.collect::<Vec<(T::AccountId, BalanceOf<T>)>>();
          
          //将剩余的验证节点拆分为所需的成员数量 和候补节点。
          new_members_sorted_by_id
          new_runners_up_sorted_by_rank
          
          // 使用borad 计数法 选取prime 成员
          let mut prime_votes = new_members_sorted_by_id
						.iter()
						.map(|c| (&c.0, BalanceOf::<T>::zero()))
						.collect::<Vec<_>>();
					for (_, stake, votes) in voters_and_stakes.into_iter() {
						for (vote_multiplier, who) in
							votes.iter().enumerate().map(|(vote_position, who)| {
								((T::MaxVotesPerVoter::get() as usize - vote_position) as u32, who)
							}) {
							if let Ok(i) = prime_votes.binary_search_by_key(&who, |k| k.0) {
								prime_votes[i].1 = prime_votes[i]
									.1
									.saturating_add(stake.saturating_mul(vote_multiplier.into()));
							}
						}
					}
          let prime = prime_votes.into_iter().max_by_key(|x| x.1).map(|x| x.0.clone());
          
          // 更新验证节点集，清楚不是验证人或候选人的验证节点，并进行罚金
          candidates_and_deposit.iter().for_each(|(c, d)| {
						if new_members_ids_sorted.binary_search(c).is_err() &&
							new_runners_up_ids_sorted.binary_search(c).is_err()
						{
							let (imbalance, _) = T::Currency::slash_reserved(c, *d);
							T::LoserCandidate::on_unbalanced(imbalance);
							Self::deposit_event(Event::CandidateSlashed {
								candidate: c.clone(),
								amount: *d,
							});
						}
					});
          
          // 记录新的选举事件
          Self::deposit_event(Event::NewTerm { new_members: new_members_sorted_by_id });
					ElectionRounds::<T>::mutate(|v| *v += 1);
         }; 	
}
```





Phragmen 算法的核心部分，它的功能是从给定的候选人和投票者列表中选出指定数量的候选人



```rust
pub fn seq_phragmen<AccountId: IdentifierT, P: PerThing128>(
	to_elect: usize,
	candidates: Vec<AccountId>,
	voters: Vec<(AccountId, VoteWeight, impl IntoIterator<Item = AccountId>)>,
	balancing: Option<BalancingConfig>,
) -> Result<ElectionResult<AccountId, P>, crate::Error> {
  
}
```

- `to_elect`: 需要选出的候选人数。
- `candidates`: 候选人列表。
- `voters`: 投票者及其投票的列表，每个投票者包括其账户ID、投票权重和一个候选人列表。
- `balancing`: 可选的额度配置。

调用核心 Phragmen 算法

```rust
let (candidates, mut voters) = seq_phragmen_core::<AccountId>(to_elect, candidates, voters)?;
```

调用 `seq_phragmen_core` 函数执行 Phragmen 算法的核心计算，返回处理后的候选人和投票者。

平衡投票者

```rust
if let Some(ref config) = balancing {
	// NOTE: might create zero-edges, but we will strip them again when we convert voter into
	// assignment.
	let _iters = balancing::balance::<AccountId>(&mut voters, config);
}
```

选出winner 并进行排序

```rust
let mut winners = candidates
	.into_iter()
	.filter(|c_ptr| c_ptr.borrow().elected)
	// defensive only: seq-phragmen-core returns only up to rounds.
	.take(to_elect)
	.collect::<Vec<_>>();

winners.sort_by_key(|c_ptr| c_ptr.borrow().round);
```

- 过滤出被选中的候选人。
- 只取前 `to_elect` 个候选人。
- 根据候选人的轮次（`round`）对胜利者进行排序。

创建分配列表

```rust
let mut assignments = voters.into_iter().filter_map(|v| v.into_assignment()).collect::<Vec<_>>();
let _ = assignments.iter_mut().try_for_each(|a| a.try_normalize().map_err(crate::Error::ArithmeticError))?;
```

- 将投票者转换为分配列表（`assignments`）。
- 尝试对每个分配进行规范化处理，捕获并返回任何算术错误。

将胜利者转换为最终的格式，其中包含账户ID和支持的质押金额。

```rust
let winners = winners
		.into_iter()
		.map(|w_ptr| (w_ptr.borrow().who.clone(), w_ptr.borrow().backed_stake))
		.collect();
```

该函数的主要步骤包括：

1. 设置和预处理输入。
2. 调用核心 Phragmen 算法进行计算。
3. 根据需要对投票者进行平衡。
4. 过滤并排序胜利者。
5. 创建并规范化分配列表。
6. 构建并返回最终的选举结果。

这段代码实现了 Phragmen 算法的完整流程，并确保在整个过程中数据的一致性和准确性。



Phragmen-core

```rust
pub fn seq_phragmen_core<AccountId: IdentifierT>(
	to_elect: usize,
	candidates: Vec<CandidatePtr<AccountId>>,
	mut voters: Vec<Voter<AccountId>>,
) -> Result<(Vec<CandidatePtr<AccountId>>, Vec<Voter<AccountId>>), crate::Error> {
  
```

* `to_elect`: 需要选出的候选人数。
* `candidates`: 候选人的引用指针列表。
* `voters`: 投票者列表。

候选人选举需要进行多轮，每一轮选出一个候选人，知道满足所需数量的候选人，或没有可选择的候选人为止。

* *initialize score*

  ```rust
  for c_ptr in &candidates {
  	let mut candidate = c_ptr.borrow_mut();
  	if !candidate.elected {
  		if candidate.approval_stake.is_zero() {
  			candidate.score = Bounded::max_value();
  		} else {
  			candidate.score = Rational128::from(DEN / candidate.approval_stake, DEN);
  		}
  	}
  }
  ```

  * 为每个尚未当选的候选人初始化score
  * 如果候选人的支持量为零，则score设置为最大值，否则根据支持量计算score。

* *increment score* 

  ```rust
  for voter in &voters {
  	for edge in &voter.edges {
  		let mut candidate = edge.candidate.borrow_mut();
  		if !candidate.elected && !candidate.approval_stake.is_zero() {
  			let temp_n = multiply_by_rational_with_rounding(
  				voter.load.n(),
  				voter.budget,
  				candidate.approval_stake,
  				Rounding::Down,
  			).unwrap_or(Bounded::max_value());
  			let temp_d = voter.load.d();
  			let temp = Rational128::from(temp_n, temp_d);
  			candidate.score = candidate.score.lazy_saturating_add(temp);
  		}
  	}
  }
  ```

  * 对每个投票者及其支持的候选人，计算候选人的incerment score
  * 更新尚未当选且支持量不为零的候选人的score

* find best  

  ```rust
  if let Some(winner_ptr) = candidates
  			.iter()
  			.filter(|c| !c.borrow().elected)
  			.min_by_key(|c| c.borrow().score)
  		{
  			let mut winner = winner_ptr.borrow_mut();
  			// loop 3: update voter and edge load
  			winner.elected = true;
  			winner.round = round;
  			for voter in &mut voters {
  				for edge in &mut voter.edges {
  					if edge.who == winner.who {
  						edge.load = winner.score.lazy_saturating_sub(voter.load);
  						voter.load = winner.score;
  					}
  				}
  			}
  		} else {
  			break
  		}
  	}
  ```

  * 选择score最低的未当选候选人作为本轮的当选者
  * 更新投票者和候选人的负载

更新候选人和投票者的支持量

```rust
for voter in &mut voters {
	for edge in &mut voter.edges {
		if edge.candidate.borrow().elected {
			edge.weight = multiply_by_rational_with_rounding(
				voter.budget,
				edge.load.n(),
				voter.load.n(),
				Rounding::Down,
			).unwrap_or(Bounded::max_value());
		} else {
			edge.weight = 0
		}
		let mut candidate = edge.candidate.borrow_mut();
		candidate.backed_stake = candidate.backed_stake.saturating_add(edge.weight);
	}
	voter.edges.retain(|e| e.weight > 0);
	debug_assert!(voter.edges.iter().all(|e| e.candidate.borrow().elected));
	voter.try_normalize_elected().map_err(crate::Error::ArithmeticError)?;
}

```

**更新支持股权 (backing stake):**

- 遍历所有投票者及其投票边：
  - 如果投票边指向的候选人是当选人：
    - 计算投票者分配给该当选人的权重：
      - 权重等于投票者分配给该当选人的预算乘以投票边负载的分子，再除以投票者总负载的分子，并向下取整。
    - 更新当选人的支持股权，加上该投票者分配的权重。
  - 如果投票边指向的候选人不是当选人，则将权重设置为 0。

**清理无效投票边:**

- 删除所有权重为 0 的投票边，这些边在后续归一化过程中可能会变成无效边。

**归一化投票者预算 (voter.try_normalize_elected)**:

* 尝试将投票者分配给当选人的权重进行归一化，确保总权重为 1。



> Phragmen算法是一种用于选举和资源分配的算法，主要用于保证选举过程中的公平性和平衡性。它最早由瑞典数学家 Lars Edvard Phragmén 提出，并在现代区块链系统（如Polkadot）中被广泛应用，尤其是在选举验证节点和委员会成员时。
>
> ### Phragmen算法的工作原理
>
> 1. **候选人和选民的初始化**：
>    - 每个候选人和选民都会初始化，候选人可以被选民投票支持。
> 2. **投票权重**：
>    - 每个选民都有一定的投票权重，他们可以将这些权重分配给不同的候选人。
> 3. **选票分配**：
>    - 投票者根据他们的偏好将他们的投票权重分配给候选人。投票者可以支持一个或多个候选人。
> 4. **计算负载**：
>    - 每个候选人的负载是他所获得的总投票权重。负载越高，候选人越不容易当选，因为算法的目标是均衡负载。
> 5. **迭代选举过程**：
>    - 算法迭代地进行多轮选举。在每一轮中，选择负载最低的候选人作为当选者，然后调整选民的负载以反映当选者的增加。
>    - 在每一轮中，更新每个候选人的负载，负载最低的候选人被选为当选者。
> 6. **负载调整**：
>    - 每当一个候选人被选为当选者，选民的负载也会相应调整，以反映他们对当选者的贡献。这样可以确保选民的总负载保持均衡。
> 7. **重复直到选出所有当选者**：
>    - 这个过程会重复进行，直到选出所有需要的当选者为止。
>
> ### 公平性的实现
>
> Phragmen算法通过以下几种方式实现公平性：
>
> 1. **负载均衡**：
>    - 通过每轮选择负载最低的候选人，并相应调整选民的负载，确保每个候选人的负载尽可能均衡。这避免了单个候选人获得过多选票，从而实现选票的公平分配。
> 2. **多轮迭代**：
>    - 通过多轮迭代，算法可以不断调整选民的负载和候选人的负载，逐步达到负载均衡。这种方法确保了每个选民的投票权重都能得到合理使用。
> 3. **最小负载优先**：
>    - 在每一轮中，总是选择负载最低的候选人，这样可以确保新加入的当选者不会过多增加选民的负载，从而保证负载的均衡。
> 4. **投票权重的分配和调整**：
>    - 选民可以自由分配他们的投票权重，并且在每轮选举后根据当选者的负载调整他们的权重。这种灵活的投票权重分配和调整机制确保了选票分配的公平性。
>
> ### 示例
>
> 假设有三个候选人（A、B、C）和三个选民（V1、V2、V3）。每个选民有1个投票权重，分配如下：
>
> - V1: 0.5给A，0.5给B
> - V2: 0.7给B，0.3给C
> - V3: 1给C
>
> 第一轮选举中：
>
> - A的负载 = 0.5
> - B的负载 = 1.2
> - C的负载 = 1.3
>
> 因此，A被选中。
>
> 调整负载后，进入第二轮：
>
> - V1的负载调整为0.5（A当选）
> - 重新计算B和C的负载：
>   - B的负载 = 1.2 - 0.5 = 0.7
>   - C的负载 = 1.3 - 0 = 1.3
>
> 第二轮选举中，B被选中。
>
> 重复这个过程，直到选出所需的所有当选者。
>
> ### 总结
>
> Phragmen算法通过负载均衡、多轮迭代、最小负载优先和灵活的投票权重分配，实现了选举过程的公平性。这使得在区块链系统中选举验证节点或委员会成员时，可以有效避免集中化和不公平的选举结果。















