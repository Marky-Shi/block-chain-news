## Relayer

按照omini的架构设定，roullpA 调用portal contract 生成xmsg ---> halo client 进行共识处理，然后移交给 relayer 进行转发，实现cross-roullp的目的。



relayer/app/app.go

```go
func Runctx context.Context, cfg Config) error{
  ... 
  do some initialilze works
  ...
  // privateKey, cprovider xprovider db etc...
  
  for _, destChain := range network.EVMChains() {
    
    // setup send provider
    sendProvider:= func() (SendAsync, error) {...}
    
    // Setup validator set awaiter
    portal, err := bindings.NewOmniPortal(destChain.PortalAddress, rpcClientPerChain[destChain.ID])
    awaitValSet := newValSetAwaiter(portal, destChain.BlockPeriod)
    
    //setup && start worker
    worker := NewWorker(
			destChain,
			network,
			cprov,
			xprov,
			CreateSubmissions,
			sendProvider,
			awaitValSet,
			cursors,
		)

		go worker.Run(ctx)
  }
}
```

Relayer/app/worker.go

```go
func (w *Worker) runOnce(ctx context.Context) error {
  ... do some initialize works
  
  // sub different chains && setup callback 
  // 设置回调函数用于接收/处理来自不同链的消息
  var logAttrs []any //nolint:prealloc // Not worth it
	for chainVer, fromOffset := range attestOffsets {
		if chainVer.ID == w.destChain.ID { // Sanity check
			return errors.New("unexpected chain version [BUG]")
		}

		callback := w.newCallback(
			msgFilter,
			buf.AddInput,
			newMsgStreamMapper(w.network),
			chainVer,
		)

		w.cProvider.StreamAsync(ctx, chainVer, fromOffset, w.destChain.Name, callback)

		logAttrs = append(logAttrs, w.network.ChainVersionName(chainVer), fromOffset)
	}

	log.Info(ctx, "Worker subscribed to chains", logAttrs...)

	return buf.Run(ctx)
}
```

```go
func (w *Worker) newCallback(
	msgFilter *msgCursorFilter,
	sendBuffer func(context.Context, xchain.Submission) error,
	msgStreamMapper msgStreamMapper,
	streamerChainVer xchain.ChainVersion, // Use streamer chain version for cursors since fuzzy overrides otherwise store latest streamed offsets to finalized streamer.
) cchain.ProviderCallback {
  ...
  return func(ctx context.Context, att xchain.Attestation) error {
    
    // 同步xblock
    block, ok, err := fetchXBlock(ctx, w.xProvider, att)
    
    // 构建xmsg tree
    msgTree, err := xchain.NewMsgTree(block.Msgs)
    
    submitted := make(map[xchain.StreamID][]xchain.Msg)
    // Split into streams
		for streamID, msgs := range msgStreamMapper(block.Msgs) {
    	// Filter out any previously submitted message offsets
      // 过滤已提交的消息
			msgs, err = filterMsgs(ctx, streamID, w.network.StreamName, msgs, msgFilter)
      
      update := StreamUpdate{
				StreamID:    streamID,
				Attestation: att,
				Msgs:        msgs,
				MsgTree:     msgTree,
				ValSet:      cachedValSet,
			}

			submissions, err := w.creator(update)
      for _, subs := range submissions {
				if err := sendBuffer(ctx, subs); err != nil {
					return err
				}
			}
			submitted[streamID] = msgs
      
    }
  }
  
}
```

```go
// Run processes the buffer, sending submissions to the async sender.
func (b *activeBuffer) Run(ctx context.Context) error {
  for {
    select {
      ...
      case submission := <-b.buffer:
      	
      	mempoolLen.WithLabelValues(b.chainName).Inc()
      	// // Trigger async send synchronously (for ordered nonces), but wait for response async.
      // sendsync 方法则是定义在 sender.go里
      	response := b.sendAsync(ctx, submission)
			go func() {
				err := <-response
				if err != nil {
					b.submitErr(errors.Wrap(err, "send submission"))
				}

				mempoolLen.WithLabelValues(b.chainName).Dec()
				sema.Release(1)
			}()
      
    }
  }
}
```

而后续的处理工作则是又交给了portal.sol 进行处理。

```go
func (s Sender) SendAsync(ctx context.Context, sub xchain.Submission) <-chan error {
  ... do somecheck && setup tx
  
  estimatedGas := s.gasEstimator(s.chain.ID, sub.Msgs)

	candidate := txmgr.TxCandidate{
		TxData:   txData,
		To:       &s.chain.PortalAddress,// ✅
		GasLimit: estimatedGas,
		Value:    big.NewInt(0),
		Nonce:    &nonce,
	}
  
  go func() {
    // call ethclient SendTransaction
    tx, rec, err := s.txMgr.Send(ctx, candidate)
    
    receiptAttrs := []any{
			"valset_id", sub.ValidatorSetID,
			"status", rec.Status,
			"nonce", tx.Nonce(),
			"height", rec.BlockNumber.Uint64(),
			"gas_used", rec.GasUsed,
			"tx_hash", rec.TxHash,
		}
    // 交易回滚？
    const statusReverted = 0
    if rec.Status == statusReverted {
      s.ethCl.CallContract(ctx, callFromTx(s.txMgr.From(), tx), rec.BlockNumber)
      errAttrs := slices.Concat(receiptAttrs, reqAttrs, []any{
				"call_resp", hexutil.Encode(resp),
				"call_err", err,
				"gas_limit", tx.Gas(),
			})
    }
    log.Info(ctx, "Sent submission", receiptAttrs...)
		asyncResp <- nil
  }()
  
  return asyncResp
}
```

