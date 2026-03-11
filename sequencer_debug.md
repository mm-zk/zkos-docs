# Debugging sequencer issues

If sequencer is alive, get the chain id (`cast chain-id -r $RPC`) and get Bridgehub address.

Getting bridgehub address:

```
curl --request POST \
  --url $RPC \
  --header 'Content-Type: application/json' \
  --data '{
      "jsonrpc": "2.0",
      "id": 1,
      "method": "zks_getBridgehubContract",
      "params": []
    }'
```


Then you can query that address to get the zk chain id - and you can check the most recent calls.
These calls are coming from some (or multiple) operator addresses.

Check that they have enough balance.
