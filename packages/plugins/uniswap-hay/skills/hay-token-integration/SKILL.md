---
name: hay-token-integration
description: >
  Guide developers on integrating, querying, and handling HayCoin (HAY) —
  the first token ever deployed on Uniswap (0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e).
  HAY is an extreme edge-case ERC-20: ultra-low circulating supply (~4.4 tokens),
  very high price per unit, and V1/V2/V3 pool history. This skill teaches developers
  how to safely fetch price data, handle fractional amounts, prevent UI overflow,
  and integrate HAY swaps via the Uniswap Trading API.
plugin: uniswap-hay
version: 0.1.0
---

# HAY Token Integration Skill

## Overview

HayCoin (HAY) is the **first token ever deployed on Uniswap**, created by Hayden Adams
in 2018 for testing Uniswap v1 before its launch. It has migrated through V1 → V2 → V3
and is actively traded today.

**Contract address (Ethereum Mainnet):**
`0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e`

HAY is a live stress-test for any Uniswap integration. If your code handles HAY correctly,
it handles all ERC-20 edge cases correctly.

---

## Why HAY Breaks Most Integrations

HAY exposes three common developer bugs:

| Problem | Root Cause | Symptom |
|---|---|---|
| Price display overflow | `BigNumber` stored as `number` | `Infinity` or `NaN` in UI |
| Swap amount underflow | Fractional token math truncated | Failed transactions |
| Slippage miscalculation | Thin liquidity + high unit price | Reverted swaps |

---

## Step 1 — Fetch HAY Price Data

Use the Uniswap Trading API or Subgraph. Never use `parseFloat()` on HAY prices —
always use `BigInt` or a BigNumber library.

### Via Uniswap Trading API

```typescript
import { createPublicClient, http, formatUnits } from 'viem'
import { mainnet } from 'viem/chains'

const HAY_ADDRESS = '0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e'
const WETH_ADDRESS = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'

// Fetch quote: how much WETH for 0.01 HAY
const response = await fetch(
  `https://api.uniswap.org/v2/quote?` +
  `tokenInAddress=${HAY_ADDRESS}` +
  `&tokenInChainId=1` +
  `&tokenOutAddress=${WETH_ADDRESS}` +
  `&tokenOutChainId=1` +
  `&amount=10000000000000000` + // 0.01 HAY in wei (18 decimals)
  `&type=exactIn`,
  {
    headers: {
      'x-api-key': process.env.UNISWAP_API_KEY!,
    },
  }
)

const quote = await response.json()
```

### Via The Graph (Subgraph)

```graphql
{
  token(id: "0xfa3e941d1f6b7b10ed84a0c211bfa8aee907965e") {
    symbol
    decimals
    derivedETH
    totalSupply
    totalValueLockedUSD
    poolCount
  }
}
```

> **Note:** HAY address in subgraph queries must be **lowercase**.

---

## Step 2 — Handle Fractional Amounts Safely

HAY has 18 decimals like standard ERC-20 tokens, but its total supply is ~95 HAY.
Users will always trade **fractions** (e.g., 0.001 HAY). Use `parseUnits` and
`formatUnits` from viem — never raw `Number()` conversion.

```typescript
import { parseUnits, formatUnits } from 'viem'

// CORRECT: parse user input safely
const amountIn = parseUnits('0.001', 18) // → 1000000000000000n

