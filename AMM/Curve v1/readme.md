# Major Invariants for Curve V1 AMM

This document outlines the critical invariants of the Curve v1 Stableswap AMM, with considerations for both fuzz testing and formal verification.

## Variable Definitions

- `n`: Number of coins in the pool.
- `x_i`: Balance of the i-th coin.
- `S`: Sum of all coin balances, `sum(x_i)`.
- `P`: Product of all coin balances, `product(x_i)`.
- `A`: Amplification coefficient, a parameter that controls the "flatness" of the curve.
- `D`: The invariant, a nominal measure of total liquidity. It is computed iteratively by `get_D(balances, A)`.
- `L`: Total supply of Liquidity Pool (LP) tokens.

---

## 1. Core Stableswap Invariants

### 1.1 The Stableswap Invariant Equation

The core of Curve finance is the Stableswap invariant, which is designed to provide low slippage for assets that are pegged to each other.

**Invariant Equation:**
```math
A \cdot n^n \cdot S + D = A \cdot D \cdot n^n + \frac{D^{n+1}}{n^n \cdot P}
```

**Description:** This equation must hold true for the given balances `x_i`, the amplification coefficient `A`, and the calculated invariant `D`. It defines the relationship between the sum and product of balances, creating a hybrid between a constant sum and constant product formula.

**Verification Approaches:**

-   **Fuzz Testing:** Store balances and `A`, call `get_D` to find `D_current`, and plug all values back into the equation. Assert that the left-hand side (LHS) and right-hand side (RHS) are approximately equal, within a defined epsilon.
    `abs(LHS - RHS) <= epsilon`

**Edge Cases:**
-   **Empty Pool:** If balances are zero, `P` is zero, leading to division by zero. `get_D` typically returns 0. Pools usually require initial liquidity.
-   **A = 0:** The formula simplifies, behaving more like a constant product AMM.
-   **Large A:** The curve flattens, behaving more like a constant sum AMM (low slippage).
-   **`get_D` Convergence:** Fuzzing can test for balance combinations where `get_D` fails to converge or consumes excessive gas.
-   **Remove Liquidity (Imbalanced):** A user withdraws a single coin, paying an implicit fee. `D` decreases, but the fee benefits remaining LPs.

**Verification:**
-   **Fuzzing:** Snapshot `D` before an operation, perform the operation, snapshot `D` after, and assert the expected change.

---

## 2. Liquidity and LP Token Invariants

### 2.1 LP Token Supply and Balances

The accounting for LP tokens must be precise to ensure fair representation of liquidity providers' shares.

-   **Total Supply Integrity:** `LP_token.totalSupply()` must equal the sum of all individual user balances.
-   **Add Liquidity:** `new_totalSupply == old_totalSupply + minted_lp_tokens`. The amount of minted tokens must be correctly calculated.
-   **Remove Liquidity:** `new_totalSupply == old_totalSupply - burned_lp_tokens`.
-   **Balance Non-Negativity:** `LP_token.balanceOf(user) >= 0` for all users.

**Verification:**
-   **Fuzzing:** Track `totalSupply` and user balances before and after `add_liquidity` or `remove_liquidity` calls and assert the changes match the returned minted/burned amounts.
-   **Formal Verification:** Prove that `totalSupply` is updated correctly based on the symbolic amount of minted/burned LP tokens.

### 2.2 Virtual Price Integrity and Monotonicity

The virtual price represents the value of one LP token and should be a reliable indicator of LP profitability.

**Definition:**
```
virtual_price = D / LP_token.totalSupply()
```

-   **General Monotonicity:** Because fees from swaps increase `D` while `totalSupply` remains constant, the virtual price should be non-decreasing between liquidity operations.
    `vp_after_swap >= vp_before_swap`
-   **Impact of `claim_admin_fees()`:** When admin fees are claimed, a portion of the balances that contribute to `D` are withdrawn. This causes both `D` and `virtual_price` to decrease. Thus, `virtual_price` is non-decreasing *unless admin fees are claimed*.
-   **Impact of `A` ramping:** Changes to `A` can cause `D` to change, affecting the virtual price. Ramping `A` slowly mitigates sharp price shocks.

**Verification:**
-   **Fuzzing:** Check that `get_virtual_price()` is non-decreasing after swaps. Note that this check must be skipped for transactions that claim admin fees.
-   **Formal Verification:** Prove that `virtual_price` is non-decreasing on swaps. A separate rule can specify the conditions under which it can decrease (i.e., admin fee claims).

**Edge Cases:**
-   If `totalSupply` is zero (new pool), `get_virtual_price` typically returns a default initial value (e.g., `10^18`) to avoid division by zero.

---

## 3. Fee Invariants

### 3.1 Fee Calculation and Accrual

Fees are the primary incentive for liquidity providers. Their calculation must be correct and transparent.

**Description:**
-   A `fee` is charged on every trade.
-   A portion of this trade fee, the `admin_fee`, is allocated to the protocol administrators.
-   The entire fee is initially added back to pool balances, benefiting all LPs. The admin's portion is tracked internally and only removed from balances when `claim_admin_fees()` is called.

**Invariants:**
-   `fee >= 0` and `admin_fee >= 0`.
-   The fee charged on a swap `(y_ideal - y_received)` must be properly reflected in the corresponding increase in `D`.

**Verification:**
-   **Fuzzing:** Verifying fee impact is complex. It involves comparing the actual swap outcome to a hypothetical zero-fee swap. A simpler approach is to focus on the resulting increase in `D` and `virtual_price`.
-   **Formal Verification:** Model the fee calculation logic precisely and prove that the calculated fee amount directly contributes to the state changes in balances and `D`.

---

## 4. State Consistency and Safety Invariants

### 4.1 Conservation of Contract Token Balances

