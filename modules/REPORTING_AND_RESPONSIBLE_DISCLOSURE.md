---
title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)1 — Responsible Disclosure & Professional Security Report Writing"
description: "Complete guide to professional Web3 security reporting and responsible disclosure: vulnerability severity scoring (Immunefi model), finding template writing, executive summary structure, audit report format, CVD timeline, bug bounty platform comparison, legal frameworks (CFAA, safe harbor), white-hat recovery case studies, and disclosure communication templates."
keywords: "responsible disclosure, bug bounty report, vulnerability severity scoring, security report writing, Immunefi submission, CVSS scoring, finding template, audit report structure, white-hat disclosure, security CVD, Safe Harbor legal frameworks CFAA, executive summary smart contract audit, bug bounty negotiation"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)1 — Responsible Disclosure & Bug Bounty Reports | Web3 Hacker Guide"
og_description: "Write professional security reports, navigate responsible disclosure, and maximize bug bounty payouts — with templates, severity scoring, and platform comparisons."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/11_REPORTING_AND_RESPONSIBLE_DISCLOSURE"
schema_type: "TechArticle"
difficulty: " Intermediate"
module: 11
tags: [responsible-disclosure, bug-bounty, security-reporting, Immunefi, finding-template, CVD, audit-report]
nav_order: 11
parent: "Web3 Hacker & Pentester Guide"
---

# Module 11 — Reporting & Responsible Disclosure

> **Difficulty:** Intermediate
>
> Finding a vulnerability is half the battle. Communicating it professionally — with the right severity, clear impact analysis, and actionable recommendations — is what separates hobbyists from professionals. This module covers vulnerability scoring, report writing, disclosure processes, and working with bug bounty platforms.

---

## 11.1 Vulnerability Severity Scoring for Web3

### Immunefi Severity Model

The Immunefi model is the de facto standard for Web3 bug bounty severity classification.

| Severity | Description | Typical Payout Range |
|----------|-------------|---------------------|
| **Critical** | Direct theft of funds, permanent freezing of funds, protocol insolvency | $50K–$10M+ |
| **High** | Theft of unclaimed yield, temporary freezing of funds, unauthorized state changes | $10K–$100K |
| **Medium** | Griefing (no fund loss), contract DoS, gas waste | $1K–$10K |
| **Low** | Contract fails to deliver promised returns, informational | $100–$1K |

### Impact vs. Likelihood Matrix

| | Low Likelihood | Medium Likelihood | High Likelihood |
|--|---------------|-------------------|-----------------|
| **Critical Impact** | High | Critical | Critical |
| **High Impact** | Medium | High | High |
| **Medium Impact** | Low | Medium | Medium |
| **Low Impact** | Informational | Low | Low |

### Web3 Severity Decision Tree

```
1. Can an attacker steal user funds?
   YES → At least HIGH, likely CRITICAL
   NO → Continue

2. Can an attacker freeze/lock user funds permanently?
   YES → CRITICAL (if permanent), HIGH (if temporary)
   NO → Continue

3. Can an attacker cause financial loss to the protocol (bad debt, fund loss)?
   YES → HIGH
   NO → Continue

4. Can an attacker disrupt protocol operation?
   YES → MEDIUM (if recoverable), HIGH (if persistent)
   NO → Continue

5. Is there a deviation from intended behavior?
   YES → LOW or INFORMATIONAL
   NO → Not a finding
```

---

## 11.2 Writing a Finding

### Finding Template

```markdown
## [C-01] Reentrancy in withdraw() allows complete vault drainage

### Severity
**Critical**

### Relevant Contract(s)
- `VaultCore.sol` — Line 142-158

### Description
The `withdraw()` function in `VaultCore.sol` sends ETH to the caller via
`msg.sender.call{value: amount}("")` at line 150 before updating the
user's balance at line 155. This violates the Checks-Effects-Interactions
pattern and allows a malicious contract to re-enter `withdraw()` through
its `receive()` function, draining the vault's entire ETH balance.

### Impact
An attacker can drain 100% of the vault's ETH balance in a single
transaction. As of block 18,500,000, the vault holds 15,420 ETH
(~$35.5M at current prices). The attack requires no special privileges
and can be executed by any user with a smart contract wallet.

**Attack cost:** ~0.01 ETH in gas
**Potential profit:** 15,420 ETH (~$35.5M)

### Proof of Concept

\```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract ReentrancyPoC is Test {
    VaultCore vault;
    address attacker = makeAddr("attacker");

    function setUp() public {
        vault = new VaultCore();
        // Simulate existing deposits
        vm.deal(address(vault), 100 ether);
        vm.deal(attacker, 1 ether);
    }

    function test_reentrancy_drain() public {
        Attacker atk = new Attacker(address(vault));
        vm.deal(address(atk), 1 ether);

        uint256 vaultBefore = address(vault).balance;
        console.log("Vault balance before:", vaultBefore);

        atk.attack();

        uint256 vaultAfter = address(vault).balance;
        console.log("Vault balance after:", vaultAfter);

        assertEq(vaultAfter, 0, "Vault should be drained");
    }
}
\```

### Recommended Mitigation

Option 1 — Apply Checks-Effects-Interactions pattern:
\```solidity
function withdraw() external {
    uint256 balance = balances[msg.sender];
    require(balance > 0, "No balance");
    balances[msg.sender] = 0; // Effect BEFORE interaction
    (bool success, ) = msg.sender.call{value: balance}("");
    require(success, "Transfer failed");
}
\```

Option 2 — Add ReentrancyGuard:
\```solidity
function withdraw() external nonReentrant { ... }
\```

### References
- [SWC-107: Reentrancy](https://swcregistry.io/docs/SWC-107)
- [The DAO Hack (2016)](https://www.gemini.com/cryptopedia/the-dao-hack-makerdao)
```