// WRONG: loses precision
const amountIn = Number('0.001') * 1e18 // → floating point error
```

---

## Step 3 — Prevent UI Price Display Overflow

HAY price per unit can exceed $1,000,000. Standard UI components that use
`toFixed(2)` or `toLocaleString()` without bounds checking will break.

```typescript
// Safe price formatter for extreme-value tokens
function formatTokenPrice(priceUSD: bigint, decimals: number): string {
  const formatted = formatUnits(priceUSD, decimals)
  const num = parseFloat(formatted)

  if (num >= 1_000_000) return `$${(num / 1_000_000).toFixed(2)}M`
  if (num >= 1_000) return `$${(num / 1_000).toFixed(2)}K`
  return `$${num.toFixed(6)}`
}
```

---

## Step 4 — Execute a HAY Swap via Universal Router

HAY trades on Uniswap V3 (0.3% fee pool with WETH). Use the Trading API
to generate a valid calldata — do not hardcode routing.

```typescript
// 1. Get a swap quote with routing
const quoteResponse = await fetch('https://api.uniswap.org/v2/quote', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': process.env.UNISWAP_API_KEY!,
  },
  body: JSON.stringify({
    tokenIn: WETH_ADDRESS,
    tokenOut: HAY_ADDRESS,
    chainId: 1,
    amount: parseUnits('0.01', 18).toString(), // 0.01 WETH
    type: 'EXACT_INPUT',
    recipient: userAddress,
    slippageTolerance: 5, // 5% — HAY has thin liquidity, set higher than usual
  }),
})

const { quote, calldata, value } = await quoteResponse.json()

// 2. Send via Universal Router
const tx = await walletClient.sendTransaction({
  to: UNIVERSAL_ROUTER_ADDRESS,
  data: calldata,
  value: BigInt(value),
})
```

> **⚠ Slippage Warning:** HAY liquidity is thin. Use at least 3–5% slippage tolerance
> or the transaction will revert. Always warn users before execution.

---

## Step 5 — HAY on Uniswap V3 (Active Pool)

HAY is actively trading on Uniswap V3 on Ethereum mainnet. This section covers
everything specific to the V3 pool: how to look up the pool address on-chain,
read live state, interpret V3's `sqrtPriceX96` for a token with extreme unit price,
and query historical data.

### 5a — Look Up the HAY/WETH V3 Pool Address

Never hardcode the pool address. Derive it on-chain from the V3 factory.

```typescript
import { createPublicClient, http, getContract } from 'viem'
import { mainnet } from 'viem/chains'

const HAY_ADDRESS  = '0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e'
const WETH_ADDRESS = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'
const V3_FACTORY   = '0x1F98431c8aD98523631AE4a59f267346ea31F984'
const FEE_TIER     = 3000 // 0.3%

const FACTORY_ABI = [
  {
    name: 'getPool',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'tokenA', type: 'address' },
      { name: 'tokenB', type: 'address' },
      { name: 'fee',    type: 'uint24'  },
    ],
    outputs: [{ name: 'pool', type: 'address' }],
  },
] as const

const client = createPublicClient({ chain: mainnet, transport: http() })

const factory = getContract({ address: V3_FACTORY, abi: FACTORY_ABI, client })

const hayPoolAddress = await factory.read.getPool([
  HAY_ADDRESS,
  WETH_ADDRESS,
  FEE_TIER,
])

console.log('HAY/WETH V3 pool:', hayPoolAddress)
// Returns the live pool address on mainnet
```

### 5b — Read Live V3 Pool State

```typescript
const POOL_ABI = [
  {
    name: 'slot0',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [
      { name: 'sqrtPriceX96', type: 'uint160' },
      { name: 'tick',         type: 'int24'   },
      { name: 'observationIndex',             type: 'uint16'  },
      { name: 'observationCardinality',       type: 'uint16'  },
      { name: 'observationCardinalityNext',   type: 'uint16'  },
      { name: 'feeProtocol', type: 'uint8'   },
      { name: 'unlocked',    type: 'bool'    },
    ],
  },
  {
    name: 'liquidity',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint128' }],
  },
] as const

const pool = getContract({ address: hayPoolAddress, abi: POOL_ABI, client })

const [slot0, liquidity] = await Promise.all([
  pool.read.slot0(),
  pool.read.liquidity(),
])

console.log('sqrtPriceX96:', slot0.sqrtPriceX96)
console.log('tick:',         slot0.tick)
console.log('liquidity:',    liquidity)
```

### 5c — Decode sqrtPriceX96 for HAY

V3 encodes price as `sqrtPriceX96`. For a token with HAY's extreme unit price,
decoding this requires careful BigInt math — never cast to `Number` mid-calculation.

```typescript
// Decode sqrtPriceX96 → price of token0 in terms of token1
function decodeSqrtPriceX96(sqrtPriceX96: bigint): bigint {
  // price = (sqrtPriceX96 / 2^96)^2
  // Use BigInt throughout — intermediate values overflow Number
  const Q96 = 2n ** 96n
  const price = (sqrtPriceX96 * sqrtPriceX96) / (Q96 * Q96)
  return price
}

