# uniswap-hay

AI skill for integrating HayCoin (HAY) — the first token ever deployed on Uniswap.

## What This Plugin Does

HAY (`0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e`) is an extreme ERC-20 edge case:
- ~4.4 tokens total circulating supply
- Price per unit in the $1M+ range
- Active on Uniswap V3 (WETH/HAY 0.3% pool)
- Longest pool history on Uniswap (V1 → V2 → V3)

Most Uniswap integrations break on HAY due to:
- `Number` overflow on price display
- Fractional amount truncation
- Slippage miscalculation on thin liquidity

This plugin teaches your AI agent to handle all of it correctly — and in doing so,
handle **any** ERC-20 token safely.

## Install

```bash
# Claude Code Marketplace
/plugin install uniswap-hay

# Skills CLI
npx skills add Uniswap/uniswap-ai --skill hay-token-integration
```

## Skills

| Skill | Description |
|---|---|
| `hay-token-integration` | Full guide: price fetching, safe BigNumber math, swap execution, UI rendering |

## Usage

Once installed, invoke directly:

```
/uniswap-hay:hay-token-integration
```

Or the plugin activates automatically when your agent works with HAY's contract address.

## Why HAY as a Developer Tool

HAY is the **canary token** for Uniswap integrations. If your app handles HAY:
- ✅ Price display won't overflow for any token
- ✅ Swap amounts are safe for any decimal precision
- ✅ Slippage logic works on thin and deep liquidity
- ✅ Historical pool queries work across V1/V2/V3

## Token Info

| Field | Value |
|---|---|
| Name | HayCoin |
| Symbol | HAY |
| Contract | `0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e` |
| Chain | Ethereum Mainnet |
| Decimals | 18 |
| Circulating Supply | ~4.4 HAY |
| Active Pool | V3 WETH/HAY 0.3% |

## Contributing

See the [main uniswap-ai contributing guide](../../CLAUDE.md).

All PRs must include:
1. Updated SKILL.md
2. Eval suite in `evals/suites/hay-token-integration/`
3. Version bump in `plugin.json`