The contract's internal accounting of token balances must match the actual token balances it holds.

**Invariant:**
For Curve pools, admin fees are typically added directly to the main balances.
`token_i.balanceOf(address(curve_pool)) == curve_pool.balances(i)`

This must hold true at all times, except transiently during a token transfer. When admin fees are claimed, `balances[i]` decreases as tokens are transferred out.

**Verification:**
-   **Fuzzing:** After every state-changing operation, iterate through all tokens and assert that the ERC20 balance held by the contract equals its internally stored `balances(i)`.
-   **Formal Verification:** `assert token[i].balanceOf(currentContract) == balances[i];`

### 4.2 No Reverts on Valid State Queries

View functions should not revert when called with valid inputs, ensuring the contract remains queryable and integrations do not break.

**Functions to check:** `get_dy`, `get_D`, `calc_token_amount`, `calc_withdraw_one_coin`, etc.

**Verification:**
-   **Fuzzing:** After any sequence of state changes, call all view functions with a variety of valid inputs derived from the current state and assert they do not revert.
-   **Formal Verification:** Prove that for any reachable state, these functions do not revert. This often involves assuming bounds on loops (e.g., in `get_D`).

### 4.3 Output Amount Sanity for Swaps

Swap outputs must be logical and bounded.

**Invariants:**
-   If the input `dx > 0`, the output `dy` from `get_dy(i, j, dx)` should be `>= 0`.
-   The output `dy` should never be greater than the balance of the output token: `dy < balances[j]`.

**Verification:**
-   **Fuzzing:** Call `get_dy` with a range of inputs (small, large, up to the entire balance) and check that outputs are non-negative and less than the available balance.
-   **Formal Verification:** `dx > 0 && balances[i] > dx ==> get_dy(i,j,dx) >= 0`.

### 4.4 No Over/Underflows in Critical Arithmetic

Arithmetic operations on large numbers must not overflow or underflow, as this can lead to catastrophic errors.

**Verification:**
-   **Fuzzing:** Use a fuzzer that automatically detects overflows (e.g., Foundry). Test with extreme values for balances (near 0 and `type(uint256).max`) and for the `A` parameter.
-   **Formal Verification:** The prover inherently checks for overflows/underflows. Assertions on intermediate values can provide additional guarantees.

### 4.5 Admin Parameter Constraints

Admin-settable parameters must be constrained to prevent abuse or misconfiguration.

**Invariants:**
-   `fee` must be below a maximum percentage.
-   `admin_fee` must be less than or equal to 100% of the `fee`.
-   `A` parameter changes should be ramped over time, with limits on the rate of change.

**Verification:**
-   **Fuzzing:** Call admin functions with boundary values and out-of-bounds values. Expect reverts for invalid inputs and check that valid changes are applied correctly.
-   **Formal Verification:** Specify constraints as invariants on the stored parameters (e.g., `fee() < MAX_FEE`).

---

## 5. Practical Considerations for Verification

-   **State Machine Fuzzing:** Structure fuzz tests as a state machine where each state transition (swap, add liquidity, etc.) is followed by a check of all relevant invariants.
-   **Precision and Epsilon:** When checking invariants involving fixed-point arithmetic, use an acceptable `epsilon` for comparisons (`abs(expected - actual) <= epsilon`) rather than strict equality.
-   **Initialization:** Some invariants only hold after a pool has been initialized with liquidity. Tests and proofs may need to assume an initialized state.
-   **Gas and Loops:** Be mindful of the iterative nature of `get_D`. Fuzzing can reveal gas exhaustion scenarios. Formal verification proofs often require an assumption on the maximum number of loop iterations.

---

## 6. Core Mathematical Functions from Implementation

The core logic of the Curve AMM is implemented in a few key functions that perform the necessary calculations. Below are the mathematical details of the most critical functions, as found in the Vyper implementation.

### 6.1 `get_D`: Calculating the Invariant

The `get_D` function iteratively calculates the value of the invariant `D` for a given set of balances `x_i` and amplification coefficient `A`. It uses Newton's method to find the root of the invariant equation:

```math
f(D) = A n^n S - (A n^n - 1) D - \frac{D^{n+1}}{n^n P} = 0
```
Where:
- `S` is the sum of balances.
- `P` is the product of balances.

The iterative update formula used in the contract to find `D` is:

```math
D_{new} = \frac{(A n^n S + n D_P) D_{old}}{ (A n^n - 1) D_{old} + (n+1) D_P }
```
With `D_P` being a helper term calculated as:
```math
D_P = \frac{D_{old}^{n+1}}{n^n P}
```
The iteration starts with `D_0 = S` and continues until `D` converges.

### 6.2 `get_y`: Calculating Swap Output

The `get_y(i, j, dx)` function calculates the amount of coin `j` a user will receive when swapping an amount `dx` of coin `i`. It does so by finding the new balance of coin `j`, let's call it `y`, that satisfies the invariant equation while keeping `D` constant.

Given new balances for all coins except `j` (i.e., `x'_k` for `k \ne j`), we must solve the invariant equation for `y = x'_j`. This reduces to solving a quadratic equation for `y`:

```math
a y^2 + b y - c = 0
```

The coefficients are derived from the main invariant equation:

-   `a = A n^n`
-   `b = D - A n^n(D - S') = D + A n^n S' - A n^n D`
-   `c = \frac{D^{n+1}}{n^n P'}`

Where:
-   `S' = \sum_{k \neq j} x'_k` (sum of all new balances except for coin j)
-   `P' = \prod_{k \neq j} x'_k` (product of all new balances except for coin j)

The contract solves this quadratic equation for `y`, which represents the new balance of coin `j`. The amount of coin `j` returned to the user is then `dy = x_j - y`. A fee is also subtracted from `dy` before it's returned.
