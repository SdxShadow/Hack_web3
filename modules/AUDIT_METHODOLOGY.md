---
title: "Module 04 — Professional Smart Contract Audit Methodology"
description: "Complete professional audit methodology for smart contract security: scoping, threat modeling, control flow analysis, invariant identification, economic attack surface modeling, composability risk assessment, test coverage analysis, and professional finding documentation."
keywords: "smart contract audit methodology, audit workflow, threat modeling DeFi, invariant testing, economic attack modeling, security audit checklist, STRIDE smart contracts, composability risk, audit findings template, formal verification methodology, top-down bottom-up code review DeFi, smart contract threat modeling strategies"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 04 — Smart Contract Audit Methodology | Web3 Hacker Guide"
og_description: "Professional audit workflow for smart contracts: scoping, threat modeling, invariant testing, economic attack surface analysis, and finding documentation used by top-tier audit firms."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/04_AUDIT_METHODOLOGY"
schema_type: "TechArticle"
difficulty: " Intermediate →  Advanced"
module: 4
tags: [audit-methodology, threat-modeling, invariants, economic-attacks, DeFi-audit, security-workflow]
nav_order: 4
parent: "Web3 Hacker & Pentester Guide"
---

# Module 04 — Smart Contract Audit Methodology

> **Difficulty:** Intermediate →  Advanced
>
> A vulnerability is only useful if you can find it consistently. This module covers the professional audit workflow used by top-tier firms like Trail of Bits, OpenZeppelin, and Spearbit. Whether you're conducting solo audits or competing on Code4rena/Sherlock, a systematic methodology is what separates finding 2 bugs from finding 20.

---

## 4.1 Scoping

Before reading a single line of code, scope the engagement properly.

### Scoping Checklist

| Dimension | What to Assess | How to Measure |
|-----------|---------------|----------------|
| **Lines of code (nSLOC)** | Complexity of the codebase | `cloc --exclude-dir=test,node_modules contracts/` |
| **Number of contracts** | Surface area | Count distinct `.sol` files in scope |
| **External dependencies** | Composability risk | Map all `import` paths and external calls |
| **Upgrade patterns** | Proxy complexity | Count proxy contracts, identify upgrade mechanisms |
| **Oracle dependencies** | Oracle attack surface | List all oracle integrations |
| **Token support** | Non-standard token risk | Which ERC-20/721/1155 tokens does the protocol interact with? |
| **Access control complexity** | Privileged role analysis | Map all roles, modifiers, multisigs, timelocks |
| **Previous audits** | Known issues baseline | Read all prior audit reports |
| **Test coverage** | Confidence in existing tests | `forge coverage` or `npx hardhat coverage` |

### Estimating Audit Duration

| nSLOC | Solo Auditor | Team (2–3) | Competitive Audit |
|-------|-------------|-----------|-------------------|
| < 500 | 3–5 days | 2–3 days | 3-day contest |
| 500–2000 | 1–3 weeks | 1–2 weeks | 7-day contest |
| 2000–5000 | 3–6 weeks | 2–4 weeks | 14-day contest |
| 5000+ | 6–12 weeks | 4–8 weeks | 21+ day contest |

```bash
# Count nSLOC using cloc (comment-aware line counter)
cloc contracts/src/ --include-lang=Solidity

# Alternative: Solidity Metrics (VS Code extension)
# Or use forge to get a quick overview
forge build --sizes
```

---

## 4.2 Manual Review Workflow

### Top-Down Analysis (Feature-First)

Start from the user-facing entry points and trace execution flow.

```
1. Identify external/public functions (entry points)
2. For each entry point:
   a. What state does it read?
   b. What state does it modify?
   c. What external calls does it make?
   d. What access control protects it?
   e. Can an attacker influence any input?
3. Follow the data flow through internal functions
4. Map state transitions
5. Identify invariants that should hold
```

### Bottom-Up Analysis (Contract-First)

