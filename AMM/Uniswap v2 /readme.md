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

#### Verification via Pseudocode
```
// This invariant ensures that for any swap, the product of reserves,
// adjusted for fees, does not decrease. This check is the heart of the AMM.
// It is checked at the end of every swap in the Pair contract.

function verify_constant_product_after_swap(
    reserve_x_before,
    reserve_y_before,
    amount_x_in,
    amount_y_out // amount_y received by swapper
) {
    FEE_NUMERATOR = 997;
    FEE_DENOMINATOR = 1000;

    // The core check from UniswapV2Pair.sol, adapted for clarity.
    // It verifies that k_new >= k_old, after accounting for fees.
    // Equivalent of: (x * 1000 + dx_in * 997) * (y - dy_out) >= x * y * 1000
    left_side = (reserve_x_before * FEE_DENOMINATOR + amount_x_in * FEE_NUMERATOR) * (reserve_y_before - amount_y_out);
    right_side = reserve_x_before * reserve_y_before * FEE_DENOMINATOR;

    // In a real swap, this must hold true.
    assert(left_side >= right_side);
}

### 2. **Swap Output Calculation**

```math
\Delta y_{out} = \frac{y \times \Delta x_{for\_swap}}{x + \Delta x_{for\_swap}}
```

Determines the exact amount of output tokens for a given input, derived from the constant product formula.

#### Verification via Pseudocode
```
// This formula calculates the output of a swap. A fuzzer can use it
// to predict the output amount and verify that the contract returns the
// expected value.

function calculate_expected_swap_output(
    reserve_x,
    reserve_y,
    input_amount_x
) {
    FEE_NUMERATOR = 997;
    FEE_DENOMINATOR = 1000;

    input_amount_x_with_fee = input_amount_x * FEE_NUMERATOR;
    
    numerator = input_amount_x_with_fee * reserve_y;
    denominator = (reserve_x * FEE_DENOMINATOR) + input_amount_x_with_fee;
    expected_output_y = numerator / denominator;

    return expected_output_y;
}

// In a test environment:
// actual_output_y = swap(input_amount_x);
// expected_output_y = calculate_expected_swap_output(reserve_x, reserve_y, input_amount_x);
// assert(actual_output_y == expected_output_y);
```

### 3. **Fee-Adjusted Product Growth**

```math
k_{after} = (x + \Delta x_{in}) \times (y - \Delta y_{out}) > k_{before}
```

When `f > 0` and `Δx_in > 0`, the product increases due to fees being retained in reserves.

#### Verification via Pseudocode
```
// This invariant states that the product of reserves (k) increases after
// every swap due to fees. This can be checked by comparing k before and after a trade.

function verify_product_growth(
    reserve_x_before,
    reserve_y_before,
    amount_x_in,
    amount_y_out
) {
    k_before = reserve_x_before * reserve_y_before;

    reserve_x_after = reserve_x_before + amount_x_in;
    reserve_y_after = reserve_y_before - amount_y_out;
    k_after = reserve_x_after * reserve_y_after;

    // The product of reserves must increase if there was a trade.
    if (amount_x_in > 0) {
        assert(k_after > k_before);
    }
}
```

### 4. **Reserve Updates**

```math
x' = x + \Delta x_{in} \quad\wedge\quad y' = y - \Delta y_{out}
```

Complete input amount (including fees) is added to reserves; calculated output is removed.

#### Verification via Pseudocode
```
// This describes how reserves are updated. A verification tool
// should track the state of reserves and assert they are updated correctly
// after each operation (swap, mint, burn).

function verify_reserve_updates_after_swap(
    reserve_x_before,
    reserve_y_before,
    reserve_x_after,
    reserve_y_after,
    amount_x_in,
    amount_y_out
) {
    expected_reserve_x_after = reserve_x_before + amount_x_in;
    expected_reserve_y_after = reserve_y_before - amount_y_out;

    assert(reserve_x_after == expected_reserve_x_after);
    assert(reserve_y_after == expected_reserve_y_after);
}
```

---

## 2. Liquidity Management

### 5. **Proportional Deposit Requirement**

```math
\frac{\Delta x}{\Delta y} = \frac{x}{y} \quad\text{or}\quad \frac{\Delta x}{x} = \frac{\Delta y}{y}
```

Liquidity providers must add tokens in proportion to current reserves to maintain price stability.

#### Verification via Pseudocode
```
// When adding liquidity, tokens must be supplied in proportion to the
// current reserves to avoid changing the price.

