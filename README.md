# OBIB case 6, as a nuthatch nest

A [nuthatch](https://github.com/nightswatchhq/nuthatch) implementation of **case 6** of Sentio's
[Open Blockchain Indexer Benchmark](https://github.com/sentioxyz/open-blockchain-indexer-benchmark):
the Uniswap V2 factory over blocks 19,000,000 to 19,010,000, discovering pairs from `PairCreated`
and indexing `Swap` on every child it finds.

This exists so the number below can be reproduced rather than believed.

## The result

Median of 5 runs, nuthatch 4.7.0 (the published `aarch64-apple-darwin` release binary), Tenderly's
keyless public gateway `https://mainnet.gateway.tenderly.co`, an 18-core Apple M5 Pro with 48 GB,
2026-10-05:

| | |
|---|---|
| wall clock | **4.67 s** (runs 4.5-5.4 s) |
| events | **35,271** |
| children discovered | **232** |
| RPC requests | **14** |
| peak RSS | **229 MB** (205-251 MB across runs) |

**35,271 = 35,039 `Swap` + 232 `PairCreated`.** The 35,039 is OBIB's expected record count exactly.
The event, child and request counts were the same in all five runs, and none was throttled.

The report is
[`docs/bench/obib-case6-4.7.0-tenderly-2026-10-05.json`](https://github.com/nightswatchhq/nuthatch/blob/main/docs/bench/obib-case6-4.7.0-tenderly-2026-10-05.json)
in the core repo, and
[the note beside it](https://github.com/nightswatchhq/nuthatch/blob/main/docs/bench/obib-case6-4.7.0-tenderly-2026-10-05.md)
records the binary's sha256, the machine, the exact command and every run.

The figure this page used to show, **49.5 s** with 16 requests and 247 MB, is **withdrawn: 1.0.1 on
a closed Alchemy account**. It cannot be rerun by anyone, so it is no longer a claim.

OBIB publishes two different sets of figures for this case. Its January 2026 results table gives
Envio HyperIndex **1.92 min**, Ponder 6.44 min, Subsquid 5.34 min and Sentio 14.36 min; the case-6
page reports Envio at **30 s**, Subsquid 2 min, Sentio and Subgraph 19 min, Ponder 21 min. We quote
both. The 4.67 s above is under both, and that is not a like-for-like ranking: those runs were on
other machines, other days and other endpoints.

One caveat we would rather state than have pointed out: Envio and Subsquid serve this from their own
pre-indexed networks, while nuthatch runs against plain JSON-RPC with no third-party data dependency.
That is the entire design goal, and it cuts both ways.

## Reproduce it

```sh
curl -fsSL https://nuthatch-indexer.com/install.sh | sh   # prebuilt; a source build needs Rust 1.95.0
# the figure above was measured on 4.7.0; this is its exact command
git clone https://github.com/nightswatchhq/obib-case6 && cd obib-case6

export RPC=https://mainnet.gateway.tenderly.co   # keyless; any mainnet archive endpoint works
nuthatch bench backfill --dir . --from 19000000 --to 19010000 --runs 5 --seal-direct --window-adaptive --rpc "$RPC"
```

The run prints a report and writes one with `--out`, carrying provider, hardware and commit. On a
factory nest the bench always adapts its window, so `--window-adaptive` is there to match the recorded
command rather than to change anything. The older artifacts, from the withdrawn Alchemy setup, stay in
the core repo for the record:
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

In August 2026, on the withdrawn Alchemy setup, the same range on the same commit measured anywhere
from 17 s to 57 s depending on when it ran. We checked the obvious explanation instead of assuming it:
re-running against an adjacent, never-fetched range landed in the same band, so provider caching was
not what made the fast runs fast. The event count was invariant across every run, and so was the
request count for a given version: 16 then, 14 on 4.7.0 in all five runs. Those are the honest measure
of range control. Treat any single wall-clock figure from a shared public endpoint, ours included, accordingly.

## Licence

`MIT OR Apache-2.0`, same as nuthatch.