Start from the core data structures and understand the state model.

```
1. Map the storage layout (all state variables)
2. Identify critical state (balances, ownership, paused state)
3. List all functions that modify critical state
4. Verify access control on each state-modifying function
5. Check initialization of all state variables
6. Verify upgrade safety
```

### Recommended Hybrid Approach

```
Phase 1 (Day 1-2): Architecture review
  ├── Read documentation, README, specs
  ├── Run Slither for initial findings
  ├── Map contract dependencies
  └── Create a mental model of the protocol

Phase 2 (Day 3-5): Detailed code review
  ├── Top-down: Trace all user flows
  ├── Bottom-up: Verify state invariants
  ├── Check all external calls
  └── Review access control model

Phase 3 (Day 6-7): Attack surface analysis
  ├── Identify economic attack vectors
  ├── Test edge cases with Foundry fuzzing
  ├── Review against vulnerability checklist
  └── Cross-reference with known exploit patterns

Phase 4 (Day 8+): Exploit development & reporting
  ├── Write PoCs for findings
  ├── Classify severity
  └── Draft report
```

---

## 4.3 Threat Modeling for DeFi Protocols

### Trust Boundary Mapping

```
┌──────────────────────────────────────────────────┐
│                 EXTERNAL (Untrusted)             │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Users   │  │  Bots    │  │  Flash Loans  │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
├──────────────────────────────────────────────────┤
│              TRUST BOUNDARY                      │
├──────────────────────────────────────────────────┤
│                 PROTOCOL (Partially Trusted)     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Core    │  │  Oracle  │  │  Governance   │  │
│  │  Contracts│  │  Adapter │  │  Module       │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
├──────────────────────────────────────────────────┤
│              TRUST BOUNDARY                      │
├──────────────────────────────────────────────────┤
│                 ADMIN (Trusted)                  │
│  ┌──────────┐  ┌──────────┐                     │
│  │ Multisig │  │ Timelock │                     │
│  └──────────┘  └──────────┘                     │
└──────────────────────────────────────────────────┘
```

### Asset Flow Mapping

For every DeFi protocol, map:
1. **How do assets enter?** (deposit functions)
2. **How are assets stored?** (custody model — contract-held vs delegated)
3. **How do assets move internally?** (rebalancing, strategy allocation)
4. **How do assets exit?** (withdrawal functions, liquidation flows)
5. **Who controls each flow?** (access control on each path)

### STRIDE for Smart Contracts

| Threat | Smart Contract Example |
|--------|----------------------|
| **Spoofing** | `tx.origin` authentication, signature forgery |
| **Tampering** | Storage manipulation via delegatecall, upgradeable proxy |
| **Repudiation** | Missing events for critical state changes |
| **Information Disclosure** | Private variables readable from storage, mempool visibility |
| **Denial of Service** | Gas griefing, unbounded loops, forced ETH |
| **Elevation of Privilege** | Missing access control, initialization bugs |

---

## 4.4 Control Flow & Data Flow Analysis

### Control Flow Analysis

```solidity
// Trace the control flow of a complex function
function liquidate(address borrower) external {
    // 1. Validation checks
    require(isHealthy(borrower) == false, "Cannot liquidate healthy position");

    // 2. State reads
    uint256 debt = getDebt(borrower);
    uint256 collateral = getCollateral(borrower);

    // 3. State modifications
    _clearDebt(borrower);

    // 4. External interactions
    collateralToken.transfer(msg.sender, collateral); // CEI: Is this after state update? [YES]
    debtToken.transferFrom(msg.sender, address(this), debt); // Is this safe?

    // 5. Events
    emit Liquidated(borrower, msg.sender, debt, collateral);
}

// Questions to ask at each step:
// - Can any require() be bypassed?
// - Are the reads and writes in the correct order (CEI)?
// - Can external calls reenter?
// - Is the collateral/debt calculation manipulable?
```