function verify_proportional_deposit(
    reserve_x,
    reserve_y,
    amount_x_deposited,
    amount_y_deposited
) {
    // To avoid division, we use cross-multiplication.
    // amount_x_deposited / reserve_x == amount_y_deposited / reserve_y
    // This must hold true for a valid deposit.
    // The router calculates the optimal deposit amounts to satisfy this.
    assert(amount_x_deposited * reserve_y == amount_y_deposited * reserve_x);
}
```

### 6. **LP Token Minting (Initial)**

```math
L_{minted} = \sqrt{\Delta x \times \Delta y}
```

For the first liquidity provider, `MINIMUM_LIQUIDITY = 1000` wei is permanently locked. The provider receives `L_{minted} - MINIMUM_LIQUIDITY` tokens, and the total supply becomes `L_{minted}`.

#### Verification via Pseudocode
```
// For the first liquidity provider, LP tokens are minted based on the
// geometric mean of the initial liquidity. A small amount is locked.

function verify_initial_mint(
    amount_x_deposited,
    amount_y_deposited,
    lp_tokens_minted,
    total_lp_supply
) {
    MINIMUM_LIQUIDITY = 1000;

    // Note: integer square root is required.
    expected_lp_minted = integer_sqrt(amount_x_deposited * amount_y_deposited);

    assert(lp_tokens_minted == expected_lp_minted - MINIMUM_LIQUIDITY);
    assert(total_lp_supply == expected_lp_minted);
}
```

### 7. **LP Token Minting (Subsequent)**

```math
\Delta L = \min\left(\frac{\Delta x \times L_{old}}{x_{old}}, \frac{\Delta y \times L_{old}}{y_{old}}\right)
```

Due to proportionality requirement, both ratios should be equal in valid deposits.

#### Verification via Pseudocode
```
// For subsequent liquidity providers, LP tokens are minted proportional to their
// share of the total reserves.

function verify_subsequent_mint(
    amount_x_deposited,
    amount_y_deposited,
    reserve_x_before,
    reserve_y_before,
    total_lp_supply_before,
    lp_tokens_minted
) {
    // Due to the proportional deposit requirement, both calculations should yield the same result.
    // We can use either one.
    expected_lp_from_x = (amount_x_deposited * total_lp_supply_before) / reserve_x_before;
    expected_lp_from_y = (amount_y_deposited * total_lp_supply_before) / reserve_y_before;
    
    assert(lp_tokens_minted == expected_lp_from_x);
    assert(lp_tokens_minted == expected_lp_from_y);
}
```

### 8. **LP Token Redemption**

```math
\Delta x_{out} = \frac{\Delta L_{burn}}{L_{total}} \times x_{current} \quad\wedge\quad \Delta y_{out} = \frac{\Delta L_{burn}}{L_{total}} \times y_{current}
```

Burned LP tokens are redeemed proportionally to current reserves.

#### Verification via Pseudocode
```
// When burning LP tokens, a provider receives a proportional share of the
// current reserves.

function verify_lp_redemption(
    lp_tokens_burned,
    total_lp_supply_before,
    reserve_x_before,
    reserve_y_before,
    amount_x_out,
    amount_y_out
) {
    // Calculate the expected amount of tokens to be returned.
    expected_x_out = (lp_tokens_burned * reserve_x_before) / total_lp_supply_before;
    expected_y_out = (lp_tokens_burned * reserve_y_before) / total_lp_supply_before;

    assert(amount_x_out == expected_x_out);
    assert(amount_y_out == expected_y_out);
}
```

### 9. **Ownership Share Preservation**

```math
\text{Share} = \frac{l}{L} \quad\text{where}\quad \text{Claimable}_0 = \frac{l}{L} \times x \quad\wedge\quad \text{Claimable}_1 = \frac{l}{L} \times y
```

An LP's proportional claim on reserves remains constant as fees accumulate (L unchanged during swaps).

#### Verification via Pseudocode
```
// An LP holder's share of the pool remains constant during swaps, but the
// underlying value of that share grows as fees accumulate.

