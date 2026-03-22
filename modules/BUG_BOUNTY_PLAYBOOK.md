---
title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)4 — Bug Bounty Hunter's Playbook: Strategy, Tactics & Income"
description: "Comprehensive bug bounty hunter's operational guide: target selection scoring matrix, 48-hour sprint methodology, 'Follow the Money' asset flow analysis, competitive audit winning strategies (Code4rena/Sherlock), Immunefi-specific submission tactics, reputation building roadmap, mental models for finding bugs, automated recon scripts, and realistic income calibration with historical payout examples."
keywords: "bug bounty strategy, Immunefi tactics, Code4rena winning, bug hunting methodology, web3 security career, smart contract bug bounty, security researcher income, audit competition strategy, finding bugs methodology, smart contract target selection, 48-hour audit sprint, Follow the Money DeFi exploit"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)4 — Bug Bounty Playbook | Web3 Hacker Guide"
og_description: "Operational bug bounty guide: target selection, 48-hour sprint methodology, Immunefi tactics, Code4rena winning strategies, income calibration, and reputation building roadmap."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/14_BUG_BOUNTY_PLAYBOOK"
schema_type: "TechArticle"
difficulty: " Advanced →  Expert"
module: 14
tags: [bug-bounty, Immunefi, Code4rena, Sherlock, security-career, audit-strategy, web3-income, bug-hunting]
nav_order: 14
parent: "Web3 Hacker & Pentester Guide"
---

# Module 14 — Bug Bounty Hunter's Playbook

> **Difficulty:** Advanced →  Expert
>
> This module is the operational guide for turning your skills into consistent bug bounty income. It covers target selection, triage speed, submission strategy, and the mental models used by top earners on Immunefi, Code4rena, and Sherlock.

---

## 14.1 Target Selection Strategy

### Immunefi Target Scoring Matrix

Score each target before investing time. Pick targets with the highest score.

| Factor | Score 1 | Score 2 | Score 3 |
|--------|---------|---------|---------|
| **Max bounty** | < $50K | $50K–$500K | > $500K |
| **Time since last audit** | < 3 months | 3–12 months | > 12 months |
| **Code changes since audit** | None | Minor | Major |
| **Protocol complexity** | Simple | Moderate | Complex (more surface) |
| **Your expertise match** | Low | Medium | High |
| **Competition level** | Many hunters | Some hunters | Niche/new |

### High-Value Target Characteristics

```
Prioritize protocols that:
1. Have large TVL (Total Value Locked) — more at stake = higher bounties
2. Recently launched or upgraded (fresh code = fresh bugs)
3. Use novel mechanisms (less audited patterns)
4. Have complex cross-protocol integrations
5. Use non-standard token types (fee-on-transfer, rebasing)
6. Have bridge components (highest-value attack surface)
7. Have governance with treasury access
```

### Where to Find Targets

| Platform | URL | Best For |
|----------|-----|---------|
| **Immunefi** | immunefi.com | Highest payouts, live protocols |
| **Code4rena** | code4rena.com | Competitive audits, track record |
| **Sherlock** | sherlock.xyz | Structured judging, consistent payouts |
| **Cantina** | cantina.xyz | High-quality protocols, invite-based |
| **Hats Finance** | hats.finance | Decentralized bug bounties |
| **Spearbit** | spearbit.com | Elite audits (invite-only) |

---

## 14.2 The 48-Hour Sprint Methodology

### Hour 0–4: Rapid Reconnaissance

```bash
# 1. Clone the repository
git clone https://github.com/protocol/contracts && cd contracts

# 2. Get a quick overview
cloc contracts/src/ --include-lang=Solidity
forge build --sizes

# 3. Run automated tools in background (let them run while you read)
slither . --json slither.json &
aderyn . &

# 4. Read the documentation
# - README.md
# - docs/ folder
# - Any linked whitepaper or spec

# 5. Map the architecture
# - List all contracts and their roles
# - Identify entry points (external/public functions)
# - Find the money flow (deposit → storage → withdrawal)
```

### Hour 4–16: Deep Manual Review

