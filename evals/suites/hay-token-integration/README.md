# Eval Suite: hay-token-integration

## Purpose

Validates that the `hay-token-integration` skill produces correct, safe guidance
for working with extreme ERC-20 tokens like HAY.

## Test Cases

### 1. Price Fetch Guidance

**Prompt:**
> How do I fetch the current price of HAY token (0xfA3E941D1F6B7b10eD84A0C211bfA8aeE907965e)?

**Expected response includes:**
- Uniswap Trading API endpoint reference
- Use of `BigInt` or `formatUnits` — NOT `parseFloat` or `Number()`
- Correct contract address (checksummed)
- Note about lowercase address requirement for subgraph queries

**Must NOT include:**
- `Number(price)` conversion on raw wei values
- `toFixed()` without overflow guard
- Hardcoded price values

---

### 2. Swap Execution Safety

**Prompt:**
> I want to swap 0.001 HAY for WETH on Uniswap. How do I do this safely?

**Expected response includes:**
- `parseUnits('0.001', 18)` for amount encoding
- Slippage tolerance warning (minimum 3%)
- Universal Router or Trading API calldata approach
- Warning about thin liquidity

**Must NOT include:**
- `0.001 * 1e18` floating point math
- Slippage set to < 1%
- Direct V3 pool call without routing

---

### 3. UI Display Safety

**Prompt:**
> My price display shows `Infinity` for HAY. How do I fix it?

**Expected response includes:**
- Root cause: `Number()` overflow on large `BigInt` values
- Fix using `formatUnits()` from viem
- Safe formatter with million/thousand abbreviation
- Example code showing correct pattern

---

### 4. Token as Test Fixture

**Prompt:**
> How can I use HAY in my integration test suite?

**Expected response includes:**
- HAY as a "canary token" framing
- Test cases for price display, swap amount, slippage
- Vitest/Jest example using the contract address
- Note that passing HAY tests = passing all ERC-20 edge cases

---

## Scoring Rubric

| Criterion | Weight |
|---|---|
| Uses BigNumber-safe math | 30% |
| Correct slippage guidance | 20% |
| Accurate contract address | 20% |
| Code examples are runnable | 20% |
| No hallucinated API endpoints | 10% |
