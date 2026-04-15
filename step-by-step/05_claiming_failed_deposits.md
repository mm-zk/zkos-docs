# Claiming Failed Deposits

When a deposit from L1 to L2 fails (e.g. the L2 transaction runs out of gas), the tokens are locked in the bridge contracts on L1. You can reclaim them by proving on L1 that the L2 transaction failed.

This is common when bridging ERC20 tokens for the first time, as the L2 transaction needs extra gas to deploy the token contract on L2.

## Prerequisites

Make sure you have the standard env variables set up (see [Setup](00_setup.md)):

```shell
export BRIDGEHUB_ADDRESS=...
export L2_CHAIN_ID=...
export L1_RPC=...
export L2_RPC=...
export PRIVATE_KEY=...
```

You'll also need:
- The **L2 transaction hash** of the failed deposit
- The **original deposit parameters** (amount, recipient, token)

## Step 1: Confirm the L2 transaction failed

```shell
cast receipt -r $L2_RPC $L2_TX_HASH
```

Look for `status: 0 (failed)` in the output. Also note the `l2ToL1Logs` section -- the log with `sender: 0x...8001` and `value: 0x00...00` confirms the failure.

## Step 2: Get the L1 Nullifier address

```shell
ASSET_ROUTER=$(cast call -r $L1_RPC $BRIDGEHUB_ADDRESS "assetRouter()(address)")
echo "Asset Router: $ASSET_ROUTER"

L1_NULLIFIER=$(cast call -r $L1_RPC $ASSET_ROUTER "L1_NULLIFIER()(address)")
echo "L1 Nullifier: $L1_NULLIFIER"
```

## Step 3: Get the merkle proof

The L2 batch containing the failed transaction must be finalized on L1 before you can claim. Once it is, request the proof:

```shell
curl -s --request POST --url $L2_RPC \
  --header 'Content-Type: application/json' \
  --data '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "zks_getL2ToL1LogProof",
    "params": ["'$L2_TX_HASH'", 0]
  }' | python3 -m json.tool
```

This returns:
- `batchNumber` -- the L2 batch number
- `proof` -- the merkle proof array
- `id` -- the log index within the batch (used as `l2MessageIndex`)

```shell
export BATCH_NUMBER=...   # from result.batchNumber
export L2_MSG_INDEX=...   # from result.id
export MERKLE_PROOF='[0x..., 0x..., ...]'  # from result.proof
```

If the RPC returns `null`, the batch hasn't been finalized on L1 yet -- wait and retry.

## Step 4: Verify the deposit exists on L1

You can confirm the deposit is recorded in the Nullifier:

```shell
cast call -r $L1_RPC $L1_NULLIFIER "depositHappened(uint256,bytes32)(bytes32)" $L2_CHAIN_ID $L2_TX_HASH
```

This should return a non-zero bytes32 hash. If it returns all zeros, the deposit was never recorded or has already been claimed.

## Step 5: Prepare the asset data

The `assetData` parameter must be the **original deposit calldata** you sent when initiating the bridge -- NOT the enriched data from event logs.

### For ERC20 deposits (via `requestL2TransactionTwoBridges`)

The original data was `abi.encode(amount, recipient, zeroAddress)`:

```shell
ASSET_DATA=$(cast abi-encode "f(uint256,address,address)" $AMOUNT $RECIPIENT 0x0000000000000000000000000000000000000000)
```

### Getting the token's asset ID

If you don't have it, query the Native Token Vault:

```shell
NTV=$(cast call -r $L1_RPC $ASSET_ROUTER "nativeTokenVault()(address)")
ASSET_ID=$(cast call -r $L1_RPC $NTV "assetId(address)(bytes32)" $TOKEN_ADDRESS)
```

### Verifying the hash matches

You can verify your parameters are correct before sending the transaction. The stored hash must equal:

```
keccak256(0x01 ++ abi.encode(sender, assetId, assetData))
```

In Python:
```python
from eth_abi import encode
from web3 import Web3

encoded = encode(['address', 'bytes32', 'bytes'], [sender, asset_id, asset_data])
expected_hash = Web3.keccak(b'\x01' + encoded)
```

Or using cast:
```shell
ENCODED=$(cast abi-encode "f(address,bytes32,bytes)" $SENDER $ASSET_ID $ASSET_DATA)
# Prepend 0x01 and hash
HASH=$(cast keccak 0x01${ENCODED:2})
echo "Computed: $HASH"
```

Compare against the value from Step 4. If they match, you're ready to claim.

## Step 6: Dry-run the claim

Always dry-run first with `cast call`:

```shell
cast call -r $L1_RPC $L1_NULLIFIER \
  "bridgeRecoverFailedTransfer(uint256,address,bytes32,bytes,bytes32,uint256,uint256,uint16,bytes32[])" \
  $L2_CHAIN_ID \
  $SENDER \
  $ASSET_ID \
  $ASSET_DATA \
  $L2_TX_HASH \
  $BATCH_NUMBER \
  $L2_MSG_INDEX \
  0 \
  "$MERKLE_PROOF" \
  --from $SENDER
```

If it returns `0x`, the call will succeed. If it reverts, check:
- `DepositDoesNotExist()` -- your assetData or sender doesn't match the stored hash (see Step 5)
- `InvalidProof()` -- wrong batch number, message index, or merkle proof

## Step 7: Execute the claim