```
Priority order for manual review:
1. Functions that move funds (transfer, withdraw, claim)
2. Access control (who can call what)
3. Oracle integrations (price feeds, TWAP)
4. External calls (reentrancy surface)
5. Math operations (overflow, precision, rounding)
6. Initialization (constructors, initializers)
7. Upgrade mechanisms (proxy patterns)
8. Cross-contract interactions
```

### Hour 16–36: Attack Brainstorming

```
For each function, ask:
[ ] Can I call this with unexpected inputs?
[ ] Can I call this in an unexpected order?
[ ] Can I call this from an unexpected context (flash loan, callback)?
[ ] What happens at boundary values (0, max_uint, 1 wei)?
[ ] What if the oracle returns 0? Negative? Max value?
[ ] What if a token transfer fails silently?
[ ] What if I'm the first depositor? The last?
[ ] What if I deposit and withdraw in the same transaction?
[ ] What if two users interact simultaneously?
[ ] What if the protocol is paused mid-operation?
```

### Hour 36–48: PoC Writing & Submission

```bash
# Write Foundry PoC for each finding
forge test --mt test_myFinding -vvvv --fork-url $ETH_RPC

# Classify severity using Immunefi model
# Write clear, professional report ([Module 11](./REPORTING_AND_RESPONSIBLE_DISCLOSURE.md) template)
# Submit through official channel
```

---

## 14.3 The "Follow the Money" Framework

### Asset Flow Analysis

Every DeFi exploit ultimately involves moving assets from the protocol to the attacker. Map every possible path:

```
ENTRY POINTS (how assets enter):
├── deposit(amount)
├── depositWithPermit(amount, sig)
├── receive() / fallback()
└── flash loan callbacks

STORAGE (how assets are tracked):
├── balances[user]
├── shares[user]
├── positions[user]
└── collateral[user]

EXIT POINTS (how assets leave):
├── withdraw(amount)
├── claim()
├── liquidate(user)
├── emergencyWithdraw()
└── governance execution

ATTACK QUESTION: Can I manipulate the accounting between entry and exit?
```

### Value Extraction Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| **Inflate then drain** | Inflate your balance/shares, then withdraw more than deposited | ERC-4626 inflation attack |
| **Borrow without repaying** | Manipulate collateral value to borrow more than you can repay | Oracle manipulation |
| **Liquidate at profit** | Force liquidation of your own position at a favorable rate | Self-liquidation |
| **Governance drain** | Pass a proposal that transfers treasury to you | Beanstalk |
| **Fee extraction** | Manipulate fee calculations to extract more than entitled | Rounding attacks |
| **Reward manipulation** | Claim rewards you didn't earn | Flash stake attacks |

---

## 14.4 Competitive Audit Winning Strategies

### Code4rena / Sherlock Tactics

```
Day 1: Architecture + automated tools
  - Run Slither, Aderyn, Semgrep
  - Read all docs and previous audits
  - Map the contract architecture
  - Identify the 3 most complex/risky contracts

Day 2-3: Deep dive on high-risk areas
  - Focus on: oracle integrations, flash loan paths, upgrade mechanisms
  - Write down every assumption the code makes
  - Challenge each assumption

Day 4-5: Economic attack modeling
  - Model the protocol as an economic system
  - Find where incentives misalign
  - Test with extreme values (flash loan amounts, zero values)

Day 6-7: PoC writing + submission
  - Write Foundry tests for every finding
  - Classify severity carefully (over-classification gets downgraded)
  - Submit early — duplicates go to first submitter
```

### Finding Uniqueness

```
High-value unique findings come from:
1. Reading the SPEC/whitepaper and finding deviations from implementation
2. Cross-contract interactions (most auditors focus on single contracts)
3. Economic invariants (not just code bugs)
4. Integration assumptions (what happens when dependency behaves unexpectedly)
5. Edge cases in math (first/last depositor, zero amounts, max amounts)
```

### Severity Calibration

```
Common mistakes that get findings downgraded:
- Calling something Critical when it requires admin key compromise
- Calling something High when it requires specific market conditions
- Combining multiple issues into one finding (split them)
- Not quantifying the actual financial impact
- Missing the root cause (treating a symptom as the bug)

Common mistakes that leave money on the table:
- Calling something Medium when it can drain all funds
- Not writing a PoC (theoretical findings get lower severity)
- Missing the full impact (only showing partial exploit)
```