function calculate_claimable_assets(
    lp_tokens_held,
    total_lp_supply,
    reserve_x,
    reserve_y
) {
    // Calculate the claimable amount of each token for an LP.
    claimable_x = (lp_tokens_held * reserve_x) / total_lp_supply;
    claimable_y = (lp_tokens_held * reserve_y) / total_lp_supply;

    return (claimable_x, claimable_y);
}

// Verification could involve:
// 1. Store claimable assets before a swap.
// 2. After swap, recalculate claimable assets.
// 3. The new value should be higher due to fee accumulation.
```

### 10. **LP-to-Product Relationship**

```math
L^2 \approx k
```

The relationship `L^2 = k` holds true when liquidity is added or removed at the current price ratio. During swaps, `k` increases due to fees while `L` remains constant, thus increasing the value backing each LP token.

#### Verification via Pseudocode
```
// The square of the total LP supply is roughly proportional to the
// constant product `k`. It holds when liquidity is added/removed, but
// diverges during swaps as `k` grows while `L` is constant.

function verify_lp_to_product_relation_on_mint(
    reserve_x_before, reserve_y_before, total_lp_supply_before,
    reserve_x_after, reserve_y_after, total_lp_supply_after
) {
    k_before = reserve_x_before * reserve_y_before;
    k_after = reserve_x_after * reserve_y_after;

    // After a mint, the ratio L^2/k should be preserved.
    // Using cross-multiplication to avoid division and precision loss.
    // (L_before^2) / k_before == (L_after^2) / k_after
    assert(
        (total_lp_supply_before**2) * k_after == (total_lp_supply_after**2) * k_before
    );
}

function verify_lp_to_product_relation_on_swap(
    k_before, k_after, total_lp_supply
) {
    // During a swap, L is constant, but k grows.
    assert(k_after > k_before);
    assert( (total_lp_supply**2) / k_after < (total_lp_supply**2) / k_before );
}
```

---

## 3. Fee Mechanics

### 11. **Fee Deduction**

```math
\Delta x_{for\_swap} = \Delta x_{in} \times (1 - f)
```

Only the fee-adjusted input participates in the constant product calculation.

#### Verification via Pseudocode
```
// This describes how the input amount is adjusted for fees. The pseudo-code
// in invariant #2 (Swap Output Calculation) demonstrates this principle in action.
// The core logic is that 0.3% of the input amount is taken as a fee, and
// the remaining 99.7% is used for the swap calculation.
```

### 12. **Fee Accumulation**

```math
\text{Fee}_{token0} = \Delta x_{in} \times f
```

Fees remain in reserves, causing `k` to grow and benefiting LP token holders proportionally.

#### Verification via Pseudocode
```
// Fees are not sent anywhere; they are simply left in the pool. This
// increases the reserves and thus the value backing LP tokens. This invariant
// is implicitly verified by #3 (Fee-Adjusted Product Growth) and
// #4 (Reserve Updates), which show that the full input amount is added to
// reserves while the output is calculated based on a smaller, fee-adjusted input.
```

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

#### Verification via Pseudocode
```
// The TWAP oracle relies on price accumulators that are updated on every
// state-changing interaction with the pool.

function verify_price_accumulator_update(
    price0_cumulative_last,
    price1_cumulative_last,
    reserve0,
    reserve1,
    block_timestamp_last,
    current_block_timestamp
) {
    time_elapsed = current_block_timestamp - block_timestamp_last;
    
    if (time_elapsed > 0 && reserve0 > 0 && reserve1 > 0) {
        // Note: The actual implementation uses fixed-point math (UQ112x112).
        // This pseudocode uses fractional representation for clarity.
        price0 = reserve1 / reserve0;
        price1 = reserve0 / reserve1;

        expected_price0_cumulative = price0_cumulative_last + price0 * time_elapsed;
        expected_price1_cumulative = price1_cumulative_last + price1 * time_elapsed;

        // In a test, one would trigger an action on the pool and check if the
        // new cumulative price matches this calculation.
        // new_price0_cumulative = get_new_cumulative_price0();
        // assert(new_price0_cumulative == expected_price0_cumulative);
    }
}
```

### 14. **Timestamp Update**

```math
\text{blockTimestampLast}_{new} = \text{currentBlockTimestamp}
```

Updated whenever reserves are modified (swap, mint, burn operations).

#### Verification via Pseudocode
```
// The timestamp must be updated whenever reserves are modified to ensure
// the TWAP is accurate.

