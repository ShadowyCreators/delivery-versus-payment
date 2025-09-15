<!DOCTYPE html>
<html>
<head>
<style>
    .full-page {
        width:  100%;
        height:  100vh;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
    .full-page img {
        max-width:  200;
        max-height:  200;
        margin-bottom: 5rem;
    }
    .full-page div{
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
</style>
</head>
<body>

<div class="full-page">
    <img src="./shadowyLogo-square.svg" alt="Shadowy Creators Logo">
    <div>
    <h1> Delivery Versus Payment Audit Report</h1>
    <h3>Prepared by: Cyfrin</h3>
    </div>
</div>

</body>
</html>

# Delivery Versus Payment Audit Report

<!-- Your report starts here! -->

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
- [Medium](#medium)
- [Low](#low)
- [Informational](#informational)
- [Gas](#gas)

# Protocol Summary

Protocol does X, Y, Z

# Disclaimer

The Shadowy Creators team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

# Findings

## Low

### [L-1] ERC-20 minimal validation let users exposed to malicious tokens

**Description:** ERC-20 minimal validation is subsceptible to false positives or false negatives, potentially exposing users to malicious tokens.
1. False Positives (Non-ERC20 contracts that pass validation)
- Contracts that implement `decimals()` function but are not ERC20 tokens (e.g., custom contracts with decimal precision)

2. False Negatives (Legitimate ERC20 tokens that fail validation):
- Very old ERC20 tokens that don't implement the `decimals()` function (extremely rare)
- ERC20 tokens with non-standard `decimals()` implementations (e.g., returning different data types)
- Malicious tokens that intentionally break the `decimals()` function

**Risks:** 
- Malicious actors could deploy contracts that pass ERC20 validation but behave unexpectedly during execution
- Status query failures: `getSettlementPartyStatus()` could revert if tokens don't implement `allowance()` or `balanceOf()`
- Users should still verify token contracts independently before creating settlements

**Recommended Mitigation:** 
- Add calls in `_isERC2()` to other standard ERC20 functions

**Acknowledgement:** 
- The potential risk is acknowledged at lines 395-396 DeliveryVersusPaymentV1.sol

---

### [L-2] Token upgradeability allows post-creation malicious behavior

**Description:** Upgradeable or governance-controlled tokens can become malicious after settlement creation, since the contract performs no re-validation of token contracts at execution time.

**Risks:** 
- Proxy-based tokens can be upgraded to malicious implementations that steal funds, revert transfers, or manipulate balances
- Governance-controlled tokens can have their logic changed to block transfers or redirect funds
- ERC20 tokens validated at creation via `decimals()` check can later remove this function or change behavior
- Settlement parties may lose funds or have settlements become unexecutable due to post-creation token changes

**Recommended Mitigation:** 
- Implement token allowlisting for high-value settlements
- Consider re-validating token contracts at execution time
- Add monitoring for token upgrade events
- Document risks clearly for users dealing with upgradeable tokens

**Acknowledgement:** 
- Risk acknowledged in AUDIT.md token upgradeability section
- Current design assumes token contracts remain static post-creation

---

### [L-3] NFT transfers use ERC20 method instead of safe transfer

**Description:** When transferring NFTs (`isNFT == true`), the contract casts to `IERC20` and uses `SafeERC20.safeTransferFrom()`, which resolves to basic `transferFrom()` for ERC721 tokens, bypassing the `onERC721Received` callback mechanism. Note that NFTs are validated via ERC165 `supportsInterface` check at creation time, ensuring only legitimate ERC721 contracts can be marked as NFTs.

**Risks:** 
- Missing `onERC721Received` callbacks for recipient contracts that expect to be notified of NFT receipts
- Does not follow ERC721 best practices for safe transfers with callback verification
- Primarily a UX/compliance issue rather than a security vulnerability, as legitimate ERC721s support basic `transferFrom`

**Recommended Mitigation:** 
- When `isNFT == true`, use `IERC721(flow.token).safeTransferFrom(flow.from, flow.to, flow.amountOrId)` instead of casting to IERC20
- Import `IERC721` interface and add proper NFT handling logic

**Acknowledgement:** 
- Current implementation at lines 307-322 treats all tokens uniformly via IERC20 interface

---

### [L-4] Unbounded loops enable gas-based denial of service

**Description:** Multiple functions contain unbounded loops over flows array and settlement IDs arrays without gas limit considerations. Large settlements or batch operations can exceed block gas limits.

**Risks:** 
- Settlements with too many flows become unexecutable within block gas limits
- Batch operations on many settlements can fail due to gas exhaustion
- Griefing: malicious actors can create settlements with excessive flows to waste gas of executors
- Legitimate large settlements may become permanently unexecutable

**Recommended Mitigation:** 
- Consider adding soft caps on number of flows per settlement
- Document gas considerations clearly for users
- Implement pagination for batch operations where feasible

**Acknowledgement:** 
- Known issue documented in README.md lines 121-122: "The current chain's block gas limit acts as a cap. In every case it is the caller's responsibility to ensure that the gas requirement can be met."

---

## Informational

### [I-1] Off-chain stored XSS risk from settlement reference

**Description:** The `settlementReference` field accepts arbitrary strings that are stored on-chain and may be rendered in user interfaces without proper sanitization.

**Risks:** 
- Malicious actors can inject HTML/JavaScript into settlement references
- If UIs render these strings without escaping, stored XSS attacks are possible
- Particularly risky when attackers can create settlements involving other parties and bypass UI input validation

**Recommended Mitigation:** 
- UI implementations must sanitize/escape settlementReference before rendering
- Consider Content Security Policy (CSP) headers
- Optionally add on-chain length limits or character set restrictions

**Acknowledgement:** 
- This is an off-chain UI security concern, not a smart contract vulnerability

---