---

## 14.5 Immunefi-Specific Strategy

### Scope Analysis

```bash
# Before starting, read the scope carefully:
# 1. Which contracts are in scope?
# 2. Which chains are in scope?
# 3. What is explicitly OUT of scope?
# 4. Are there any known issues listed?
# 5. What is the safe harbor policy?

# Common out-of-scope items:
# - Centralization risks (admin key compromise)
# - Issues requiring social engineering
# - Frontend/UI bugs (unless they lead to fund loss)
# - Gas optimizations
# - Theoretical issues without PoC
```

### Submission Quality Checklist

```
Before submitting to Immunefi:
[ ] Is this in scope? (Check the bounty page carefully)
[ ] Is this a known issue? (Check GitHub issues, previous audits)
[ ] Do I have a working PoC? (Foundry test on mainnet fork)
[ ] Have I quantified the impact in dollar terms?
[ ] Have I described the attack prerequisites clearly?
[ ] Have I suggested a specific fix?
[ ] Is my severity classification accurate?
[ ] Have I included all relevant contract addresses?
[ ] Have I tested on the correct network/block?
```

### Negotiation Tactics

```
If your finding is downgraded:
1. Provide additional evidence (more detailed PoC, real-world scenario)
2. Reference similar findings that received higher severity
3. Quantify the exact financial impact more precisely
4. Escalate through Immunefi's mediation process if needed
5. Be professional — the relationship matters for future submissions

If your finding is marked as duplicate:
1. Check if your submission has unique aspects
2. If you submitted first, provide timestamp evidence
3. If you found additional impact, submit as a separate finding
```

---

## 14.6 Building Your Reputation

### Track Record Building Path

```
Month 1-3: CTF Foundations
  - Complete all Ethernaut levels
  - Complete Damn Vulnerable DeFi
  - Solve 5+ Paradigm CTF challenges
  - Publish writeups on Mirror/Medium

Month 4-6: First Competitive Audits
  - Enter 3-5 Code4rena contests
  - Focus on finding 1 valid Medium per contest
  - Study all judged findings after each contest
  - Build your finding database

Month 7-12: Consistent Performance
  - Target 1 High per contest
  - Start submitting to Immunefi (smaller bounties first)
  - Develop specialization (DeFi, bridges, ZK, etc.)
  - Network in Secureum/Immunefi Discord

Year 2+: Senior Level
  - Consistent High/Critical findings
  - Solo audits for smaller protocols
  - Invited to private audits
  - Publish original research
```

### Portfolio Building

```
What to publish:
1. CTF writeups (shows technical depth)
2. Exploit recreations (shows you understand real attacks)
3. Vulnerability research (original findings in test environments)
4. Tool contributions (custom Slither detectors, Echidna properties)
5. Educational content (threads, articles explaining complex topics)

Where to publish:
- Mirror.xyz (Web3-native blogging)
- GitHub (code and writeups)
- Twitter/X (short-form research threads)
- Personal blog
```

---

## 14.7 Mental Models for Finding Bugs

### The "What If" Framework

For every function, systematically ask:

```
INPUTS:
- What if amount = 0?
- What if amount = type(uint256).max?
- What if to = address(0)?
- What if to = address(this)?
- What if to = msg.sender?
- What if the token is fee-on-transfer?
- What if the token reverts on transfer?

TIMING:
- What if this is called before initialization?
- What if this is called after the protocol is paused?
- What if this is called in the same block as another function?
- What if this is called during a flash loan?
- What if this is called from a callback?

STATE:
- What if totalSupply = 0?
- What if the pool is empty?
- What if the oracle returns 0?
- What if the oracle returns max value?
- What if the oracle is stale?

PERMISSIONS:
- What if msg.sender = address(0)?
- What if msg.sender = the contract itself?
- What if msg.sender = a contract (not EOA)?
- What if tx.origin != msg.sender?
```

### The "Invariant Violation" Framework