function verify_timestamp_update(
    block_timestamp_last_before,
    current_block_timestamp
) {
    // After any state-changing operation (swap, mint, burn):
    block_timestamp_last_after = get_block_timestamp_last();

    assert(block_timestamp_last_after == current_block_timestamp);
    assert(block_timestamp_last_after > block_timestamp_last_before);
}
```

### 15. **TWAP Calculation**

```math
\text{TWAP}(t_1, t_2) = \frac{\text{priceCumulative}(t_2) - \text{priceCumulative}(t_1)}{t_2 - t_1}
```

Time-weighted average price over any period using the cumulative accumulators.

#### Verification via Pseudocode
```
// A TWAP over a period can be calculated from two snapshots of the
// cumulative price accumulator.

function calculate_twap(
    price_cumulative_t1,
    price_cumulative_t2,
    t1,
    t2
) {
    // Ensure t2 > t1
    if (t2 <= t1) return 0;

    twap = (price_cumulative_t2 - price_cumulative_t1) / (t2 - t1);
    return twap;
}

// Verification would involve taking two snapshots (at t1 and t2) from the contract
// and ensuring an off-chain calculation matches a potential on-chain one,
// or that it behaves as expected.
```

---

## 5. State Consistency

### 16. **Reserve Non-Negativity**

```math
x \geq 0 \quad\wedge\quad y \geq 0 \quad\wedge\quad L \geq \text{MINIMUM\_LIQUIDITY}
```

Reserves and LP supply must remain non-negative, with minimum liquidity always locked.

#### Verification via Pseudocode
```
// Pool reserves and LP supply must never become negative. The minimum liquidity
// is a special constant that is locked in the contract forever.

function verify_non_negativity(
    reserve_x,
    reserve_y,
    total_lp_supply,
    initial_liquidity_has_been_added // Test-harness state
) {
    MINIMUM_LIQUIDITY = 1000;

    assert(reserve_x >= 0);
    assert(reserve_y >= 0);

    // Total supply can be 0 only before the first mint.
    // After the first mint, it must be at least MINIMUM_LIQUIDITY.
    if (initial_liquidity_has_been_added) {
       assert(total_lp_supply >= MINIMUM_LIQUIDITY);
    }
}
```

### 17. **Atomic State Updates**

All state changes (reserves, LP supply, price accumulators) within a single transaction must be consistent and complete.

#### Verification via Pseudocode
```
// All state changes within a single transaction must be consistent. For example,
// if reserves are updated, the price accumulators must also be updated in the
// same transaction.

// A fuzzer can verify this by invoking a function (e.g., swap) and then
// immediately checking all related state variables for consistency.

function verify_atomic_updates_after_swap(
    state_before_swap,
    state_after_swap
) {
    // Check reserve updates (Invariant 4)
    verify_reserve_updates_after_swap(...);
    
    // Check timestamp update (Invariant 14)
    verify_timestamp_update(...);

    // Check price accumulator updates (Invariant 13)
    verify_price_accumulator_update(...);

    // All these checks should pass together after a single swap transaction.
}
```

### 18. **No Zero-Amount Operations**

```math
\Delta x_{in} > 0 \quad\wedge\quad \Delta y_{out} > 0 \text{ for valid swaps}
```

#### Verification via Pseudocode
```
// Swaps must involve a non-zero amount of tokens to be valid.
// The contract enforces this with `require` statements.

function verify_no_zero_amount_swaps(
    input_amount_x,
    output_amount_y
) {
    // This check is typically at the entry point of a swap function.
    // The transaction should revert if these conditions are not met.
    assert(input_amount_x > 0);
    assert(output_amount_y > 0);
}
```
