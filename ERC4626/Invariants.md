# Major invariants for ERC-4626 vaults
### 1. Supply & Asset Accounting

1. **Total-Supply Consistency**

   $$
     \mathit{totalSupply}() \;=\; \sum_{a \in \mathit{accounts}} \! \mathit{balanceOf}(a)
   $$

   Ensures no phantom shares exist.

2. **Total-Assets Accuracy**

   $$
     \mathit{totalAssets}() \;=\; \text{on-chain balance of underlying tokens held by vault}
   $$

   After a successful deposit of `A` assets:

   $$
     \mathit{totalAssets}_{\text{new}} = \mathit{totalAssets}_{\text{old}} + A
   $$

   (minus any protocol fees, which must be inclusive in all asset accounting)

---

### 2. Conversion Round-Trips

3. **Asset→Share→Asset**

   $$
     \mathit{convertToAssets}\bigl(\mathit{convertToShares}(A)\bigr)\;\ge\;A
   $$

   Prevents “free” asset creation by round-tripping.

4. **Share→Asset→Share**

   $$
     \mathit{convertToShares}\bigl(\mathit{convertToAssets}(S)\bigr)\;\le\;S
   $$

   Prevents “free” share creation by round-tripping. 
---

### 3. Preview vs. Ideal Conversion

5. **PreviewDeposit ≤ Ideal**

   $$
     \mathit{previewDeposit}(A)\;\le\;\mathit{convertToShares}(A)
   $$

   Accounts for deposit fees and slippage bounds.

6. **PreviewMint ≥ Ideal**

   $$
     \mathit{previewMint}(S)\;\ge\;\mathit{convertToAssets}(S)
   $$

   Accounts for mint fees and slippage bounds.

7. **PreviewWithdraw ≥ Ideal**

   $$
     \mathit{previewWithdraw}(A)\;\ge\;\mathit{convertToShares}(A)
   $$

   Ensures withdrawals factor in fees/slippage conservatively.

8. **PreviewRedeem ≤ Ideal**

   $$
     \mathit{previewRedeem}(S)\;\le\;\mathit{convertToAssets}(S)
   $$

   Ensures redemptions factor in fees/slippage conservatively.

---

## 4. Preview vs. Actual


9. **Deposit vs. Preview**

   $$
     \mathit{deposit}(A) \;\ge\; \mathit{previewDeposit}(A)
     \quad\wedge\quad
     \mathit{deposit}(A) \;\le\; \mathit{convertToShares}(A)
   $$

10. **Mint vs. Preview**

    $$
      \mathit{mint}(S) \;\le\; \mathit{previewMint}(S)
      \quad\wedge\quad
      \mathit{mint}(S) \;\ge\; \mathit{convertToAssets}(S)
    $$

11. **Withdraw vs. Preview**

    $$
      \mathit{withdraw}(A) \;\le\; \mathit{previewWithdraw}(A)
      \quad\wedge\quad
      \mathit{withdraw}(A) \;\ge\; \mathit{convertToShares}(A)
    $$

12. **Redeem vs. Preview**

    $$
      \mathit{redeem}(S) \;\ge\; \mathit{previewRedeem}(S)
      \quad\wedge\quad
      \mathit{redeem}(S) \;\le\; \mathit{convertToAssets}(S)
    $$

---

## 5. Caller-Agnostic Behavior

13. **Uniform Convert**
    For any two callers $X, Y$ and amount $A$:

    $$
      \mathit{convertToShares}_X(A)\;=\;\mathit{convertToShares}_Y(A)
      \quad\wedge\quad
      \mathit{convertToAssets}_X(S)\;=\;\mathit{convertToAssets}_Y(S)
    $$

    (No caller-dependent pricing.)
    Ensures that the conversion rate between assets and shares is the same for all callers, preventing preferential treatment.

---

## 6. Limits & Non-Reverting

14. **maxDeposit / maxMint / maxWithdraw / maxRedeem**
    Each limit function:

    * Must return a value $\\in [0, 2^{256}-1]$.
    * Must never revert.
    * Must **underestimate** (never overestimate) the actual permissible action. 

    These functions provide safe upper bounds for operations, ensuring they don't fail due to exceeding vault or system limits.

15. **No Silent Reverts**
    All view functions (`asset()`, `totalAssets()`, all `max*` and `preview*`) must be non-reverting, per spec. 
    View functions should always return a value or revert with a clear error, not fail silently, to provide reliable information.

---

## 7. State-Change Consistency

16. **Balance Updates**

    * After `deposit(A)`:
      $\\mathit{balanceOf}(\\mathit{receiver})_{\\text{new}} = \\mathit{balanceOf}_{\\text{old}} + \\mathit{deposit}(A)$
    * After `withdraw(A)`:
      $\\mathit{balanceOf}(\\mathit{owner})_{\\text{new}} = \\mathit{balanceOf}_{\\text{old}} - \\mathit{withdraw}(A)$

    Guarantees that a user's share balance accurately reflects the shares received from a deposit or shares taken during a withdrawal.

17. **Supply Updates**

    * After `mint(S)`:
      $\\mathit{totalSupply}_{\\text{new}} = \\mathit{totalSupply}_{\\text{old}} + S$
    * After `redeem(S)`:
      $\\mathit{totalSupply}_{\\text{new}} = \\mathit{totalSupply}_{\\text{old}} - S$

    Ensures the vault's total supply of shares is correctly updated when shares are minted or redeemed.

---

## 8. Event Emission

18. **Required Events**

    * `Deposit(from, to, assets)` & ERC-20 `Transfer` (minted shares)
    * `Withdraw(from, to, assets)` & ERC-20 `Transfer` (burned shares)
    * Similarly for `Mint` and `Redeem` flows.

Vaults **must** emit these to enable reliable indexing and composability. 

Resources:
- https://zokyo.io/blog/formal-verification-with-certora-prover/
- https://medium.com/%40vikram.arun/multichain-erc-4626-conformance-f34b682b273b
- https://a16zcrypto.com/posts/article/generalized-property-tests-for-erc4626-vaults