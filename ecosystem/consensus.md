## Consensus module 

Path: lib/cchain

## source code

### contract

OmniPortal.sol

```solidity
struct Msg {
        uint64 sourceChainId;
        uint64 destChainId;
        uint64 shardId;
        uint64 offset;
        address sender;
        address to;
        bytes data;
        uint64 gasLimit;
    }
    
struct Submission {
        bytes32 attestationRoot;
        uint64 validatorSetId;
        BlockHeader blockHeader;
        Msg[] msgs;
        bytes32[] proof;
        bool[] proofFlags;
        SigTuple[] signatures;
    }

event XReceipt(
        uint64 indexed sourceChainId,
        uint64 indexed shardId,
        uint64 indexed offset,
        uint256 gasUsed,
        address relayer,
        bool success,
        bytes error
    );
```



```go
type XMsg struct {
	Chainer

	ID            graphql.ID
	Block         XBlock
	To            common.Address
	Data          hexutil.Bytes
	DestChainID   hexutil.Big
	GasLimit      hexutil.Big
	DisplayID     string
	Offset        hexutil.Big
	Receipt       *XReceipt
	Sender        common.Address
	SourceChainID hexutil.Big
	Status        Status
	TxHash        common.Hash
}

type XReceipt struct {
	Chainer
	ID            graphql.ID
	GasUsed       hexutil.Big
	Success       bool
	Relayer       common.Address
	SourceChainID hexutil.Big
	DestChainID   hexutil.Big
	Offset        hexutil.Big
	TxHash        common.Hash
	Timestamp     graphql.Time
	RevertReason  *string
}

type XBlock struct {
	Chainer
	ID        graphql.ID
	ChainID   hexutil.Big
	Height    hexutil.Big
	Hash      common.Hash
	Messages  []XMsg
	Timestamp graphql.Time
}
```
