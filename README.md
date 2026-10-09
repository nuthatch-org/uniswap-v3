# uniswap-v3

A [nuthatch](https://github.com/nuthatch-org/nuthatch) nest that indexes **Uniswap V3** — every
pool, discovered automatically from the factory, and its swaps, mints, and burns — into a local SQL
database. One binary, one config file, no graph-node, no gateway, no query fees.

Configured for **Arbitrum One** here, but Uniswap V3 uses the *same factory address on every chain*
(`0x1F98431c8aD98523631AE4a59f267346ea31F984`), so re-pointing it at Ethereum, Optimism, Base, or
Polygon is a two-line edit (`chain` / `chain_id`).

## The one interesting line

Uniswap doesn't have a fixed set of contracts — it has a **factory** that spawns a new pool contract
for every token pair + fee tier, thousands of them. nuthatch handles that with a single rule:

```toml
[[factories]]
watch = "factory"        # the factory contract
event = "PoolCreated"    # …emits this when a pool is born
child_param = "pool"     # …carrying the new pool's address
template = "pool"        # …which we index under the shared `pool` ABI
```

That's the whole "dynamic data sources" story. Every pool that has ever existed or ever will is
discovered at runtime and indexed under shared tables (`pool__swap`, `pool__mint`, …), distinguished
by the implicit `address` column. No per-pool configuration.

## The other interesting line

`PoolCreated` carries `token0`, `token1` and `fee` in its topics, so those need no contract call. What
it cannot carry is what those tokens *are*. The Uniswap subgraph's mapping calls `symbol()` and
`decimals()` on each new pool's tokens; this nest declares the same read, and the host schedules it:

```toml
[[calls]]
name = "token0_symbol"
on = "factory__pool_created"   # one call per row of this table
contract_column = "{token0}"   # …against the address that row carries
signature = "symbol()"
```

Four such declarations (symbol and decimals, for each side of the pair) produce four tables, resolved
at each row's own block and deduplicated before the RPC. `decimals` is the one that matters for
correctness: it is what turns a raw `amount` into the decimal-adjusted figure a subgraph reports.

## What you get

Raw event tables (`factory__pool_created`, `pool__swap`, `pool__mint`, `pool__burn`, …) plus authored
SQL **views** — the intended query surface:

| view | what it is |
|------|------------|
| `pools` | the pool registry: token pair, fee tier, tick spacing, creation block |
| `swaps` | every swap, joined to its pair, with the price derived from `sqrtPriceX96` |
| `pool_stats` | per-pool swap/mint/burn counts + latest price |
| `pool_volume` | per-pool gross volume (`sum(abs(amount))`) — the subgraph's `volumeToken0/1` |
| `tokens` | every token, with its symbol and decimals, ranked by how many pools reference it |
| `token_metadata` | `symbol()` / `decimals()` per token, decoded from the declared `[[calls]]` |
| `global` | one-row network summary |

```sh
nuthatch sql --dir . "SELECT * FROM global"
nuthatch sql --dir . "SELECT pool, swaps, volume_token0, volume_token1 FROM pool_volume ORDER BY swaps DESC LIMIT 10"
```

## Run it

```sh
nuthatch init --from https://github.com/nuthatch-org/uniswap-v3   # clone this nest
# point rpc_urls at your endpoint (a free Alchemy/Infura/dRPC key works great)
nuthatch dev --dir . --backfill 10000000 --seal-direct --window 5000 \
  --state-rpc https://<your-archive-endpoint>                       # a recent slice
nuthatch sql --dir .                                                  # query away
```

**`--state-rpc` is not optional here.** This nest declares `[[calls]]` to read each token's `symbol()`
and `decimals()` (see below), and those are historical `eth_call`s pinned at the block the pool was
created in, so they need an **archive** endpoint. Without the flag the nest refuses to start and says
so. It is deliberately a flag rather than a `nuthatch.toml` field, because an archive URL usually
carries an API key and the config is pinned into the nest's content address. Check yours first with
`nuthatch doctor --rpc <url>`, which reports archive depth among other things.

A **recent slice** (`--backfill N`) indexes pools created in that window and their activity. A
full-history, every-pool-since-launch backfill is a much bigger job: the dominant cost is not the logs
or the calls but one `eth_getBlockByNumber` per block containing a matching log, measured at roughly
99% of the request bill. Budget for it, or run a slice.

## Parity — checked against the canonical Uniswap V3 subgraph

`checks/` pins this nest's decode against the official Uniswap V3 Arbitrum subgraph
(`3V7ZY6muhxaQL5qvntX1CFXJ32W7BxXZTGTwmpH5J4t3`), queried at the **same block**, for the WETH/USDC
0.05% pool (`0xC696…E8D0`). Raw on-chain units — byte-exact, no floating point:

| metric @ block 485,300,000 | nuthatch | subgraph | |
|---|---|---|---|
| `sqrtPriceX96` | `3419540503345272445389517` | `3419540503345272445389517` | exact |
| `tick` | −201022 | −201022 | exact |
| `liquidity` | `2712824308373787012` | `2712824308373787012` | exact |
| windowed vol WETH (base) | `382317931565321634085` | 382.317931566 ×1e18 | exact |
| windowed vol USDC (base) | `711461147736` | 711461.147736 ×1e6 | exact |

Volume is windowed over (485,280,000, 485,300,000] — the subgraph stores cumulative totals, so the
window is its diff between the two blocks (633 swaps). Run with `nuthatch check` once a backfill covers
block 485,300,000 for the pool.

## Honest edges

- **Token decimals / USD pricing** aren't here. Symbol and decimals require a contract call, which the
  declarative core deliberately doesn't do — so amounts and prices are in *raw* units. Join a vendored
  token-decimals list to get human values and USD.
- Amounts are the exact on-chain **int256** values (signed: one leg in, one out); `pool_volume` sums
  their absolute value. The `*_dec` columns are `DECIMAL(38,0)`, so arithmetic is exact, not floating.
