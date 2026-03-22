---
title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)6 — Master Smart Contract Audit Checklist"
description: "Complete master audit checklist for smart contract security: pre-audit setup, per-function review (access control, input validation, CEI pattern, external calls, math), token handling (weird ERC-20, fee-on-transfer, rebasing), oracle validation, proxy and upgradeable contract review, DeFi-specific checklists (AMM, lending, vault/ERC-4626, governance, bridge), signature/cryptography, gas/DoS, MEV, economic security, and final pre-submission verification."
keywords: "smart contract audit checklist, security review checklist, ERC-4626 checklist, oracle validation checklist, proxy audit checklist, CEI pattern, audit methodology, DeFi security checklist, token handling audit, MEV checklist, economic security checklist, gas limit bypass, signature cryptography checklist"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)6 — Master Audit Checklist | Web3 Hacker Guide"
og_description: "Complete smart contract audit checklist: access control, CEI pattern, oracle validation, token quirks, proxy review, DeFi-specific checks, MEV, economic security, and pre-submission verification."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/16_AUDIT_CHECKLIST_MASTER"
schema_type: "TechArticle"
difficulty: "All Levels"
module: 16
tags: [audit-checklist, security-review, CEI-pattern, oracle-check, proxy-audit, token-handling, ERC-4626, DeFi-checklist]
nav_order: 16
parent: "Web3 Hacker & Pentester Guide"
---

# Module 16 — Master Audit Checklist

> **Difficulty:** All Levels
>
> This is your go-to reference during every audit. Print it, bookmark it, run through it systematically. Missing a single item on this list has cost auditors millions in missed findings.

---

## 16.1 Pre-Audit Setup Checklist

- [ ] Clone repository and verify commit hash matches scope
- [ ] Run: forge build (confirm no compilation errors)
- [ ] Run: cloc contracts/src/ (understand codebase size)
- [ ] Run: slither . (automated first pass)
- [ ] Run: aderyn . (second automated pass)
- [ ] Run: forge coverage (understand test coverage)
- [ ] Read ALL documentation (README, docs/, whitepaper)
- [ ] Read ALL previous audit reports
- [ ] Run: git log --oneline -50 (recent changes)
- [ ] Run: git diff AUDITED_COMMIT..HEAD (changes since last audit)
- [ ] Map all contracts and their relationships
- [ ] Identify all external dependencies (oracles, DEXes, bridges)
- [ ] Identify all privileged roles (owner, admin, guardian, etc.)


---

## 16.2 Per-Function Checklist

Run this for EVERY external and public function:

### Access Control
- [ ] Is there appropriate access control?
- [ ] Does it use msg.sender (not tx.origin)?
- [ ] Are role checks correct (right role for the action)?
- [ ] Can the access control be bypassed via delegatecall?
- [ ] Is there a two-step ownership transfer (not instant)?
- [ ] Are there any unprotected initializer functions?


### Input Validation
- [ ] Are all inputs validated?
- [ ] What happens with amount = 0?
- [ ] What happens with amount = type(uint256).max?
- [ ] What happens with address = address(0)?
- [ ] What happens with address = address(this)?
- [ ] Are array lengths validated (matching lengths, max length)?
- [ ] Are deadlines validated (not expired, not too far future)?


### State Updates (CEI Pattern)
- [ ] Are all state updates done BEFORE external calls?
- [ ] Is there a reentrancy guard where needed?
- [ ] Can the function be called recursively?
- [ ] Can the function be called from a callback?
- [ ] Is there cross-function reentrancy risk?
- [ ] Is there read-only reentrancy risk (view functions called by others)?


### External Calls
- [ ] Is the return value of every external call checked?
- [ ] Is SafeERC20 used for token transfers?
- [ ] Is there a gas limit on external calls?
- [ ] Can the external call reenter?
- [ ] What happens if the external call reverts?
- [ ] What happens if the external call returns unexpected data?
- [ ] Is the called address trusted? Can it be changed?


### Math
- [ ] Is multiplication done before division?
- [ ] Are there any division-by-zero possibilities?
- [ ] Can any value overflow (even in Solidity 0.8+ with unchecked)?
- [ ] Is rounding direction correct (round against user for safety)?
- [ ] Are there precision loss issues with small amounts?
- [ ] Are there precision loss issues with large amounts?
- [ ] Is the math correct for edge cases (first/last depositor)?


---

## 16.3 Token Handling Checklist

