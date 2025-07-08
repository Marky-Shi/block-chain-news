# 区块链项目/协议学习笔记

区块链项目/协议的学习笔记，尽可能的偏向于实用方向，深入分析主流区块链项目的核心技术实现。

## 📚 项目概览

本项目专注于主流区块链项目的技术深度分析，包括共识机制、虚拟机实现、跨链技术等核心组件的源码级解读。

### 🎯 学习目标
- 理解不同区块链项目的技术架构差异
- 掌握共识算法、虚拟机、跨链等核心技术
- 提供实用的技术对比和选型参考

## 📖 内容结构

### 🔗 NEAR Protocol
- [项目简介](./near/简介.md) - 账户模型、分片技术概览
- [共识机制](./near/consensus.md) - Nightshade + Doomslug 共识详解
- [Epoch 管理](./near/epoch.md) - 验证者轮换、奖励分配机制
- [分片技术](./near/shard.md) - 验证者分配、跨分片通信
- [交易生命周期](./near/tx%20apply.md) - 从签名到状态更新的完整流程
- [P2P 网络](./near/p2p.md) - 节点发现算法、Libp2p 应用
- [Staking 机制](./near/staking.md) - 奖励计算、惩罚机制

### 🌌 Cosmos 生态
- [CometBFT 详解](./cosmos/cometbft/overview.md) - 共识引擎、ABCI 接口
- [Ignite CLI](./cosmos/iginte/简介.md) - 一键发链工具使用

### 🔴 Polkadot 生态  
- [Polkadot 2.0](./polkadot/polkdot.md) - JAM、Coretime、PVM 新架构
- [Substrate 框架](./polkadot/polkadot-sdk/) - Runtime 开发、共识实现
- [NPoS 机制](./polkadot/staking%20&&%20npos/) - 提名权益证明详解

### ⚡ Ethereum 生态
- [虚拟机对比](./eth/VM.md) - EVM vs WASM 技术分析  
- [WASM 技术](./eth/wasm.md) - WebAssembly 在区块链中的应用
- [zkEVM](./eth/zkevm.md) - 零知识证明虚拟机实现
- [数据可用性](./eth/eth.md) - EIP-4444、EIP-4844、EthStorage

### 🌐 跨链生态
- [共识机制对比](./ecosystem/consensus.md) - 多链共识算法分析
- [Omni Network](./ecosystem/omini-network.md) - 跨 L2 资产转移协议
- [Halo 共识](./ecosystem/halo.md) - Omni 共识客户端实现
- [跨链中继](./ecosystem/cross-roullp.md) - Relayer 实现机制

## 🛣️ 学习路径建议

### 初学者路径
1. 先阅读各项目的简介文档了解基本概念
2. 重点学习 [NEAR 简介](./near/简介.md) 和 [Polkadot 2.0](./polkadot/polkdot.md)
3. 对比不同项目的 [共识机制](./ecosystem/consensus.md)

### 进阶开发者路径  
1. 深入研究 [CometBFT 实现](./cosmos/cometbft/overview.md)
2. 学习 [NEAR 交易生命周期](./near/tx%20apply.md) 的源码分析
3. 了解 [虚拟机技术对比](./eth/VM.md)

### 架构师路径
1. 系统性学习各项目的技术架构差异
2. 重点关注跨链技术和扩容方案
3. 参考技术选型和性能对比

## 🔄 更新计划

- ✅ 已完成：Polkadot NPoS、Cosmos Staking、NEAR 核心模块
- 🚧 进行中：CometBFT ABCI++、IBC 协议、JAM 技术
- 📋 计划中：Ethereum L2 扩容方案、更多跨链协议分析

## 📊 技术对比速查

| 项目 | 共识算法 | 虚拟机 | 分片 | 跨链 |
|------|----------|--------|------|------|
| NEAR | Nightshade + Doomslug | WASM | ✅ | Receipt |
| Polkadot | GRANDPA + BABE | WASM | ✅ | XCM |
| Cosmos | Tendermint | WASM | ❌ | IBC |
| Ethereum | PoS + LMD-GHOST | EVM | 🚧 | 桥接 |

## 🤝 贡献指南

欢迎提交 Issue 和 PR 来完善内容！请确保：
- 基于真实的源码分析，不添加虚构内容
- 保持技术深度和实用性
- 遵循现有的文档格式和风格
