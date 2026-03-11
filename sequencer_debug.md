# Debugging sequencer issues

## Step 1: Check if sequencer is alive

Get the chain id and latest block:

```bash
cast chain-id --rpc-url $RPC
cast block latest --rpc-url $RPC
```

Compare the latest block's `timestamp` to current time. If the block is stale (e.g. minutes/hours old), the sequencer has stopped producing blocks.

## Step 2: Get Bridgehub address

```bash
curl -s --request POST --url $RPC \
  --header 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"zks_getBridgehubContract","params":[]}'
```

## Step 3: Get L1 chain info

The L2 RPC may have limited `zks_` methods available. These methods may return "Method not found":
- `zks_L1ChainId`
- `zks_getBlockDetails`
- `zks_getL1BatchDetails`
- `zks_L1BatchNumber`

If so, you need to determine the L1 network separately (e.g. ask the user or check docs).

## Step 4: Get the ZKChain diamond proxy address on L1

Using the Bridgehub address and chain ID on the **L1 RPC**:

```bash
cast call $BRIDGEHUB "getZKChain(uint256)(address)" $CHAIN_ID --rpc-url $L1_RPC
```

## Step 5: Get the Chain Type Manager (CTM)

```bash
cast call $BRIDGEHUB "chainTypeManager(uint256)(address)" $CHAIN_ID --rpc-url $L1_RPC
```

## Step 6: Get the ValidatorTimelock

Post v29, the validator timelock is on the CTM:

```bash
cast call $CTM "validatorTimelockPostV29()(address)" --rpc-url $L1_RPC
```

Note: `validatorTimelock()` on CTM may return `0x0` — use `validatorTimelockPostV29()` instead.

## Step 7: Check batch settlement status

Query the diamond proxy on L1 for committed, proven, and executed batches:

```bash
cast call $DIAMOND "getTotalBatchesCommitted()(uint256)" --rpc-url $L1_RPC
cast call $DIAMOND "getTotalBatchesVerified()(uint256)" --rpc-url $L1_RPC
cast call $DIAMOND "getTotalBatchesExecuted()(uint256)" --rpc-url $L1_RPC
```

If all three numbers are equal, the pipeline is fully drained — no pending batches.

## Step 8: Identify the L1 operator addresses

**Important:** The L2 `miner` address (from block headers) is the L2 fee account, NOT the L1 operator. L1 operators may be completely different addresses.

There are typically **3 separate operator roles** on L1, each potentially a different address:
- **Commit** — calls `commitBatchesSharedBridge`
- **Prove** — calls `proveBatchesSharedBridge`
- **Execute** — calls `executeBatchesSharedBridge`

These operators submit transactions to the **ValidatorTimelock** (not the diamond proxy directly).

To find them, query the timelock's transaction history on a block explorer (e.g. Blockscout):

```bash
curl -s "https://eth-sepolia.blockscout.com/api/v2/addresses/$TIMELOCK/transactions?filter=to" | python3 -m json.tool
```

Look at the `from` addresses and `method` fields to identify which address handles which role. Some deployments use a single address for all three, others use separate ones.

## Step 9: Check operator L1 balances

```bash
cast balance $COMMIT_OPERATOR --rpc-url $L1_RPC --ether
cast balance $PROVE_OPERATOR --rpc-url $L1_RPC --ether
cast balance $EXECUTE_OPERATOR --rpc-url $L1_RPC --ether
```

If any operator is out of L1 ETH, it cannot submit transactions and the pipeline stalls.

## Step 10: Get the verifier address

```bash
cast call $DIAMOND "getVerifier()(address)" --rpc-url $L1_RPC
```

## Common issues

- **Operator out of L1 ETH** — the most common cause. Fund the operator address with L1 ETH.
- **Sequencer process crashed** — L2 stops producing blocks but L1 operators may still have funds. Check server logs.
- **L1 batches stuck** — if committed > executed, the prove or execute step is failing. Check those operators' recent transactions for reverts.
- **All batches settled but L2 stalled** — sequencer issue, not an L1 settlement issue.

## Notes on public RPCs

- `rpc.sepolia.org` can be unreliable. Alternatives: `ethereum-sepolia-rpc.publicnode.com`
- Etherscan V1 API is deprecated; use V2 with an API key, or use Blockscout API (no key needed).