### Data Flow Analysis

Track how user-supplied data flows through the system:

```
User Input (calldata)
  → Function parameter
    → Used in require() check
    → Used in calculation
      → Written to storage
        → Read by another function
          → Used in external call
```

**Key question at each step:** Can an attacker craft input that passes validation but causes unexpected behavior downstream?

---

## 4.5 Invariant Identification & Verification

### What Are Invariants?

Invariants are conditions that must **always** hold true, regardless of what functions are called or in what order.

### Common DeFi Invariants

| Protocol Type | Invariant |
|--------------|-----------|
| **Token** | `sum(balances) == totalSupply` |
| **AMM** | `x * y >= k` (after every swap) |
| **Lending** | `totalBorrowed ≤ totalDeposited` |
| **Vault** | `shares_to_assets(total_shares) ≤ total_assets + dust` |
| **Staking** | `sum(user_stakes) == contract_balance` |
| **Governance** | `totalVotingPower == totalTokenSupply` at snapshot |

### Writing Invariant Tests in Foundry

```solidity
// forge test --mt invariant_
contract InvariantTest is Test {
    Token token;
    Handler handler;

    function setUp() public {
        token = new Token();
        handler = new Handler(token);

        // Target the handler contract for invariant testing
        targetContract(address(handler));
    }

    // This function is called after every random sequence of handler calls
    function invariant_totalSupplyMatchesBalances() public {
        assertEq(token.totalSupply(), handler.ghost_totalMinted() - handler.ghost_totalBurned());
    }

    function invariant_noUserExceedsTotalSupply() public {
        for (uint i = 0; i < handler.actorCount(); i++) {
            assertLe(token.balanceOf(handler.actors(i)), token.totalSupply());
        }
    }
}

contract Handler is Test {
    Token token;
    uint256 public ghost_totalMinted;
    uint256 public ghost_totalBurned;

    function deposit(uint256 amount) external {
        amount = bound(amount, 1, 1e24);
        token.mint(msg.sender, amount);
        ghost_totalMinted += amount;
    }

    function withdraw(uint256 amount) external {
        amount = bound(amount, 0, token.balanceOf(msg.sender));
        token.burn(msg.sender, amount);
        ghost_totalBurned += amount;
    }
}
```

---

## 4.6 State Machine Analysis

### Modeling Contract State

Many contracts implement implicit state machines. Making these explicit helps find invalid state transitions.

```
    ┌─────────┐     deposit()     ┌──────────┐
    │  IDLE   │──────────────────→│ ACTIVE   │
    └─────────┘                   └──────────┘
                                     │
         │            liquidate()     │
         │         ┌──────────────────┘
         │         
    ┌─────────┐   ┌──────────────┐
    │  IDLE   │←──│ LIQUIDATING  │
    └─────────┘   └──────────────┘

Questions:
- Can you call deposit() while LIQUIDATING? (Should be blocked)
- Can you call withdraw() while ACTIVE? (Should be allowed)
- Can the state skip from IDLE to LIQUIDATING? (Should not happen)
- What happens if two state transitions are called in the same tx?
```

---

## 4.7 Economic Attack Surface Modeling

### Categories of Economic Attacks

| Category | Description | Example |
|----------|-------------|---------|
| **Price manipulation** | Changing the price an oracle reports | Flash loan → dumps token → oracle reads low price |
| **Liquidity manipulation** | Adding/removing liquidity to affect operations | JIT liquidity in Uniswap V3 |
| **Collateral manipulation** | Artificially inflating collateral value | Donating tokens to inflate share price |
| **Governance manipulation** | Acquiring voting power temporarily | Flash loan governance (Beanstalk) |
| **Interest rate manipulation** | Changing borrow/supply rates profitably | Large deposit to reduce rates, then borrow cheaply |

### Analysis Framework

