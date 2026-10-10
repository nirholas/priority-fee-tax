# PriorityFeeTax

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/priority-fee-tax?style=flat&logo=github)](https://github.com/nirholas/priority-fee-tax/stargazers) [![License](https://img.shields.io/github/license/nirholas/priority-fee-tax?style=flat)](https://github.com/nirholas/priority-fee-tax/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/priority-fee-tax?style=flat)](https://github.com/nirholas/priority-fee-tax/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/priority-fee-tax/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/priority-fee-tax/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Charges a swap in proportion to what it paid the block producer to get where it is in the block.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://priority-fee-tax.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/PriorityFeeTaxHook.sol`](src/hooks/PriorityFeeTaxHook.sol)
- **Licence:** Apache-2.0

## How it works

Position in a block is worth something only to flow that is racing: an arbitrageur closing a gap against a centralized venue, a liquidator, a sandwicher. A person swapping a hundred dollars of one token for another does not care whether they land at index 3 or index 30, and does not bid for it. So the priority fee a transaction attaches is a revealed measure of how much the trade is worth to the trader beyond the trade itself, and that surplus is value the pool's liquidity providers are the counterparty to.

basefee`, the priority fee per unit of gas actually paid, and adds a surcharge along a saturating curve: surcharge(priority) = maxSurcharge * priority / (priority + halfPriority) At `priority == halfPriority` the swap pays half the cap. The surcharge is an LP fee, so it goes to in-range liquidity; the hook takes nothing and holds nothing. The trader cannot dodge it by bidding low, because bidding low is exactly the concession the hook is asking for: a searcher who drops their priority fee to avoid the surcharge loses the race that made the trade profitable.

That is the point. The hook prices the option to be early rather than trying to detect who is early. Prior art: fee mechanisms keyed on realized volatility, on price movement and on swap size are all well covered.

The idea that priority fees reveal flow toxicity is discussed in the ordering-fee literature and in Uniswap's own research on priority-ordering auctions, but the surcharge itself has been implemented at the sequencer or the router, never inside the pool where the liquidity providers who bear the cost can be paid directly. Chain support, stated plainly. This works where there is a real priority-fee market: Ethereum, Base, Unichain, Optimism, Blast and other OP-stack chains.

On Arbitrum One transactions are ordered first-come-first-served and the priority fee is normally zero, so on that chain the hook charges `baseFee` and nothing more. It is safe there, it is simply inert, and a pool on Arbitrum should use {ArbTaxDecayHook} instead.

## Prior art

Fee mechanisms keyed on realized volatility, on price movement and on swap size are all well covered. That priority fees reveal flow toxicity is discussed in the ordering-fee literature and in Uniswap research on priority-ordering auctions, but the surcharge has only ever been implemented at the sequencer or the router, never inside the pool where the liquidity providers who bear the cost can be paid directly.

## Where it does not help

Needs a real priority-fee market. On Arbitrum One, where ordering is first-come-first-served and the priority fee is normally zero, the hook is safe but inert and a pool there should use ArbTaxDecay instead.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    PriorityFeeTaxHook.Config({
        baseFee: /* uint24 */ 0,
        maxSurcharge: /* uint24 */ 0,
        halfPriorityWei: /* uint128 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `baseFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `maxSurcharge` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `halfPriorityWei` | `uint128` | wei per gas |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidHalfPriority()` | `halfPriorityWei` was zero, which would make every non-zero priority fee pay the full surcharge. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SurchargeTooLarge()` | `baseFee + maxSurcharge` must leave room under the 100% protocol maximum. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # PriorityFeeTax
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, dynamic-fee, order-flow, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/priority-fee-tax
cd priority-fee-tax
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/priority-fee-tax&type=Date)](https://www.star-history.com/#nirholas/priority-fee-tax&Date)

<!-- three.ws:growth -->
## Support the project

If priority-fee-tax saves you time, **[star it on GitHub](https://github.com/nirholas/priority-fee-tax)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=priority-fee-tax%3A%20Charges%20a%20swap%20in%20proportion%20to%20what%20it%20paid%20the%20block%20producer%20to%20get%20where%20it%20is%20in%20the%20block&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fpriority-fee-tax) · [Share on Bluesky](https://bsky.app/intent/compose?text=priority-fee-tax%3A%20Charges%20a%20swap%20in%20proportion%20to%20what%20it%20paid%20the%20block%20producer%20to%20get%20where%20it%20is%20in%20the%20block%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fpriority-fee-tax) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fpriority-fee-tax) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fpriority-fee-tax&t=priority-fee-tax%3A%20Charges%20a%20swap%20in%20proportion%20to%20what%20it%20paid%20the%20block%20producer%20to%20get%20where%20it%20is%20in%20the%20block) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fpriority-fee-tax&title=priority-fee-tax%3A%20Charges%20a%20swap%20in%20proportion%20to%20what%20it%20paid%20the%20block%20producer%20to%20get%20where%20it%20is%20in%20the%20block)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/priority-fee-tax` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/priority-fee-tax/issues) or [start a discussion](https://github.com/nirholas/priority-fee-tax/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/priority-fee-tax)](https://github.com/nirholas/priority-fee-tax/graphs/contributors)

<!-- /three.ws:growth -->
