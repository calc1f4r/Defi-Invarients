# ERC-721 Security Invariants

## 1. Supply & Existence Invariants

### 1.1 Existence ↔ Ownership
A token exists if and only if it has an owner. This prevents "ghost" tokens that exist without proper ownership.

$$
\forall\,t:\; \text{\_exists}(t)\;\iff\;\text{ownerOf}(t)\neq\mathbf{0}
$$

### 1.2 Total Supply Accounting
For enumerable implementations, the sum of all individual balances must equal the total supply. This invariant detects balance leaks or double-counting errors.

$$
\sum_{a}\text{balanceOf}(a) = \text{totalSupply}()
$$

### 1.3 Minting Precondition
Before minting token $t$, the token must not exist: $\lnot\text{\_exists}(t)$. After successful minting, the token must exist and both individual balance and total supply must increment by exactly one.

### 1.4 Burning Precondition  
Before burning token $t$, the token must exist: $\text{\_exists}(t)$. After successful burning, the token must not exist and both individual balance and total supply must decrement by exactly one.

---

## 2. Ownership & Balance Invariants

### 2.1 Balance Consistency
An address's balance must exactly equal the count of tokens it owns. This keeps internal balance mappings synchronized with ownership records.

$$
\forall\,a:\;\text{balanceOf}(a) = \bigl|\{\,t\mid\text{ownerOf}(t)=a\}\bigr|
$$

### 2.2 Zero-Address Balance Guard
Querying `balanceOf(address(0))` must either revert or consistently return zero, as the zero address cannot own tokens.

### 2.3 Uniqueness of Ownership
Each existing token has exactly one owner at any given time, preventing multi-ownership conflicts.

$$
\forall\,t:\;\exists!\,a\;:\;\text{ownerOf}(t)=a
$$

---

## 3. Transfer Invariants

### 3.1 Transfer State Transition
For any successful `transferFrom(from, to, tokenId)` operation, the post-state must satisfy all of the following conditions atomically:

$$
\begin{cases}
\text{balanceOf}(from)' = \text{balanceOf}(from) - 1\\
\text{balanceOf}(to)' = \text{balanceOf}(to) + 1\\
\text{ownerOf}(tokenId)' = to\\
\text{getApproved}(tokenId)' = \mathbf{0}
\end{cases}
$$

This ensures atomic state updates and automatic approval reset.

### 3.2 Zero-Address Transfer Guard
Any transfer call with `from == address(0)` or `to == address(0)` must revert, with the exception that minting operations use `from == address(0)`.

### 3.3 Safe-Transfer Receiver Check
When using `safeTransferFrom`, if the recipient `to` is a contract, it must implement `onERC721Received` and return the correct selector to prevent token lock-ups.

$$
\text{\_checkOnERC721Received} = \text{IERC721Receiver.onERC721Received.selector}
$$

### 3.4 Reentrancy Safety
All transfer operations must follow the checks-effects-interactions pattern, updating internal state before making external calls.

---

## 4. Approval Invariants

### 4.1 Approval Authorization
Only the token owner or an address approved via `setApprovalForAll` may call `approve(to, tokenId)`:

$$
\text{approve}(to,t)\;\implies\;\bigl(\text{msg.sender}=\text{ownerOf}(t)\;\vee\;\text{isApprovedForAll}(\text{ownerOf}(t),\text{msg.sender})\bigr)
$$

### 4.2 Stale Approval Reset
Any transfer or burn operation on token $t$ must reset its individual approval: `getApproved(t) = address(0)`.

### 4.3 Operator Enumeration Bounds
If `isApprovedForAll(owner, operator) = true`, then the operator must appear in the owner's internal operator list (implementation-dependent).

---

## 5. Enumeration Invariants (ERC-721Enumerable)

### 5.1 Owner-Index Consistency
If `tokenOfOwnerByIndex(owner, index) = tokenId`, then `ownerOf(tokenId) = owner` and $0 \leq index < \text{balanceOf}(owner)$.

### 5.2 Global-Index Consistency  
If `tokenByIndex(globalIndex) = tokenId`, then $0 \leq globalIndex < \text{totalSupply}()$.

### 5.3 Enumerable Removal Integrity
When burning or transferring tokens, enumeration arrays must be updated to maintain contiguous indexing without gaps.

---

## 6. Metadata & Interface Invariants

### 6.1 Interface Support Declaration
The contract must return true for all supported interfaces when queried via `supportsInterface`:

$$
\forall\,I\in\{\text{ERC165},\text{ERC721},\text{ERC721Metadata}\}:\;\text{supportsInterface}(I)=\text{true}
$$

### 6.2 Token URI Consistency
If token URIs follow a pattern like `tokenURI(tokenId) = baseURI + tokenId`, then changes to `baseURI` must be reflected consistently across all token URIs.

---

## 7. Event ↔ State Alignment

### 7.1 Transfer Event Alignment
A `Transfer(from, to, tokenId)` event must be emitted if and only if `ownerOf(tokenId)` changes from `from` to `to`.

### 7.2 Approval Event Alignment  
An `Approval(owner, approved, tokenId)` event must be emitted if and only if `getApproved(tokenId)` becomes `approved`.

### 7.3 ApprovalForAll Event Alignment
An `ApprovalForAll(owner, operator, approved)` event must be emitted if and only if `isApprovedForAll(owner, operator)` changes state.

---

## References

[1]: https://github.com/crytic/properties "GitHub - crytic/properties: Pre-built security properties for common Ethereum operations"
[2]: https://www.cyfrin.io/glossary/erc-721 "Cyfrin EIP and ERC Glossary: ERC-721"
[3]: https://blog.trailofbits.com/2025/02/12/the-call-for-invariant-driven-development/ "The call for invariant-driven development - The Trail of Bits Blog"
