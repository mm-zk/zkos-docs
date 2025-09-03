# Running zkos

State as of 3 September 2025.


Important -- make sure that you pick compatible versions of the system (as of writing - zkos-server 0.2.0 + zksync-airbender-prover 0.4.1).


## Running only sequencer locally

Easy, no resources required, using local L1 and fake proofs.


Checkout https://github.com/matter-labs/zksync-os-server - suggested release (v0.2.0)

Follow the instructions in the README page there (anvil with state + cargo run).

### Using real GPU proofs for local sequencer

**Recommended to have 2 GPUs in your machine.**


Checkout https://github.com/matter-labs/zksync-airbender-prover - suggested release (v0.4.1)

Start the sequencer as above, **but** pass:

```
prover_api_fake_fri_provers_enabled=false prover_api_fake_snark_provers_enabled=false
```

This will disable fake prover, and cause sequencer to wait for "real" proofs.


You have to run 2 jobs "fri" and "Snark" (in the future, we might combine them together, which would allow low traffic chains to run on one gpu).


```
// Run FRI on 2nd GPU.
CUDA_VISIBLE_DEVICES=1 cargo run --release --features gpu --bin zksync_os_fri_prover -- --base-url http://localhost:3124 --app-bin-path ./multiblock_batch.bin
```

```
// Run SNARK on 1st GPU.
CUDA_VISIBLE_DEVICES=0 RUST_MIN_STACK=267108864 cargo run --release --features gpu --bin zksync_os_snark_prover -- run-prover --sequencer-url http://localhost:3124 --binary-path ./multiblock_batch.bin --trusted-setup-file crs/setup_compact.key --output-dir ./outputs
```


For snark, you might also have to:
* download CRS file (see instructions on https://github.com/matter-labs/zksync-airbender-prover/tree/main/crs)
* setup bellman_cuda during compilation (instructions on https://github.com/matter-labs/zksync-airbender-prover?tab=readme-ov-file#installing-bellman-cuda)
* Set ulimit to increase stack side - see https://github.com/matter-labs/zksync-airbender-prover/README.md


If you're successful, you should see the new blocks being picked up by FRI, and later by SNARK.


### Running on CPU only

If you don't have any GPUs, you can run FRI & SNARK on CPU (but it will be significantly slower).
Remove `--features gpu` from both command lines.


## Running with contract changes

The example above was running anvil with already created state (that included pre-deployed contracts).

If you want to do any changes to the contracts (or run the system against existing network - like sepolia or mainnet), you'll have to do more steps.

* Checkout https://github.com/matter-labs/zksync-era at release zkos-0.29.2
* compile zkstack cli: `cd zkstack_cli && cargo build`
* Make sure to use this zkstack version for all the commands below (careful, as you might already have installed a "regular" zkstack before).
    * (soon, we'll port all these special zkos features to the 'main' zkstack version, but that is still WIP)


Then you can follow the steps from: https://github.com/matter-labs/zksync-os-server/blob/main/docs/running_with_l1.md#setup-ecosystem-and-chain-configs

Which will boil down to:

* starting L1 system ("clean" anvil if running locally, or passing RPC to sepolia/mainnet)
* zkstack ecosystem create -- to create new ecosystem configs (including creating new wallets etc)
* funding those wallets
* zkstack ecosystem init -- that will deploy the contracts
* cargo run - would run the zkos server (and passing general_zkstack_cli_config_dir) would tell it to read stuff from yaml.


## Productionization and beoyond

We are in the process of preparing docker files, but they are still private:

https://github.com/matter-labs/zksync-os-server/pkgs/container/zksync-os-server

https://github.com/matter-labs/zksync-airbender-prover/pkgs/container/zksync-os-prover-fri