For each economic interaction:
1. **Who profits?** Follow the money
2. **Who loses?** Are losses socialized (shared by all depositors)?
3. **What is the capital requirement?** (Flash loans make it effectively zero)
4. **Is the profit extractable atomically?** (Single transaction = flash-loan-able)
5. **What parameters can an attacker control?** (Amounts, timing, order)

---

## 4.8 Integration Risk Analysis (Composability Attacks)

DeFi's biggest risk is composability — protocols built on top of other protocols inherit all downstream risks.

### Integration Audit Checklist

- [ ] **External call safety** — Can external contracts reenter?
- [ ] **Token compatibility** — Does the protocol handle non-standard tokens?
- [ ] **Oracle dependency** — What happens if the oracle returns stale/zero/negative data?
- [ ] **Upgrade dependency** — What if an integrated protocol upgrades its contracts?
- [ ] **Price impact** — Do large operations affect prices used by other components?
- [ ] **Cross-protocol interactions** — Can interactions between protocols create unexpected states?
- [ ] **Gas forwarding** — Is sufficient gas sent to external calls?
- [ ] **Failure handling** — What happens if an integrated protocol is paused or bricked?

---

## 4.9 Test Coverage Analysis

```bash
# Foundry coverage
forge coverage

# Foundry coverage with specific report format
forge coverage --report lcov
# Then view in VS Code with Coverage Gutters extension

# Hardhat coverage
npx hardhat coverage

# What to look for:
# - Lines not covered by tests → untested edge cases
# - Branches not covered → conditional logic untested
# - Functions not covered → entire code paths untested
```

### Coverage Is Necessary But Not Sufficient

100% line coverage does NOT mean the contract is secure. Coverage tells you which code **executed**, not whether the **assertions are correct**. A test that calls every function but never checks the output gives 100% coverage with 0% assurance.

---

## 4.10 Checklist-Driven Audit (SWC Registry)

### SWC Registry Quick Reference

| SWC ID | Name | Severity |
|--------|------|----------|
| SWC-100 | Function Default Visibility | Medium |
| SWC-101 | Integer Overflow/Underflow | High |
| SWC-104 | Unchecked Return Value | High |
| SWC-106 | Unprotected SELFDESTRUCT | Critical |
| SWC-107 | Reentrancy | High-Critical |
| SWC-110 | Assert Violation | Medium |
| SWC-111 | Use of Deprecated Functions | Low |
| SWC-112 | Delegatecall to Untrusted Callee | Critical |
| SWC-113 | DoS with Failed Call | Medium |
| SWC-114 | Transaction Order Dependence | Medium |
| SWC-115 | Authorization through tx.origin | High |
| SWC-116 | Block Timestamp Dependence | Low |
| SWC-120 | Weak Randomness | Medium |
| SWC-123 | Requirement Violation | Medium |
| SWC-124 | Write to Arbitrary Storage | Critical |
| SWC-128 | DoS with Block Gas Limit | Medium |
| SWC-131 | Presence of Unused Variables | Low |
| SWC-136 | Unencrypted Private Data | Medium |

