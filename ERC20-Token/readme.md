### All Erc20 Token Invariants

### General Invariants

- User balances should always be non-negative.
  ```math
  ∀ user ∈ Users : balance[user] ≥ 0
  ```

- Total supply of tokens should always be non-negative.
  ```math
  totalSupply ≥ 0
  ```

- The sum of all user balances should equal the total supply of tokens.
  ```math
  ∑(user ∈ Users) balance[user] = totalSupply
  ```

- TotalSupply should be always greater than to some of one user balance.
  ```math
  ∀ user ∈ Users : totalSupply ≥ balance[user]
  ```

### Transfer Function Invariants

#### Transfer Security Invariants
- **erc20-transfer-revert-zero**: Function `transfer` must prevent transfers to the zero address. Any call `transfer(recipient, amount)` must fail if the recipient address is the zero address.
- **erc20-transfer-exceed-balance**: Function `transfer` must fail if requested amount exceeds available balance. Any transfer of an amount that exceeds the balance of `msg.sender` must fail.
- **erc20-transfer-recipient-overflow**: Function `transfer` must prevent overflows in the recipient's balance. Any invocation must fail if it causes the recipient's balance to overflow.

#### Transfer Success Conditions
- **erc20-transfer-succeed-normal**: Function `transfer` must succeed on admissible non-self transfers when:
  - The recipient address is not the zero address
  - Amount does not exceed the balance of `msg.sender`
  - Transferring amount to recipient does not cause overflow
  - Sufficient gas is provided
- **erc20-transfer-succeed-self**: Function `transfer` must succeed on admissible self transfers when:
  - The value in amount does not exceed the balance of `msg.sender`
  - Sufficient gas is provided

#### Transfer Correctness Invariants
- **erc20-transfer-correct-amount**: Function `transfer` must transfer the correct amount in non-self transfers. Must subtract the amount from sender's balance and add the same value to recipient's balance.
  ```math
  transfer(sender, recipient, amount) ⟹ 
  balance'[sender] = balance[sender] - amount ∧
  balance'[recipient] = balance[recipient] + amount
  (where sender ≠ recipient)
  ```

- **erc20-transfer-correct-amount-self**: Function `transfer` must handle self transfers correctly. Self-transfers must not change the balance of `msg.sender`.
  ```math
  transfer(sender, sender, amount) ⟹ 
  balance'[sender] = balance[sender]
  ```

- **erc20-transfer-change-state**: Function `transfer` must have no unexpected state changes. Must only modify balance entries of `msg.sender` and recipient addresses.
  ```math
  ∀ user ∉ {sender, recipient} : balance'[user] = balance[user] ∧
  totalSupply' = totalSupply ∧
  allowances' = allowances
  ```

#### Transfer Error Handling
- **erc20-transfer-false**: If function `transfer` returns `false`, the contract state must not be changed. Must undo all state changes before returning.
- **erc20-transfer-never-return-false**: Function `transfer` must never return `false` to signal failure.

### TransferFrom Function Invariants

#### TransferFrom Security Invariants
- **erc20-transferfrom-revert-from-zero**: Function `transferFrom` must fail for transfers from the zero address.
- **erc20-transferfrom-revert-to-zero**: Function `transferFrom` must fail for transfers to the zero address.
- **erc20-transferfrom-fail-exceed-balance**: Function `transferFrom` must fail if the requested amount exceeds the available balance of the `from` address.
- **erc20-transferfrom-fail-exceed-allowance**: Function `transferFrom` must fail if the requested amount exceeds the available allowance.
- **erc20-transferfrom-fail-recipient-overflow**: Function `transferFrom` must prevent overflows in the recipient's balance.

#### TransferFrom Success Conditions
- **erc20-transferfrom-succeed-normal**: Function `transferFrom` must succeed on admissible non-self transfers when:
  - Amount does not exceed the balance of `from` address
  - Amount does not exceed the allowance of `msg.sender` for `from` address
  - Transfer does not cause recipient balance overflow
  - Sufficient gas is provided
- **erc20-transferfrom-succeed-self**: Function `transferFrom` must succeed on admissible self transfers when conditions are met.

#### TransferFrom Correctness Invariants
- **erc20-transferfrom-correct-amount**: Function `transferFrom` must transfer the correct amount in non-self transfers.
  ```math
  transferFrom(from, to, amount) ⟹ 
  balance'[from] = balance[from] - amount ∧
  balance'[to] = balance[to] + amount
  (where from ≠ to)
  ```

- **erc20-transferfrom-correct-amount-self**: Function `transferFrom` must perform self transfers correctly without changing the balance.
  ```math
  transferFrom(from, from, amount) ⟹ 
  balance'[from] = balance[from]
  ```

- **erc20-transferfrom-correct-allowance**: Function `transferFrom` must update the allowance correctly by decreasing it by the transferred amount.
  ```math
  transferFrom(from, to, amount) ⟹ 
  allowance'[from][msg.sender] = allowance[from][msg.sender] - amount
  (unless allowance[from][msg.sender] = MAX_UINT256 or from = msg.sender)
  ```

- **erc20-transferfrom-change-state**: Function `transferFrom` must have no unexpected state changes. May only modify:
  - Balance entry for the destination address
  - Balance entry for the source address  
  - Allowance for `msg.sender` over the source address
  ```math
  ∀ user ∉ {from, to} : balance'[user] = balance[user] ∧
  ∀ (owner, spender) ≠ (from, msg.sender) : allowance'[owner][spender] = allowance[owner][spender] ∧
  totalSupply' = totalSupply
  ```

#### TransferFrom Error Handling
- **erc20-transferfrom-false**: If function `transferFrom` returns `false`, the contract's state must not be changed.
- **erc20-transferfrom-never-return-false**: Function `transferFrom` must never return `false`.

