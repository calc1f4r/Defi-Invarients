### All Erc20 Token Invariants

### General Invariants

- User balances should always be non-negative.
- Total supply of tokens should always be non-negative.
- The sum of all user balances should equal the total supply of tokens.
- TotalSupply should be always greater than to some of one user balance.

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
- **erc20-transfer-correct-amount-self**: Function `transfer` must handle self transfers correctly. Self-transfers must not change the balance of `msg.sender`.
- **erc20-transfer-change-state**: Function `transfer` must have no unexpected state changes. Must only modify balance entries of `msg.sender` and recipient addresses.

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
- **erc20-transferfrom-correct-amount-self**: Function `transferFrom` must perform self transfers correctly without changing the balance.
- **erc20-transferfrom-correct-allowance**: Function `transferFrom` must update the allowance correctly by decreasing it by the transferred amount.
- **erc20-transferfrom-change-state**: Function `transferFrom` must have no unexpected state changes. May only modify:
  - Balance entry for the destination address
  - Balance entry for the source address  
  - Allowance for `msg.sender` over the source address

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
- **erc20-approve-change-state**: Function `approve` must have no unexpected state changes. Must only update the allowance mapping and incur no other state changes.

#### Approve Error Handling
- **erc20-approve-false**: If function `approve` returns `false`, the contract's state must not be changed.
- **erc20-approve-never-return-false**: Function `approve` must never return `false`.

### View Function Invariants

#### TotalSupply Function Invariants
- **erc20-totalsupply-succeed-always**: Function `totalSupply` must always succeed (assuming sufficient gas).
- **erc20-totalsupply-correct-value**: Function `totalSupply` must return the value held in the corresponding state variable.
- **erc20-totalsupply-change-state**: Function `totalSupply` must not change any state variables.

#### BalanceOf Function Invariants
- **erc20-balanceof-succeed-always**: Function `balanceOf` must always succeed (assuming sufficient gas).
- **erc20-balanceof-correct-value**: Function `balanceOf(owner)` must return the value held in the contract's balance mapping for the owner address.
- **erc20-balanceof-change-state**: Function `balanceOf` must not change any state variables.

#### Allowance Function Invariants
- **erc20-allowance-succeed-always**: Function `allowance` must always succeed (assuming sufficient gas).
- **erc20-allowance-correct-value**: Function `allowance(owner, spender)` must return the allowance that spender has over tokens held by owner.
- **erc20-allowance-change-state**: Function `allowance` must not change any state variables.

### Mathematical Invariants

#### Balance Invariants
- All user balances must be non-negative at all times
- Individual balances must not exceed the maximum value for the underlying integer type
- Balance changes must be atomic and consistent

#### Supply Invariants  
- Total supply must be non-negative
- Total supply must equal the sum of all individual balances
- Total supply changes must only occur through minting/burning operations
- Total supply must not exceed the maximum value for the underlying integer type

#### Allowance Invariants
- All allowances must be non-negative
- Allowances must not exceed the maximum value for the underlying integer type
- Allowance changes must be properly authorized by the token owner

#### Overflow/Underflow Protection
- All arithmetic operations must be protected against integer overflow
- All arithmetic operations must be protected against integer underflow
- Balance transfers must not cause recipient balance overflow
- Allowance operations must not cause overflow

Resources 
https://www.certik.com/resources/blog/iLhagzcWkOVzOxoy0AWG3-erc-20-properties-for-formal-verification