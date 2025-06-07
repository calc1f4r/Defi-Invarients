# Major Invariants for Uniswap V2 AMM

## Variable Definitions

- `x`: Reserve of token0 in the pool
- `y`: Reserve of token1 in the pool  
- `L`: Total supply of Liquidity Pool (LP) tokens
- `k = x * y`: The constant product
- `f = 0.003`: The 0.3% trading fee

---

## 1. Core AMM Invariants

### 1. **Constant Product Formula**

```math
(x + \Delta x_{for\_swap}) \times (y - \Delta y_{out}) = x \times y
```

Where `Δx_for_swap = Δx_in × (1 - f)` after fees are deducted.

The core principle that determines swap output amounts. It applies the constant product principle to the reserves *before* the trade, using the fee-adjusted input amount.

### 2. **Swap Output Calculation**

```math
\Delta y_{out} = \frac{y \times \Delta x_{for\_swap}}{x + \Delta x_{for\_swap}}
```

Determines the exact amount of output tokens for a given input, derived from the constant product formula.

### 3. **Fee-Adjusted Product Growth**

```math
k_{after} = (x + \Delta x_{in}) \times (y - \Delta y_{out}) > k_{before}
```

When `f > 0` and `Δx_in > 0`, the product increases due to fees being retained in reserves.

### 4. **Reserve Updates**

```math
x' = x + \Delta x_{in} \quad\wedge\quad y' = y - \Delta y_{out}
```

Complete input amount (including fees) is added to reserves; calculated output is removed.

---

## 2. Liquidity Management

### 5. **Proportional Deposit Requirement**

```math
\frac{\Delta x}{\Delta y} = \frac{x}{y} \quad\text{or}\quad \frac{\Delta x}{x} = \frac{\Delta y}{y}
```

Liquidity providers must add tokens in proportion to current reserves to maintain price stability.

### 6. **LP Token Minting (Initial)**

```math
L_{minted} = \sqrt{\Delta x \times \Delta y}
```

For the first liquidity provider, `MINIMUM_LIQUIDITY = 1000` wei is permanently locked. The provider receives `L_{minted} - MINIMUM_LIQUIDITY` tokens, and the total supply becomes `L_{minted}`.

### 7. **LP Token Minting (Subsequent)**

```math
\Delta L = \min\left(\frac{\Delta x \times L_{old}}{x_{old}}, \frac{\Delta y \times L_{old}}{y_{old}}\right)
```

Due to proportionality requirement, both ratios should be equal in valid deposits.

### 8. **LP Token Redemption**

```math
\Delta x_{out} = \frac{\Delta L_{burn}}{L_{total}} \times x_{current} \quad\wedge\quad \Delta y_{out} = \frac{\Delta L_{burn}}{L_{total}} \times y_{current}
```

Burned LP tokens are redeemed proportionally to current reserves.

### 9. **Ownership Share Preservation**

```math
\text{Share} = \frac{l}{L} \quad\text{where}\quad \text{Claimable}_0 = \frac{l}{L} \times x \quad\wedge\quad \text{Claimable}_1 = \frac{l}{L} \times y
```

An LP's proportional claim on reserves remains constant as fees accumulate (L unchanged during swaps).

### 10. **LP-to-Product Relationship**

```math
L^2 \approx k
```

The relationship `L^2 = k` holds true when liquidity is added or removed at the current price ratio. During swaps, `k` increases due to fees while `L` remains constant, thus increasing the value backing each LP token.

---

## 3. Fee Mechanics

### 11. **Fee Deduction**

```math
\Delta x_{for\_swap} = \Delta x_{in} \times (1 - f)
```

Only the fee-adjusted input participates in the constant product calculation.

### 12. **Fee Accumulation**

```math
\text{Fee}_{token0} = \Delta x_{in} \times f
```

Fees remain in reserves, causing `k` to grow and benefiting LP token holders proportionally.

---

## 4. Price Oracle (TWAP)

### 13. **Price Accumulator Updates**

```math
\begin{align}
\text{price0Cumulative}_{new} &= \text{price0Cumulative}_{old} + \frac{y}{x} \times \text{timeElapsed} \\
\text{price1Cumulative}_{new} &= \text{price1Cumulative}_{old} + \frac{x}{y} \times \text{timeElapsed}
\end{align}
```

Where `timeElapsed = currentBlockTimestamp - blockTimestampLast > 0`.

### 14. **Timestamp Update**

```math
\text{blockTimestampLast}_{new} = \text{currentBlockTimestamp}
```

Updated whenever reserves are modified (swap, mint, burn operations).

### 15. **TWAP Calculation**

```math
\text{TWAP}(t_1, t_2) = \frac{\text{priceCumulative}(t_2) - \text{priceCumulative}(t_1)}{t_2 - t_1}
```

Time-weighted average price over any period using the cumulative accumulators.

---

## 5. State Consistency

### 16. **Reserve Non-Negativity**

```math
x \geq 0 \quad\wedge\quad y \geq 0 \quad\wedge\quad L \geq \text{MINIMUM\_LIQUIDITY}
```

Reserves and LP supply must remain non-negative, with minimum liquidity always locked.

### 17. **Atomic State Updates**

All state changes (reserves, LP supply, price accumulators) within a single transaction must be consistent and complete.

### 18. **No Zero-Amount Operations**

```math
\Delta x_{in} > 0 \quad\wedge\quad \Delta y_{out} > 0 \text{ for valid swaps}
```