### Writing Tips from Professional Auditors

| Do | Don't |
|----|-------|
| Quantify impact in dollar terms | Use vague language ("could be bad") |
| Provide working PoC (Foundry test) | Submit theoretical findings without PoC |
| Reference affected code lines precisely | Say "somewhere in Vault.sol" |
| Suggest specific code fixes | Just say "fix the bug" |
| One finding per issue | Combine multiple issues into one finding |
| Use clear, professional language | Use slang or be condescending |

---

## 11.3 Executive Summary Writing

### Template

```markdown
# Security Audit Report — [Protocol Name]

## Executive Summary

### Engagement Overview
| Item | Detail |
|------|--------|
| **Client** | [Protocol Name] |
| **Audit Period** | December 1–14, 2024 |
| **Commit Hash** | `abc1234...` |
| **Scope** | 8 Solidity smart contracts, 2,847 nSLOC |
| **Methods** | Manual review, Slither static analysis, Foundry fuzz testing |
| **Auditor(s)** | [Your name/firm] |

### Findings Summary
| Severity | Count |
|----------|-------|
|  Critical | 1 |
| 🟠 High | 3 |
|  Medium | 5 |
|  Low | 4 |
| [INFO] Informational | 7 |

### Key Observations
The protocol's core lending logic is well-implemented with appropriate use
of OpenZeppelin libraries. However, the oracle integration lacks staleness
checks (H-01), and the withdrawal mechanism is vulnerable to reentrancy
(C-01). We recommend addressing all Critical and High findings before
mainnet deployment.

### Overall Risk Assessment
**MODERATE RISK** — Critical and High findings must be resolved before launch.
Medium findings should be addressed in the next development cycle.
```

---

## 11.4 Full Audit Report Template

```
 AUDIT REPORT STRUCTURE

1. COVER PAGE
   - Protocol name and logo
   - Audit firm name and logo
   - Date range
   - Confidentiality notice

2. TABLE OF CONTENTS

3. EXECUTIVE SUMMARY ([Section 11.3 above](#113-executive-summary-writing))

4. SCOPE & METHODOLOGY
   - Contracts in scope (file paths, nSLOC, addresses)
   - Out-of-scope items
   - Assessment methodology
   - Tools used
   - Limitations and caveats

5. SYSTEM OVERVIEW
   - Architecture diagram
   - Contract descriptions
   - Key mechanisms
   - Trust model / admin privileges
   - External dependencies

6. FINDINGS
   6.1 Critical Findings
   6.2 High Findings
   6.3 Medium Findings
   6.4 Low Findings
   6.5 Informational / Gas Optimizations

7. APPENDIX
   - Slither output summary
   - Test coverage report
   - Storage layout analysis
   - Disclaimer / legal notice
```

---

## 11.5 Responsible Disclosure Process

### Timeline

```
Day 0:    Vulnerability discovered
Day 0-1:  Verify the finding (write PoC)
Day 1:    Submit through official channel (bug bounty platform or security email)
Day 1-3:  Protocol team acknowledges receipt
Day 3-14: Protocol team investigates and develops fix
Day 14-21: Fix deployed to testnet, verified
Day 21-30: Fix deployed to mainnet
Day 30-90: Public disclosure (coordinated with protocol team)
```

### Do's and Don'ts

| [YES] Do | [NO] Don't |
|-------|---------|
| Report through official channels | Exploit on mainnet (even to "prove" it works) |
| Write a clear PoC on a fork | Tweet about it before disclosure |
| Give the team reasonable time to fix | Demand immediate payment |
| Keep all communication private | Share findings with third parties |
| Use testnets / forks for testing | Front-run the fix for profit |
| Document your discovery timeline | Destroy evidence of the disclosure process |

---

## 11.6 Working with Bug Bounty Platforms

### Platform Comparison

| Platform | Bounty Range | Response Time | Mediation |
|----------|-------------|---------------|-----------|
| **Immunefi** | Up to $10M+ | 48h acknowledgment | Immunefi mediates disputes |
| **HackenProof** | Up to $1M | 72h acknowledgment | Platform mediates |
| **Code4rena** | Contest-based | During contest period | Judging committee |
| **Sherlock** | Contest-based + fixed-pay | During contest period | Lead Senior Auditor |