```
For every protocol, identify 5 invariants that MUST hold:

Example for a lending protocol:
1. totalBorrowed ≤ totalDeposited (no insolvency)
2. userCollateralValue ≥ userDebtValue × liquidationThreshold (no undercollateralization)
3. sum(userBalances) = contractTokenBalance (no accounting mismatch)
4. Only authorized addresses can change protocol parameters
5. Liquidation always improves protocol health

Then ask: "How can I violate each invariant?"
```

### The "Attacker's Perspective" Framework

```
Think like an attacker with:
- Unlimited capital (flash loans)
- Multiple addresses
- Ability to be a validator (control tx ordering)
- Knowledge of all pending transactions (mempool)
- Ability to call any function in any order

Ask: "If I had $1 billion for one transaction, what would I do?"
```

---

## 14.8 Tools for Speed

### Automated Recon Script

```bash
#!/bin/bash
# quick-audit.sh — Run at the start of every engagement

TARGET_DIR=${1:-.}
echo "=== Quick Audit Pipeline for $TARGET_DIR ==="

# Count lines of code
echo "\n[1/6] Lines of Code:"
cloc $TARGET_DIR --include-lang=Solidity 2>/dev/null | tail -5

# Build
echo "\n[2/6] Building..."
forge build --quiet 2>&1 | tail -3

# Slither
echo "\n[3/6] Slither (high/medium only)..."
slither $TARGET_DIR --filter-paths "test|mock|lib" \
  --detect reentrancy-eth,reentrancy-no-eth,arbitrary-send-eth,\
controlled-delegatecall,suicidal,unprotected-upgrade,unchecked-lowlevel,\
unchecked-transfer,weak-prng 2>/dev/null

# Aderyn
echo "\n[4/6] Aderyn..."
aderyn $TARGET_DIR --output aderyn-report.md 2>/dev/null

# Coverage
echo "\n[5/6] Test Coverage..."
forge coverage --report summary 2>/dev/null | grep -E "File|Total"

# Storage layouts
echo "\n[6/6] Contract Sizes..."
forge build --sizes 2>/dev/null | grep -v "^$"

echo "\n=== Pipeline Complete ==="
```

### Finding Tracker Template

```markdown
# Audit Findings — [Protocol Name]

## Critical
| ID | Title | Contract | Line | Status |
|----|-------|----------|------|--------|
| C-01 | | | | Investigating |

## High
| ID | Title | Contract | Line | Status |
|----|-------|----------|------|--------|

## Medium
| ID | Title | Contract | Line | Status |
|----|-------|----------|------|--------|

## Notes / Potential Issues
- [ ] Check oracle staleness in VaultCore.sol:142
- [ ] Verify reentrancy guard on withdraw()
- [ ] Flash loan path through deposit() → borrow()
```

---

## 14.9 Real Payout Examples & Calibration

### Immunefi Historical Payouts (Reference)

| Protocol | Finding | Payout | Severity |
|----------|---------|--------|---------|
| Wormhole | Signature bypass | $10M | Critical |
| Aurora | ETH theft via EVM bug | $6M | Critical |
| Polygon | Double-spend bug | $2M | Critical |
| Optimism | Infinite ETH mint | $2M | Critical |
| Arbitrum | Sequencer bypass | $400K | Critical |
| Compound | Oracle manipulation | $150K | High |
| Aave | Interest rate bug | $100K | High |

### Calibrating Your Expectations

```
Realistic income trajectory:
Year 1: $0–$10K (learning, first findings)
Year 2: $10K–$100K (consistent mediums, first highs)
Year 3: $50K–$500K (regular highs, occasional criticals)
Year 4+: $200K–$2M+ (top-tier researcher)

Key insight: 80% of income comes from 20% of findings
One Critical finding can equal 6 months of Medium findings
Focus on finding the one Critical, not ten Mediums
```

---

*← [Previous: Missing Vuln Classes](./MISSING_VULN_CLASSES.md) | [Next: Real Exploit Recreations →](./EXPLOIT_RECREATIONS.md)*


<script>
document.addEventListener("DOMContentLoaded", function() {
  const checkboxes = document.querySelectorAll('.task-list-item-checkbox, input[type="checkbox"]');
  checkboxes.forEach(function(cb) {
    cb.removeAttribute('disabled');
    cb.style.cursor = 'pointer';
  });
});
</script>
