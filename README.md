# OBIB case 6, as a nuthatch nest

A [nuthatch](https://github.com/nightswatchhq/nuthatch) implementation of **case 6** of Sentio's
[Open Blockchain Indexer Benchmark](https://github.com/sentioxyz/open-blockchain-indexer-benchmark):
the Uniswap V2 factory over blocks 19,000,000 to 19,010,000, discovering pairs from `PairCreated`
and indexing `Swap` on every child it finds.

This exists so the number below can be reproduced rather than believed.

## The result

Median of 5 runs, 11-core laptop, Alchemy endpoint, nuthatch 1.0.1:

| | |
|---|---|
| wall clock | **49.5 s** |
| events | **35,271** |
| children discovered | **232** |
| RPC requests | **16** |
| peak RSS | **247 MB** |

**35,271 = 35,039 `Swap` + 232 `PairCreated`.** The 35,039 is OBIB's expected record count exactly.

OBIB publishes two different sets of figures for this case. Its January 2026 results table gives
Envio HyperIndex **1.92 min**, Ponder 6.44 min, Subsquid 5.34 min and Sentio 14.36 min; the case-6
page reports Envio at **30 s**, Subsquid 2 min, Sentio and Subgraph 19 min, Ponder 21 min. We quote
both rather than the flattering one. Against the second, Envio is faster than this.

One caveat we would rather state than have pointed out: Envio and Subsquid serve this from their own
pre-indexed networks, while nuthatch runs against plain JSON-RPC with no third-party data dependency.
That is the entire design goal, and it cuts both ways.

## Reproduce it

```sh
curl -fsSL https://nuthatch-indexer.com/install.sh | sh   # prebuilt; a source build needs Rust 1.95.0
# needs nuthatch >= 2.7.1: `--window-adaptive` is how this artefact was measured.
# `--seal-direct` alone is the fixed-window arm, and a different run.
git clone https://github.com/nightswatchhq/obib-case6 && cd obib-case6

export RPC=https://your-mainnet-archive-endpoint/    # keep the key in your environment, not the repo
nuthatch bench backfill --dir . --from 19000000 --to 19010000 --runs 5 --seal-direct --window-adaptive --rpc "$RPC"
```

The run prints a report and writes one with `--out`, carrying provider, hardware and commit. Our
artifacts live in the core repo:
[`docs/bench/obib-case6.json`](https://github.com/nightswatchhq/nuthatch/blob/main/docs/bench/obib-case6.json)
and a [cold-control run](https://github.com/nightswatchhq/nuthatch/blob/main/docs/bench/obib-case6-cold-control.json).

To index it into a queryable database rather than time it:

```sh
nuthatch dev --dir . --seal-direct --rpc "$RPC"
nuthatch sql "SELECT count(*) FROM pair__swap"
nuthatch sql 'SELECT address, discovered_block FROM "pair__children" LIMIT 5'
```

## How the case is expressed

The whole implementation is the config. There are no handlers.

```toml
[[contracts]]                 # the factory itself
alias = "factory"
address = "0x5c69bee701ef814a2b6a3edd4b1652cb9cc5aa6f"
start_block = 19000000
events = ["PairCreated"]

[[templates]]                 # what a discovered child is
name = "pair"
abi = "abis/pair.json"

[[factories]]                 # the rule that connects them
watch = "factory"
event = "PairCreated"
child_param = "pair"
template = "pair"
start = 19000000
```

Children are discovered at runtime and indexed into shared `pair__*` tables, distinguished by an
`address` column, with a `pair__children` view recording where each came from. No redeploy per child,
and the discovery is rebuilt deterministically from the stored factory events on restart.

## Two things worth knowing if you copy this

- **The vendored `abis/pair.json` is trimmed to `Swap` deliberately.** A `[[templates]]` block has no
  `events` allowlist the way `[[contracts]]` does, so the ABI is the allowlist. With the full
  UniswapV2Pair ABI this nest also decodes `Sync`, `Mint`, `Burn`, `Transfer` and `Approval`, giving
  72,201 rows instead of 35,271. That is a different workload, not a slower one.
- **`block_timestamps = false`.** Case 6 stores no timestamp, and fetching one costs a serial block-header
  round trip per block. On case 1 that column was ~85% of the wall clock. Nuthatch only buys it when a
  nest declares it wants it.

## Wall clock is partly the provider's number

The same range on the same commit measured anywhere from 17 s to 57 s depending on when it ran. We
checked the obvious explanation instead of assuming it: re-running against an adjacent, never-fetched
range landed in the same band, so provider caching is not what made the fast runs fast. The event count
and the 16 RPC requests are invariant across every run, and they are the honest measure of range
control. Treat any single wall-clock figure from a shared public endpoint, ours included, accordingly.

## Licence

`MIT OR Apache-2.0`, same as nuthatch.