### Immunefi Submission Best Practices

```markdown
## Submission to Immunefi

### Title
[One-line description of the vulnerability]

### Severity
[Critical / High / Medium / Low]

### Description
[Detailed technical description]

### Impact
[Quantified impact in dollar terms]
[Number of affected users]
[Attack prerequisites]

### Proof of Concept
[Foundry test or step-by-step reproduction]

### Recommended Fix
[Specific code changes]

### Supporting Materials
- [Link to relevant contract on Etherscan]
- [Screenshot of Tenderly simulation]
- [Test output showing the exploit]
```

### Common Rejection Reasons

| Reason | How to Avoid |
|--------|-------------|
| "Out of scope" | Read the bounty scope carefully before submitting |
| "Known issue" | Check project's GitHub issues and previous audits |
| "Theoretical" | Always include a working PoC |
| "User error" | Focus on protocol-level vulnerabilities, not user mistakes |
| "Informational" | Demonstrate concrete financial impact |
| "Duplicate" | Submit quickly, be thorough in your first submission |

---

## 11.7 Legal Considerations

### Key Legal Frameworks

| Law | Jurisdiction | Relevance |
|-----|-------------|-----------|
| **CFAA** (Computer Fraud and Abuse Act) | USA | Unauthorized access to computer systems |
| **Computer Misuse Act** | UK | Similar to CFAA |
| **GDPR** | EU | If vulnerability exposes personal data |
| **Safe harbor clauses** | Varies | Some protocols explicitly protect good-faith researchers |

### Safe Harbor

Many bug bounty programs include safe harbor provisions:

```
Typical safe harbor language:
"We will not pursue legal action against security researchers who:
1. Act in good faith
2. Avoid accessing private user data
3. Report through official channels
4. Do not exploit on mainnet
5. Give reasonable time for remediation"
```

### Protecting Yourself

- Always read the bug bounty rules before testing
- Use testnets and forks exclusively
- Never exploit a vulnerability for personal profit
- Document your timeline of discovery
- Communicate exclusively through official channels
- If unsure about scope, ask the team first

---

## 11.8 White-Hat Recovery Case Studies

### Case 1: Poly Network ($611M, Aug 2021)

```
What happened: Attacker exploited cross-chain relay to steal $611M
White-hat angle: Attacker returned all funds within 2 weeks
Key takeaway: Attacker claimed it was "for fun" and to expose the vulnerability
The protocol offered $500K bug bounty and a chief security advisor position
```

### Case 2: Wormhole ($326M, Feb 2022)

```
What happened: Signature verification bypass on Solana
Response: Jump Crypto (Wormhole backer) replaced $326M from their own funds
White-hat lesson: Protocol teams sometimes absorb losses rather than negotiate
```

### Case 3: Euler Finance ($197M, Mar 2023)

```
What happened: Donation/liquidation exploit drained $197M
Recovery: Attacker returned most funds after on-chain negotiations
Process: Protocol froze remaining contracts, published on-chain messages
Timeline: ~3 weeks for recovery negotiations
```

---

## 11.9 Communication Templates

### Initial Disclosure Email

```
Subject: [CONFIDENTIAL] Security Vulnerability in [Protocol Name]

Dear [Protocol Name] Security Team,

I am a security researcher and have identified a vulnerability in your
smart contracts that could result in [brief impact description].

**Severity:** [Critical/High/Medium/Low]
**Affected Contracts:** [addresses or file names]
**Impact:** [one-line impact summary]

I have a working proof of concept tested on a mainnet fork (no mainnet
interaction occurred). I am reporting this through your official security
channel per your responsible disclosure policy.

I am happy to provide full details and PoC through your preferred secure
communication channel. Please acknowledge receipt of this report.

Regards,
[Your name]
[Your contact]
[PGP Key fingerprint, if applicable]
```

### Follow-Up (No Response After 48h)

```
Subject: Re: [CONFIDENTIAL] Security Vulnerability — Follow-Up

Dear [Protocol Name] Team,

I submitted a security vulnerability report on [date] and have not
received acknowledgment. Given the severity of this finding ([severity]),
I want to ensure it has been received by your security team.

Please confirm receipt at your earliest convenience. If this communication
channel is not monitored for security reports, please direct me to the
appropriate channel.

I will follow standard responsible disclosure timelines (90 days) absent
communication from your team.

Regards,
[Your name]
```

> **Key Takeaway:** Your reputation as a security researcher is built on the quality of your reports and the integrity of your disclosure process. A well-written report with a working PoC, clear impact analysis, and professional communication is worth 10x a vague "I think there might be a bug." Invest time in report writing — it directly translates to higher bounty payouts and repeat engagements.

---

*← [Previous: CTF & Wargames](./CTF_AND_WARGAMES.md) | [Next: Advanced Topics →](./ADVANCED_TOPICS.md)*