Full registry: [swcregistry.io](https://swcregistry.io/)

---

## 4.11 Writing Professional Audit Findings

### Finding Template

```markdown
## [H-01] Title describing the vulnerability concisely

### Severity
**High** / **Critical** / **Medium** / **Low** / **Informational**

### Description
Technical description of the vulnerability. What is wrong, where is it, and why does it exist?

### Impact
What happens if this is exploited? Quantify the damage:
- Funds at risk: $X locked in the contract
- Affected users: all depositors / specific role
- Exploitability: requires flash loan / requires admin key / anyone can exploit

### Proof of Concept
```solidity
// Foundry test demonstrating the exploit
function test_exploit() public {
    // Setup
    // Action
    // Assertion showing the vulnerability
}
\```

### Recommended Mitigation
```solidity
// Specific code change with before/after
// BEFORE (vulnerable):
function withdraw() external { ... }

// AFTER (fixed):
function withdraw() external nonReentrant { ... }
\```

### Likelihood
- Attack complexity: Low / Medium / High
- Prerequisites: None / Flash loan / Admin key

### References
- Similar exploit: [Rekt News link]
- SWC Registry: SWC-107
```

### Severity Classification

| Severity | Impact | Likelihood | Examples |
|----------|--------|-----------|---------|
| **Critical** | Direct loss of funds, protocol takeover | High (easy to exploit) | Reentrancy draining all funds, unprotected admin functions |
| **High** | Significant fund loss, major protocol disruption | Medium-High | Oracle manipulation, governance attack, signature replay |
| **Medium** | Limited fund loss, protocol degradation | Medium | Precision loss giving small advantage, DoS vectors |
| **Low** | No direct fund loss, minor issues | Low | Missing events, gas optimizations, informational |

---

## 4.12 Audit Report Structure

### Professional Report Structure

```
1. Executive Summary
   ├── Scope (contracts, commit hash, nSLOC)
   ├── Methodology overview
   ├── Summary of findings (Critical: X, High: Y, Medium: Z, Low: W)
   └── Overall risk assessment

2. Table of Contents

3. Scope & Methodology
   ├── Contracts in scope (with addresses if deployed)
   ├── Lines of code
   ├── Tools used
   ├── Review timeline
   └── Audit team

4. System Overview
   ├── Architecture diagram
   ├── Key contracts and their roles
   ├── Trust model
   └── External dependencies

5. Findings
   ├── Critical
   ├── High
   ├── Medium
   ├── Low
   └── Informational

6. Appendix
   ├── Tool output (Slither, Mythril)
   ├── Test coverage report
   └── Disclaimer
```

---

## 4.13 Automated Tools Integration Workflow

### Recommended Tool Pipeline

```bash
#!/bin/bash
# audit-pipeline.sh — Run before manual review

echo "=== Step 1: Compilation ==="
forge build

echo "=== Step 2: Slither (Static Analysis) ==="
slither . --json slither-output.json
slither . --print human-summary

echo "=== Step 3: Slither Detectors (Specific) ==="
slither . --detect reentrancy-eth,reentrancy-no-eth,unchecked-lowlevel,arbitrary-send-eth,\
controlled-delegatecall,suicidal,uninitialized-storage,unprotected-upgrade

echo "=== Step 4: Aderyn (Fast AST Analysis) ==="
aderyn .

echo "=== Step 5: Test Coverage ==="
forge coverage --report summary

echo "=== Step 6: Storage Layout ==="
forge inspect MainContract storage-layout

echo "=== Step 7: Contract Size Check ==="
forge build --sizes

echo "=== Step 8: Gas Report ==="
forge test --gas-report
```

### Triage Process for Automated Findings

```
Automated Finding → Categorize:
├── True Positive (confirmed vulnerability)  → Write up as finding
├── True Positive (known/accepted risk)      → Note in informational
├── False Positive                           → Dismiss with reason
└── Needs Investigation                      → Manual review required
```

> **Key Takeaway:** Tools find ~20–30% of real vulnerabilities. The remaining 70–80% — logic errors, economic exploits, cross-contract interactions — require manual analysis. Use tools as an initial pass to catch low-hanging fruit, then spend the majority of your time on manual review of business logic and economic invariants.

---

*← [Previous: Vulnerabilities](./SMART_CONTRACT_VULNERABILITIES.md) | [Next: Tools & Frameworks →](./TOOLS_AND_FRAMEWORKS.md)*


<script>
document.addEventListener("DOMContentLoaded", function() {
  const checkboxes = document.querySelectorAll('.task-list-item-checkbox, input[type="checkbox"]');
  checkboxes.forEach(function(cb) {
    cb.removeAttribute('disabled');
    cb.style.cursor = 'pointer';
  });
});
</script>