- [ ] Does the protocol handle fee-on-transfer tokens?
-    → Measure balance before/after transfer, not trust the amount
- [ ] Does the protocol handle rebasing tokens?
-    → Use shares, not absolute balances
- [ ] Does the protocol handle ERC-777 tokens?
-    → Reentrancy via tokensReceived hook
- [ ] Does the protocol handle tokens with non-standard decimals?
-    → USDC = 6, WBTC = 8, not 18
- [ ] Does the protocol handle tokens that return false instead of reverting?
-    → Use SafeERC20
- [ ] Does the protocol handle pausable tokens?
-    → What happens if USDC is paused?
- [ ] Does the protocol handle blacklistable tokens?
-    → What if the contract address is blacklisted?
- [ ] Does the protocol handle tokens with multiple entry points?
-    → TUSD has two addresses
- [ ] Does the protocol handle tokens with approval race conditions?
-    → USDT requires setting to 0 before changing allowance
- [ ] Does the protocol handle tokens with no return value?
-    → USDT on mainnet


---

## 16.4 Oracle Checklist

- [ ] What oracle(s) does the protocol use?
- [ ] Is the price validated for staleness?
-    → require(block.timestamp - updatedAt <= heartbeat)
- [ ] Is the price validated for completeness?
-    → require(updatedAt != 0)
- [ ] Is the price validated for positivity?
-    → require(answer > 0)
- [ ] Is the answeredInRound checked?
-    → require(answeredInRound >= roundId)
- [ ] Is the L2 sequencer uptime checked? (for Arbitrum, Optimism, etc.)
- [ ] Does the feed have minAnswer/maxAnswer circuit breakers?
-    → If so, what happens when real price is outside bounds?
- [ ] Is a TWAP used instead of spot price?
-    → What is the TWAP window? Is it long enough?
- [ ] Can the oracle be manipulated via flash loan?
- [ ] What happens if the oracle returns 0?
- [ ] What happens if the oracle returns max uint?
- [ ] Is there a fallback oracle if the primary fails?


---

## 16.5 Proxy & Upgradeable Contract Checklist

- [ ] What proxy pattern is used? (Transparent, UUPS, Beacon, Diamond)
- [ ] Are EIP-1967 storage slots used for proxy variables?
- [ ] Is there a storage collision between proxy and implementation?
- [ ] Is the implementation initialized? (not just the proxy)
- [ ] Is _disableInitializers() called in the implementation constructor?
- [ ] Is there a timelock on upgrades?
- [ ] Who can trigger upgrades? (multisig? timelock? single EOA?)
- [ ] Is the upgrade function protected against reentrancy?
- [ ] For UUPS: Is the upgrade function in the implementation (not proxy)?
- [ ] For Diamond: Are facet selectors correctly mapped?
- [ ] For Diamond: Can selectors clash between facets?
- [ ] What is the storage layout? Does it match across versions?
- [ ] Are __gap variables used for future storage slots?


---

## 16.6 DeFi-Specific Checklist

### AMM / DEX
- [ ] Is slippage protection enforced?
- [ ] Is there a deadline parameter?
- [ ] Is the price oracle manipulation-resistant?
- [ ] Can the pool be drained via flash swap?
- [ ] Are fee calculations correct?
- [ ] Can liquidity be added/removed atomically to manipulate price?


### Lending Protocol
- [ ] Is collateral value calculated using a manipulation-resistant oracle?
- [ ] Are borrow caps enforced?
- [ ] Is the liquidation threshold correct?
- [ ] Is the liquidation bonus reasonable (not too high)?
- [ ] Can a user self-liquidate for profit?
- [ ] Can bad debt be created?
- [ ] Is there a minimum position size?
- [ ] What happens during a market crash (cascading liquidations)?
- [ ] Is interest accrual correct?
- [ ] Can interest overflow?


### Vault / ERC-4626
- [ ] Is the first depositor protected against inflation attack?
-    → Virtual shares/assets offset, or minimum deposit
- [ ] Is rounding direction correct?
-    → Deposits: round down (user gets fewer shares)
-    → Withdrawals: round up (user pays more assets)
- [ ] Can the share price be manipulated via donation?
- [ ] Is there a withdrawal fee or vesting period?
- [ ] Can the vault be sandwiched (deposit before harvest, withdraw after)?
- [ ] Is the totalAssets() calculation correct?
- [ ] What happens if the underlying strategy loses funds?


### Governance
- [ ] Is voting based on snapshots (not current balance)?
- [ ] Is there a timelock between proposal and execution?
- [ ] Is the proposal threshold high enough to prevent spam?
- [ ] Can a flash loan acquire enough voting power?
- [ ] Are emergency functions protected by multisig?
- [ ] Can proposals be executed before the voting period ends?
- [ ] Is there a quorum requirement?
- [ ] Can the governance contract be upgraded via governance?
-    → Circular dependency risk


### Bridge
- [ ] How are cross-chain messages verified?
- [ ] Is there replay protection (nonce, message hash)?
- [ ] Is the chain ID included in message verification?
- [ ] Is the destination contract address included?
- [ ] What is the validator/relayer trust model?
- [ ] What happens if a message is never delivered?
- [ ] What happens if the destination chain runs out of gas?
- [ ] Is there a timeout/expiry mechanism?
- [ ] Can the same message be processed twice?
- [ ] Is the bridge contract upgradeable? Who controls upgrades?


