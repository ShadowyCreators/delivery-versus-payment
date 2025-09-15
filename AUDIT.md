## Stats
- nSLOC: 473
- Complexity Score:
- Security Review Rimeline: Date --> Date

### Settlements Functional Immutability

settlments are not marked as immutable but the value of key settlment parameters (settlmentReference, cutoffDate, isAutoSettled, flows) are assigned only when calling `createSettlement` to create a settlement. 
There is no other function that modifies them.
There is admin or private modification function:
- No owner/admin functions
- No upgrade mechanisms
- No functions to modify existing settlements
- No functions to add/remove flows

### Settlment Fields that can be modified after creation

- approvals[address]: Modified in `approveSettlements()` and `revokeApprovals()`
- ethDeposits[address]: Modified in `approveSettlements()`, `revokeApprovals()`, and `withdrawETH()`
- isSettled: Modified in `executeSettlementInner()`

As for documentation, it is expected that these fields can be modified.

### All Settlement-Related Functions

External/Public functions:
- `createSettlement()` - Creates new settlements
- `approveSettlements()` - Modifies approvals and deposits
- `executeSettlement()` - Executes settlements
- `revokeApprovals()` - Modifies approvals and deposits
- `withdrawETH()` - Modifies deposits
- `getSettlement()` - Read-only access
- `isSettlementApproved()` - Read-only access
- `getSettlementPartyStatus()` - Read-only access

Internal/Helper Functions:
- `executeSettlementInner()` - Internal execution logic (external but only callable by the contract itself)

Helper functions: 
- `getSettlementsByToken()` - Read-only helper
- `getSettlementsByInvolvedParty()` - Read-only helper
- `getSettlementsByTokenType()` - Read-only helper
- `_getPagedSettlementIds()` - Internal helper
- `_getPagedSettlementIdsByType()` - Internal helper

Note that `executeSettlementInner()` is marked as `external` because  try/catch blocks can only catch exceptions from external or public function calls (as specified in the comment by dev).

Even though it is external it was implemented a secure access to the function: 
```
// this function can only be called by the DVP contract itself
if (msg.sender != address(this)) {
  revert CallerMustBeDvpContract();
}
```

### Token Type Validation

The contract validates token types during settlement creation using the ERC165 interface detection standard:

**ERC721 Validation:**
- Uses `token.supportsInterface(type(IERC721).interfaceId)` to verify NFT tokens
- If a flow is marked as `isNFT = true` but the token doesn't support ERC721 interface, creation reverts with `InvalidERC721Token()`
- This prevents accidental misclassification of ERC20 tokens as NFTs

**ERC20 Validation:**
- Uses `token.staticcall(abi.encodeWithSelector(SELECTOR_ERC20_DECIMALS))` to verify ERC20 tokens
- Checks if the token implements the `decimals()` function and returns a valid 32-byte result
- If validation fails, creation reverts with `InvalidERC20Token()`

This validation ensures that settlements can only be created with properly typed tokens, preventing execution failures and maintaining system integrity.

### Limitations of ERC20 Token Validation

The current ERC20 validation approach has several limitations:

**False Positives (Non-ERC20 contracts that pass validation):**
- Contracts that implement `decimals()` function but are not ERC20 tokens (e.g., custom contracts with decimal precision)
- ERC20-like contracts that aren't true ERC20 implementations
- Contracts that return `uint8` from `decimals()` but lack other ERC20 functions

**False Negatives (Legitimate ERC20 tokens that fail validation):**
- Very old ERC20 tokens that don't implement the `decimals()` function (extremely rare)
- ERC20 tokens with non-standard `decimals()` implementations (e.g., returning different data types)
- Malicious tokens that intentionally break the `decimals()` function

**Heuristic Nature:**
- The validation is based on a single function check rather than comprehensive ERC20 compliance
- It doesn't verify other essential ERC20 functions like `balanceOf()`, `transfer()`, or `approve()`
- A contract could pass validation but still fail during actual token transfers

**Security Implications:**
- Malicious actors could deploy contracts that pass ERC20 validation but behave unexpectedly during execution
- The validation provides reasonable protection against common mistakes but is not foolproof
- Users should still verify token contracts independently before creating settlements

**⚠️ Potential Issues:**
Execution-time failures: If a token was valid during creation but becomes invalid later, execution could fail
Status query failures: getSettlementPartyStatus() could revert if tokens don't implement allowance() or balanceOf()
No re-validation: Once a settlement is created, there's no ongoing validation of token contracts

### Token Upgradeability Risk