### Approve Function Invariants

#### Approve Security Invariants
- **erc20-approve-revert-zero**: Function `approve` must prevent giving approvals for the zero address. All calls `approve(spender, amount)` must fail if spender is the zero address.

#### Approve Success Conditions
- **erc20-approve-succeed-normal**: Function `approve` must succeed for admissible inputs when:
  - The spender address is not the zero address
  - Execution does not run out of gas

#### Approve Correctness Invariants
- **erc20-approve-correct-amount**: Function `approve` must update the approval mapping correctly according to `msg.sender`, `spender`, and `amount`.
  ```math
  approve(spender, amount) ⟹ 
  allowance'[msg.sender][spender] = amount
  ```

- **erc20-approve-change-state**: Function `approve` must have no unexpected state changes. Must only update the allowance mapping and incur no other state changes.
  ```math
  ∀ (owner, spender) ≠ (msg.sender, spender) : allowance'[owner][spender] = allowance[owner][spender] ∧
  balance' = balance ∧
  totalSupply' = totalSupply
  ```

#### Approve Error Handling
- **erc20-approve-false**: If function `approve` returns `false`, the contract's state must not be changed.
- **erc20-approve-never-return-false**: Function `approve` must never return `false`.

### View Function Invariants

#### TotalSupply Function Invariants
- **erc20-totalsupply-succeed-always**: Function `totalSupply` must always succeed (assuming sufficient gas).
  ```math
  ∀ state : totalSupply() → ℕ
  ```

- **erc20-totalsupply-correct-value**: Function `totalSupply` must return the value held in the corresponding state variable.
  ```math
  totalSupply() = totalSupply_state
  ```

- **erc20-totalsupply-change-state**: Function `totalSupply` must not change any state variables.
  ```math
  totalSupply() ⟹ state' = state
  ```

#### BalanceOf Function Invariants
- **erc20-balanceof-succeed-always**: Function `balanceOf` must always succeed (assuming sufficient gas).
  ```math
  ∀ owner ∈ Addresses : balanceOf(owner) → ℕ
  ```

- **erc20-balanceof-correct-value**: Function `balanceOf(owner)` must return the value held in the contract's balance mapping for the owner address.
  ```math
  balanceOf(owner) = balance[owner]
  ```

- **erc20-balanceof-change-state**: Function `balanceOf` must not change any state variables.
  ```math
  balanceOf(owner) ⟹ state' = state
  ```

#### Allowance Function Invariants
- **erc20-allowance-succeed-always**: Function `allowance` must always succeed (assuming sufficient gas).
  ```math
  ∀ (owner, spender) ∈ Addresses × Addresses : allowance(owner, spender) → ℕ
  ```

- **erc20-allowance-correct-value**: Function `allowance(owner, spender)` must return the allowance that spender has over tokens held by owner.
  ```math
  allowance(owner, spender) = allowance[owner][spender]
  ```

- **erc20-allowance-change-state**: Function `allowance` must not change any state variables.
  ```math
  allowance(owner, spender) ⟹ state' = state
  ```

### Mathematical Invariants

#### Balance Invariants
- All user balances must be non-negative at all times
  ```math
  ∀ user ∈ Users, ∀ time t : balance_t[user] ≥ 0
  ```

- Individual balances must not exceed the maximum value for the underlying integer type
  ```math
  ∀ user ∈ Users : balance[user] ≤ MAX_UINT256
  ```

- Balance changes must be atomic and consistent
  ```math
  ∀ transaction T : (∑ balance_before = ∑ balance_after) ∨ (T fails ∧ state unchanged)
  ```

#### Supply Invariants  
- Total supply must be non-negative
  ```math
  totalSupply ≥ 0
  ```

- Total supply must equal the sum of all individual balances
  ```math
  totalSupply = ∑_{user ∈ Users} balance[user]
  ```

- Total supply changes must only occur through minting/burning operations
  ```math
  Δ(totalSupply) ≠ 0 ⟹ (mint_operation ∨ burn_operation)
  ```

- Total supply must not exceed the maximum value for the underlying integer type
  ```math
  totalSupply ≤ MAX_UINT256
  ```

#### Allowance Invariants
- All allowances must be non-negative
  ```math
  ∀ (owner, spender) ∈ Users × Users : allowance[owner][spender] ≥ 0
  ```

- Allowances must not exceed the maximum value for the underlying integer type
  ```math
  ∀ (owner, spender) : allowance[owner][spender] ≤ MAX_UINT256
  ```

- Allowance changes must be properly authorized by the token owner
  ```math
  Δ(allowance[owner][spender]) ≠ 0 ⟹ msg.sender = owner
  ```

#### Overflow/Underflow Protection
- All arithmetic operations must be protected against integer overflow
  ```math
  ∀ (a, b) ∈ ℕ × ℕ : a + b < MAX_UINT256 ∨ operation_reverts
  ```

- All arithmetic operations must be protected against integer underflow
  ```math
  ∀ (a, b) ∈ ℕ × ℕ : a ≥ b ∨ (a - b operation_reverts)
  ```

- Balance transfers must not cause recipient balance overflow
  ```math
  transfer(from, to, amount) ⟹ 
  balance[to] + amount ≤ MAX_UINT256 ∨ transaction_reverts
  ```

- Allowance operations must not cause overflow
  ```math
  approve(spender, amount) ⟹ 
  amount ≤ MAX_UINT256 ∨ transaction_reverts
  ```

Resources 
https://www.certik.com/resources/blog/iLhagzcWkOVzOxoy0AWG3-erc-20-properties-for-formal-verification