---

## 16.7 Signature & Cryptography Checklist

- [ ] Is EIP-712 used for typed data signing?
- [ ] Does the domain separator include chainId?
- [ ] Does the domain separator include verifyingContract?
- [ ] Is there a nonce to prevent replay?
- [ ] Is there a deadline to prevent stale signatures?
- [ ] Is ECDSA.recover() used (not raw ecrecover)?
-    → OpenZeppelin's version enforces lower-s (prevents malleability)
- [ ] Are signatures validated against the correct signer?
- [ ] Can a signature be used on a different contract?
- [ ] Can a signature be used on a different chain?
- [ ] Can a signature be used multiple times?
- [ ] Is the signed data correctly encoded (abi.encode vs abi.encodePacked)?
-    → abi.encodePacked can cause hash collisions with dynamic types


---

## 16.8 Gas & DoS Checklist

- [ ] Are there unbounded loops?
-    → What is the maximum array size? Can it exceed block gas limit?
- [ ] Are there push patterns that could be DoS'd?
-    → Use pull pattern instead
- [ ] Can ETH be force-sent to break invariants?
-    → Don't rely on address(this).balance == X
- [ ] Can a malicious contract's receive() cause gas griefing?
- [ ] Is there a return bomb risk?
-    → Limit returndata size from external calls
- [ ] Can the contract be permanently DoS'd?
- [ ] Are gas limits set on external calls?
- [ ] Can block stuffing delay time-sensitive operations?


---

## 16.9 MEV & Ordering Checklist

- [ ] Are there operations that are profitable to front-run?
- [ ] Are there operations that are profitable to sandwich?
- [ ] Are there operations that depend on transaction ordering?
- [ ] Is there a commit-reveal scheme where needed?
- [ ] Are slippage limits enforced on all swaps?
- [ ] Are deadline parameters enforced?
- [ ] Can a validator manipulate block.timestamp to their advantage?
- [ ] Can a validator manipulate block.prevrandao to their advantage?
- [ ] Are there any operations that should use private mempool?


---

## 16.10 Economic Security Checklist

- [ ] What are the protocol's economic invariants?
-    → List them explicitly and verify each one
- [ ] Can a flash loan break any invariant?
- [ ] Can a large deposit/withdrawal break any invariant?
- [ ] Are there any arbitrage opportunities that drain the protocol?
- [ ] Are incentives aligned? (liquidators, arbitrageurs, governance)
- [ ] Can the protocol be drained via repeated small operations?
-    → Rounding errors accumulate
- [ ] Is the protocol solvent under all market conditions?
- [ ] What happens during extreme market volatility?
- [ ] Are there any circular dependencies between protocols?
- [ ] Can the protocol be used to manipulate prices in other protocols?


---

## 16.11 Final Pre-Submission Checklist

- [ ] Every finding has a working Foundry PoC
- [ ] Every finding has a clear severity classification
- [ ] Every finding has a quantified impact (dollar amount)
- [ ] Every finding has a specific recommended fix
- [ ] Every finding references the exact file and line number
- [ ] No duplicate findings (each issue is unique)
- [ ] Findings are ordered by severity (Critical first)
- [ ] Executive summary is accurate
- [ ] All contract addresses are correct
- [ ] Fork block number is specified in PoCs
- [ ] PoCs run successfully with: forge test --mt test_finding -vvvv
- [ ] Report is professional and free of typos


---

## 16.12 Quick Reference — Vulnerability → Tool Mapping

| Vulnerability | Best Detection Method |
|--------------|----------------------|
| Reentrancy | Slither + manual CEI review |
| Access control | Slither + manual function review |
| Integer overflow | Slither (pre-0.8) + manual unchecked review |
| Oracle manipulation | Manual + economic modeling |
| Flash loan attacks | Manual + Foundry fork tests |
| Signature replay | Manual + EIP-712 review |
| Proxy collision | Manual storage layout analysis |
| Upgradeable bugs | Manual + Slither unprotected-upgrade |
| Token quirks | Manual + weird-erc20 checklist |
| DoS | Manual + Foundry gas tests |
| Precision loss | Manual + Foundry fuzz |
| Governance attacks | Manual + economic modeling |
| MEV/front-running | Manual + mempool analysis |
| Compiler bugs | Version check + bytecode verification |
| ZK circuit bugs | Manual circuit review + Halmos |

---

*← [Previous: Exploit Recreations](./EXPLOIT_RECREATIONS.md) | [Back to Index →](./INDEX.md)*


<script>
document.addEventListener("DOMContentLoaded", function() {
  const checkboxes = document.querySelectorAll('.task-list-item-checkbox, input[type="checkbox"]');
  checkboxes.forEach(function(cb) {
    cb.removeAttribute('disabled');
    cb.style.cursor = 'pointer';
  });
});
</script>