**Critical Risk**: Tokens can become malicious after settlement creation if they are upgradeable:

**Upgradeable Token Scenarios:**
- Proxy-based tokens (TransparentProxy, UUPS, BeaconProxy) can be upgraded to malicious implementations
- Admin-controlled tokens can change behavior through governance or admin functions
- Even tokens that pass initial validation can later become malicious through upgrades

**Attack Vectors:**
- Token admin upgrades implementation to steal funds during settlement execution
- Governance-controlled tokens change behavior through malicious proposals
- Compromised admin keys allow attackers to upgrade tokens to malicious versions

**No Protection Mechanisms:**
- DVP contract performs no re-validation of tokens after settlement creation
- No detection of token upgrades or implementation changes
- No whitelist of approved tokens
- No monitoring of token behavior changes

**Mitigation Recommendations:**
- Users should avoid settlements with upgradeable tokens
- Prefer well-known, non-upgradeable tokens (though even these may have admin functions)
- Consider implementing token whitelist or re-validation mechanisms
- Monitor token contracts for upgrade events before settlement execution

## Test Scenarios

### Basic Vulnerability Test
- Creates settlement with legitimate token
- Token passes ERC20 validation
- Token admin upgrades to malicious behavior
- Settlement execution steals all funds instead of normal transfer

### Multi-Flow Vulnerability Test
- Creates settlement with multiple flows using the same token
- Shows how malicious token can selectively target specific parties
- Demonstrates that not all flows are affected (only targeted ones)

### Proxy-Based Vulnerability Test
- Simulates realistic proxy token (like USDC/USDT)
- Shows legitimate upgrade followed by malicious upgrade
- Demonstrates admin compromise scenario

### Auto-Settlement Vulnerability Test
- Shows vulnerability with auto-settlements
- Malicious token can steal funds during auto-execution
- No protection even with automatic settlement execution

## Key Findings

1. **No Re-validation**: DVP contract never re-validates tokens after settlement creation
2. **No Upgrade Detection**: Contract has no mechanism to detect token upgrades
3. **No Whitelist**: No approved token list or blacklist mechanism
4. **Complete Fund Theft**: Malicious tokens can steal ALL tokens, not just the intended transfer amount
5. **Multi-Party Impact**: One malicious token can affect multiple parties in the same settlement


### Reentrancy considerations (current vs future)

As implemented today, there is no practical reentrancy vector: all external state-changing entrypoints (`approveSettlements`, `executeSettlement`, `revokeApprovals`, `withdrawETH`) are `nonReentrant`, and the inner execution function `executeSettlementInner` is only callable by the contract itself via a strict `msg.sender == address(this)` check. External recipients/tokens invoked during execution cannot call `executeSettlementInner`, and any attempt to re-enter other entrypoints is blocked by `nonReentrant`.

However, this posture could degrade if future versions introduce new external/public state-changing functions that are not `nonReentrant` and can be invoked during token callbacks. For example, items in the roadmap such as adding PERMIT2 support, netting logic, or ERC‑1155 support could add new entrypoints (e.g., `permitAndApprove`, `processNettedSettlement`, or batch settlement helpers). If any such function is externally callable and lacks `nonReentrant`, a malicious token/recipient could re-enter DVP during `executeSettlementInner` (via ERC‑721/1155 receiver hooks or crafted token behavior) and mutate state in the same transaction.

Mitigations for future work:
- Keep all external/public mutating functions `nonReentrant`.
- Preserve the invariant that `executeSettlementInner` remains self-call only; avoid adding external pathways to settlement mutation without the guard.
- When adding features mentioned in `ROADMAP.md` (e.g., PERMIT2, netting, ERC‑1155), ensure new entrypoints cannot be called from token hooks to affect in-flight settlement execution, or guard them with `nonReentrant` and clear access control.

### Off-chain UI rendering risks (Informational)

`settlementReference` is arbitrary user-supplied data stored on-chain. If a UI renders this value unsafely (e.g., via `innerHTML`/React `dangerouslySetInnerHTML`), a malicious payload could execute in users' browsers (stored XSS). This does not affect on-chain security but can compromise end users of dapps, explorers, ops dashboards, emails, or PDFs that display the field.

Mitigations (UI):
- Render as plain text (escape HTML) by default; avoid raw HTML injection.
- If rich text is required, sanitize strictly (e.g., DOMPurify with a restrictive policy).
- Enforce a strong Content Security Policy (disallow inline scripts; use nonces).
- Consider length/character limits to reduce attack surface.