// Determine which token is token0 (lower address alphabetically)
// HAY: 0xfA3E... vs WETH: 0xC02a...
// 0xC02a < 0xfA3E → WETH is token0, HAY is token1
// So slot0 price = WETH per HAY (need to invert for HAY/WETH)

const rawPrice = decodeSqrtPriceX96(slot0.sqrtPriceX96)

// Since both tokens have 18 decimals, no decimal adjustment needed
// rawPrice is already in WETH-per-HAY units as a ratio
console.log('Approximate WETH per HAY:', rawPrice)
```

> **⚠ Token Order Matters:** In V3, `token0` is always the address that sorts lower
> alphabetically. WETH (`0xC02a...`) < HAY (`0xfA3E...`), so WETH = token0, HAY = token1.
> Always verify token order before interpreting price direction.

### 5d — Query V3 Pool History via Subgraph

```graphql
# Get HAY/WETH pool daily data (last 30 days)
# Use lowercase address for subgraph
{
  pool(id: "<hay-weth-v3-pool-address-lowercase>") {
    token0 { symbol }
    token1 { symbol }
    feeTier
    liquidity
    token0Price
    token1Price
    totalValueLockedUSD
    volumeUSD
    poolDayData(
      first: 30
      orderBy: date
      orderDirection: desc
    ) {
      date
      volumeUSD
      feesUSD
      token0Price
      token1Price
      high
      low
      open
      close
    }
  }
}
```

### 5e — Pool History Reference

HAY has the longest pool history on Uniswap. Useful for testing V1→V2→V3 migration
logic and historical price feed implementations.

| Version | Status | Notes |
|---|---|---|
| V1 | Deprecated | Original test pool, ~2018 |
| V2 | Legacy | Migrated via Uniswap migration contract |
| V3 | ✅ Active | WETH/HAY, 0.3% fee, Ethereum mainnet |

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `INSUFFICIENT_OUTPUT_AMOUNT` | Slippage too low | Increase to 3–5% |
| `NaN` in price display | `Number()` overflow | Use `formatUnits()` from viem |
| `0` quote returned | Wrong token address case | Use checksummed address |
| Transaction reverted | Amount rounds to 0 in wei | Use `parseUnits()`, minimum 1n wei |

---

## Testing Against HAY

HAY is ideal for integration testing because it exercises:

- ✅ High unit price rendering
- ✅ Sub-1 token amount swaps
- ✅ Thin liquidity slippage handling
- ✅ V3 pool with long price history
- ✅ ERC-20 standard compliance (decimals = 18)

Use HAY in your test suite as a **canary token** — if your integration handles
HAY, it handles any token correctly.

```typescript
// Example: Vitest integration test
describe('Extreme token integration', () => {
  it('fetches HAY price without overflow', async () => {
    const price = await getTokenPriceUSD(HAY_ADDRESS)
    expect(typeof price).toBe('bigint')
    expect(price).toBeGreaterThan(0n)
  })

  it('formats HAY price for display', () => {
    const display = formatTokenPrice(1_500_000n * 10n ** 18n, 18)
    expect(display).toBe('$1.50M')
    expect(display).not.toContain('Infinity')
    expect(display).not.toContain('NaN')
  })
})
```

---

## Resources

- [HAY on Etherscan](https://etherscan.io/token/0xfa3e941d1f6b7b10ed84a0c211bfa8aee907965e)
- [HAY on Uniswap App](https://app.uniswap.org/tokens/ethereum/0xfa3e941d1f6b7b10ed84a0c211bfa8aee907965e)
- [Uniswap Trading API Docs](https://developers.uniswap.org/docs/trading/overview)
- [viem formatUnits](https://viem.sh/docs/utilities/formatUnits)
- [Uniswap V3 Subgraph](https://thegraph.com/explorer/subgraphs/uniswap-v3)