```shell
cast send -r $L1_RPC $L1_NULLIFIER \
  "bridgeRecoverFailedTransfer(uint256,address,bytes32,bytes,bytes32,uint256,uint256,uint16,bytes32[])" \
  $L2_CHAIN_ID \
  $SENDER \
  $ASSET_ID \
  $ASSET_DATA \
  $L2_TX_HASH \
  $BATCH_NUMBER \
  $L2_MSG_INDEX \
  0 \
  "$MERKLE_PROOF" \
  --private-key $PRIVATE_KEY
```

## Step 8: Verify the tokens were returned

```shell
cast call -r $L1_RPC $TOKEN_ADDRESS "balanceOf(address)(uint256)" $SENDER
```

The tokens should now be back in your wallet.

## Complete example

Here's a real example where a DAI deposit to chain 30716 failed due to insufficient L2 gas:

```shell
# Failed L2 tx: 0xb4005b7e8061f7916986e7c798f08647c7d3d9918c9025b4830b667d647c438b
# Original L1 deposit tx: 0x8060d022740b254e66809b621cf5f13920b3b15fde033fec83c0ff09e619b2b5

export L2_TX_HASH=0xb4005b7e8061f7916986e7c798f08647c7d3d9918c9025b4830b667d647c438b
export SENDER=0xC01b665B9d0350b0778beff7ac1273F118e10408
export ASSET_ID=0x28d79ad6f1eceadaec9fa58f9ab505ada95eecf66d29bc404034e9646efd3a84  # DAI
export DAI_AMOUNT=2153924272469592394
export ZERO=0x0000000000000000000000000000000000000000

# Encode the original deposit data
ASSET_DATA=$(cast abi-encode "f(uint256,address,address)" $DAI_AMOUNT $SENDER $ZERO)

# Get proof from L2 RPC
# Result: batchNumber=16, id=4, proof=[...]

# Execute claim
cast send -r $L1_RPC $L1_NULLIFIER \
  "bridgeRecoverFailedTransfer(uint256,address,bytes32,bytes,bytes32,uint256,uint256,uint16,bytes32[])" \
  30716 \
  $SENDER \
  $ASSET_ID \
  $ASSET_DATA \
  $L2_TX_HASH \
  16 \
  4 \
  0 \
  '[0x010f000100000000000000000000000000000000000000000000000000000000,0xe0fd556ac77d64e7b15cfbea85c5fd9a2c801f2dae67f399cb6a7a7b2ec246d8,0x0438cb163f7a4cd5d034d456d0c5b02165c7a9782d0ba30b9d85aa51552d09bc,0xa64245bec806d387bb4ce5624d52a25a2fbbf1d4d817a890541cc7569508e983,0x8e9d697bbe2c229ed639f25bf1baf86edc116f9d231e17c6cd21d990f8cda149,0xe4733f281f18ba3ea8775dd62d2fcd84011c8c938f16ea5790fd29a03bf8db89,0x1798a1fd9c8fbb818c98cff190daa7cc10b6e5ac9716b4a2649f7c2ebcef2272,0x66d7c5983afe44cf15ea8cf565b34c6c31ff0cb4dd744524f7842b942d08770d,0xb04e5ee349086985f74b73971ce9dfe76bbed95c84906c5dffd96504e1e5396c,0xac506ecb5465659b3a927143f6d724f91d8d9c4bdb2463aee111d9aa869874db,0x124b05ec272cecd7538fdafe53b6628d31188ffb6f345139aac3c3c1fd2e470f,0xc3be9cbd19304d84cca3d045e06b8db3acd68c304fc9cd4cbffe6d18036cb13f,0xfef7bd9f889811e59e4076a0174087135f080177302763019adaf531257e3a87,0xa707d1c62d8be699d34cb74804fdd7b4c568b6c1a821066f126c680d4b83e00b,0xf6e093070e0389d2e529d60fadb855fdded54976ec50ac709e3a36ceaa64c291,0x0000000000000000000000000000000000000000000000000000000000000000]' \
  --private-key $PRIVATE_KEY

# Result: 2.15 DAI returned to wallet
```

## Common pitfalls

### `DepositDoesNotExist()` revert

The hash verification failed. The most common cause is using the wrong `assetData`. The Nullifier verifies:

```
keccak256(0x01 ++ abi.encode(sender, assetId, assetData)) == depositHappened[chainId][l2TxHash]
```

The `assetData` must be the **original** `abi.encode(amount, recipient, zeroAddress)` you passed when creating the deposit -- not the enriched bridgeMintCalldata from event logs (which includes token metadata like name, symbol, and decimals).

### `InvalidProof()` revert

- The batch hasn't been finalized on L1 yet
- Wrong `l2MessageIndex` (should be the `id` field from `zks_getL2ToL1LogProof`)
- Wrong `l2TxNumberInBatch` (usually 0 for priority operations)

### Null proof from RPC

If `zks_getL2ToL1LogProof` returns null, the batch containing your failed transaction hasn't been committed and proven on L1 yet. Wait for the sequencer to post the batch.

## Preventing failed deposits

To avoid needing to claim failed deposits:
- Use **5,000,000 L2 gas** for first-time ERC20 deposits (they need to deploy a token contract on L2)
- Use **2,000,000 L2 gas** for subsequent ERC20 deposits
- Use **300,000 L2 gas** for ETH deposits
- Set **800,000 L1 gas limit** for ERC20 bridge transactions (they use ~370k)
