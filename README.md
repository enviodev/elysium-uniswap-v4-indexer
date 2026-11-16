# Every Uniswap v4 pool on Elysium

A [HyperIndex](https://docs.envio.dev/docs/HyperIndex/overview) indexer for every Uniswap v4 pool created on [Elysium](https://elysium.kinetiq.xyz/docs), Kinetiq's Hyperliquid L2 (chain ID 99901). Elysium isn't on HyperSync yet, so this indexer syncs over RPC.

From the PoolManager's deployment on 3 Nov 2026 (block 256,083) to Elysium block 5,448,796 (17:00 UTC on 12 Nov 2026), **115.9K Uniswap v4 pools** were created on Elysium. **94.7% of them use a hook**, and 28.7K were created on 9 Nov alone, the day Elysium opened its public mainnet.

| | Count |
| --- | --- |
| v4 pools | 115,872 |
| Pools with a hook | 109,728 (94.7%) |
| Pools paired with native HYPE | 109,537 (94.5%) |
| Busiest UTC day | 9 Nov 2026, 28,722 pools |
| Blocks covered | 256,083 to 5,448,796 |

For context, here is every v4 `Initialize` event on each chain's PoolManager as of 12 Nov 2026, pulled with HyperSync and again with HyperRPC `eth_getLogs` (both gave the same numbers). Addresses come from Uniswap's [deployments list](https://developers.uniswap.org/deployments.json).

| Chain | v4 pools | With a hook |
| --- | ---: | ---: |
| Ethereum | 198,041 | 24.3% |
| Elysium | 115,872 | 94.7% |
| Arbitrum | 27,037 | 18.3% |
| Unichain | 7,801 | 70.7% |

We also pulled every `Initialize` log on Elysium's PoolManager from its public RPC with `eth_getLogs`, up to the same block. That returned 115,872 pools with the same split for native HYPE pairs and hooks, and the same count for every UTC day. The Elysium explorer API returned 115,872 `Initialize` logs for the same range.

## Why it matters

Hooks are the main new idea in Uniswap v4. They let a pool run its own contract logic around swaps and liquidity. On most chains a minority of pools use one. On Elysium almost every pool does. If you build on Elysium, the hook is the first thing to read on a pool, since it decides how swaps and liquidity behave there, and `HookPools` shows which hooks the most pools rely on.

It also shows how to index a brand new chain on day one. Elysium had no HyperSync support at launch, so the config points HyperIndex at an RPC with `for: sync`. The handler code is the same as it would be on any HyperSync chain.

## What it indexes

| Contract | Chain | Address |
| --- | --- | --- |
| Uniswap v4 PoolManager | Elysium (99901) | `0xE2e105d3F7209A9DA11f83cCf3E7398a753823F1` |

The address is from Uniswap's [v4 deployments page](https://developers.uniswap.org/docs/protocols/v4/deployments), under "Elysium: 99901". The indexer reads one event, `Initialize`, which the PoolManager emits once per pool, so no pool is counted twice. The handler is [`src/handlers/PoolManager.ts`](src/handlers/PoolManager.ts).

Entities in [`schema.graphql`](schema.graphql):

- `Pool` stores one row per pool with its currencies, fee, tick spacing, hook, UTC day and block
- `Totals` keeps pools created, pools paired with native HYPE and pools with a hook
- `DailyPools` counts pools per UTC day
- `HookPools` counts pools per hook address (the zero address means no hook)

A pool counts as paired with HYPE when `currency0` is the zero address, which v4 uses for the chain's native token. HYPE is Elysium's gas token. A pool has a hook when its `hooks` address isn't the zero address.

## Run it

You need Node 22+, pnpm and Docker, plus a free Envio API token from [envio.dev/app/api-tokens](https://envio.dev/app/api-tokens).

```bash
echo 'ENVIO_API_TOKEN=<your-token>' > .env
pnpm install
pnpm codegen
pnpm dev
```

This repo has no tests yet. The agent prompt further down asks your agent to add them, after which `pnpm test` runs them.

The public Elysium RPC is rate limited and an RPC backfill reads a block per pool, so a full backfill on it takes about an hour. Set `ENVIO_ELYSIUM_RPC_URL` in `.env` to your own Elysium RPC to go faster.

GraphQL is at http://localhost:8080 (local admin secret `testing`).

```graphql
{
  Totals { pools poolsPairedWithHype poolsWithHooks lastBlock }
  DailyPools(order_by: { id: asc }) { id pools }
  HookPools(order_by: { pools: desc }, limit: 5) { id pools }
}
```

To pin the count to the block used above, filter `Pool` on `blockNumber`.

```graphql
{
  Pool_aggregate(where: { blockNumber: { _lte: 5448796 } }) { aggregate { count } }
}
```

### Make it live

There's no `end_block` in `config.yaml`, so once the backfill is done the indexer keeps following Elysium's head and your totals keep growing.

### Deploy it

Push your fork to GitHub and deploy it on [Envio Cloud](https://docs.envio.dev/docs/HyperIndex/hosted-service) to get a hosted GraphQL endpoint. Set `ENVIO_ELYSIUM_RPC_URL` as an environment variable on the deployment if you use your own RPC. The [deployment guide](https://docs.envio.dev/docs/HyperIndex/hosted-service-deployment) covers connecting the repo.

## Build your own with an AI agent

This repo doesn't ship Envio's agent skills yet. To add them, run `pnpx envio init` in an empty folder and copy the `.claude/skills/` folder it creates into this repo, or follow [Quickstart with AI](https://docs.envio.dev/docs/HyperIndex/quickstart-with-ai). Then paste a prompt like this into Claude Code or Cursor from the repo root.

```text
This is an Envio HyperIndex indexer for Uniswap v4 pools on Elysium (chain 99901),
synced over RPC. Read README.md, config.yaml, schema.graphql,
src/handlers/PoolManager.ts and any skills in .claude/skills, then
https://docs.envio.dev/docs/HyperIndex/rpc-sync and
https://docs.envio.dev/docs/HyperIndex/testing.

Add a second chain: the Uniswap v4 PoolManager on <CHAIN> (address from
https://developers.uniswap.org/docs/protocols/v4/deployments), from its deployment
block. Key Totals, DailyPools and HookPools by chain so the numbers stay separate.
Add a rollup of pools per fee tier per UTC day. Add vitest with a "test" script
and tests using createTestIndexer for both chains, run `pnpm codegen` and
`pnpm test` until they pass, then run `pnpm dev` and show me the GraphQL query
for pools with a hook per chain.
```

Swap in Unichain, Base or any chain Uniswap v4 is deployed on. Chains that are on HyperSync don't need the `rpc` entry.

## Links

- [HyperIndex overview](https://docs.envio.dev/docs/HyperIndex/overview)
- [Using RPC as your indexing data source](https://docs.envio.dev/docs/HyperIndex/rpc-sync)
- [Testing HyperIndex indexers](https://docs.envio.dev/docs/HyperIndex/testing)
- [Envio Cloud](https://docs.envio.dev/docs/HyperIndex/hosted-service)
- [Quickstart with AI](https://docs.envio.dev/docs/HyperIndex/quickstart-with-ai)
- [Uniswap v4 docs](https://developers.uniswap.org/docs/protocols/v4/overview)
- [Uniswap v4 deployments](https://developers.uniswap.org/docs/protocols/v4/deployments)
- [Elysium docs](https://elysium.kinetiq.xyz/docs)
- [Elysium contract addresses](https://elysium.kinetiq.xyz/docs/contract-addresses)
- [Kinetiq](https://kinetiq.xyz)
- [HyperEVM docs](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperevm)
- [HYPE deposits into Elysium](https://github.com/enviodev/elysium-hype-deposits-indexer), a sister indexer on the HyperEVM side
- [Envio Discord](https://discord.gg/envio)
