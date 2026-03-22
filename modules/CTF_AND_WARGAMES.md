---
title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)0 — Web3 CTF & Security Wargames Guide"
description: "Complete guide to Web3 security CTF challenges and wargames: Ethernaut solving methodology, Damn Vulnerable DeFi challenge approaches, Paradigm CTF strategies, competitive audit platform comparison (Code4rena, Sherlock, CodeHawks, Immunefi), additional practice platforms, speed auditing techniques, and CTF pattern recognition cheat sheet."
keywords: "Ethernaut CTF, Damn Vulnerable DeFi, Paradigm CTF, web3 wargames, Code4rena, Sherlock audit, CodeHawks, Immunefi bug bounty, smart contract CTF, blockchain security challenges, competitive audit strategies, smart contract speed auditing, CTF pattern recognition blockchain"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)0 — Web3 CTF & Wargames | Web3 Hacker Guide"
og_description: "Solve Ethernaut, Damn Vulnerable DeFi, and Paradigm CTF with proven strategies — includes speed auditing techniques and pattern recognition for competitive security contests."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/10_CTF_AND_WARGAMES"
schema_type: "TechArticle"
difficulty: " Beginner →  Advanced"
module: 10
tags: [CTF, Ethernaut, Damn-Vulnerable-DeFi, Paradigm-CTF, Code4rena, Sherlock, Immunefi, wargames]
nav_order: 10
parent: "Web3 Hacker & Pentester Guide"
---

# Module 10 — CTF & Wargames

> **Difficulty:** Beginner →  Advanced
>
> Hands-on practice is the fastest way to internalize Web3 security concepts. This module covers every major CTF platform, competitive audit arena, and practice resource with solving strategies and pattern recognition techniques.

---

## 10.1 Ethernaut (OpenZeppelin)

**URL:** [ethernaut.openzeppelin.com](https://ethernaut.openzeppelin.com/)

Ethernaut is the definitive starter CTF for blockchain security. 30+ levels covering fundamental vulnerability classes.

### Level Categories & Solving Approach

| Category | Levels | Key Skills |
|----------|--------|-----------|
| **Basic Ethereum** | 0-3 (Hello, Fallback, Fallout, Coin Flip) | Wallet interaction, fallback functions, randomness |
| **Access Control** | 4-6 (Telephone, Token, Delegation) | `tx.origin` vs `msg.sender`, overflow, delegatecall |
| **Storage & Basics** | 7-10 (Force, Vault, King, Re-entrancy) | Force ETH, private storage, DoS, reentrancy |
| **Advanced** | 11-20 (Elevator, Privacy, GatekeeperOne/Two, ...) | Interface tricks, storage layout, gas manipulation |
| **Expert** | 21+ (Shop, Dex, Puzzle Wallet, Motorbike, ...) | View manipulation, DEX math, proxy attacks |

### Solving Methodology

```
For each Ethernaut level:
1. READ the contract source code completely
2. IDENTIFY what the "win condition" requires
3. MAP the vulnerability class (check [Module 03](./SMART_CONTRACT_VULNERABILITIES.md))
4. PLAN the exploit (which cheatcode/pattern to use)
5. WRITE interaction code (Foundry, cast, or browser console)
6. EXECUTE on the testnet instance
7. VERIFY with the Submit Instance button

Example — Level 4 (Telephone):
- Win condition: Claim ownership
- Vulnerability: tx.origin vs msg.sender confusion
- Solution: Call through an intermediary contract
  → msg.sender = intermediary, tx.origin = you
```

### Common Ethernaut Patterns

| Pattern | Levels It Appears |
|---------|-------------------|
| `tx.origin` authentication bypass | Telephone |
| Integer overflow (pre-0.8.x) | Token |
| `delegatecall` context preservation | Delegation, Puzzle Wallet |
| Private storage is publicly readable | Vault, Privacy |
| Force-send ETH via selfdestruct | Force, King |
| EVM gas mechanics | GatekeeperOne |
| Proxy storage collision | Puzzle Wallet, Motorbike |

---

## 10.2 Damn Vulnerable DeFi (tinchoabbate)

**URL:** [damnvulnerabledefi.xyz](https://www.damnvulnerabledefi.xyz/)

DvD is the gold standard for DeFi-specific challenges. More complex than Ethernaut, focused on DeFi primitives.

### Challenge Categories

| Category | Challenges | Concepts |
|----------|-----------|----------|
| **Flash Loans** | Unstoppable, Naive Receiver, Truster, Side Entrance | Flash loan mechanics, callback exploitation |
| **Token/NFT** | The Rewarder, Selfie, Compromised | Token manipulation, reward distribution |
| **Lending** | Puppet, Puppet V2, Free Rider | Oracle manipulation, flash loan price manipulation |
| **Governance** | Selfie, Backdoor | Flash loan governance, GnosisSafe abuse |
| **Bridges & Advanced** | Climber, Wallet Mining, ABI Smuggling | Timelock bypass, CREATE2, ABI edge cases |

### Approach for Each Challenge

```
1. Read the README and understand the goal
2. Read ALL contracts in the challenge, including tests
3. Understand the "success condition" in the test file
4. Identify the invariant that you need to break
5. Write the exploit in the test framework provided
6. Run: forge test --mt test_challengeName -vvvv
```

### Challenge Hints (Spoiler-Free Approaches)

| Challenge | Approach Hint |
|-----------|---------------|
| **Unstoppable** | How can you break the flash loan invariant without using the pool? |
| **Naive Receiver** | Who pays for the flash loan fee? |
| **Truster** | What can you do during the flash loan callback? |
| **Side Entrance** | Can you deposit into the pool during a flash loan? |
| **The Rewarder** | When are reward snapshots taken? |
| **Selfie** | Can you acquire governance power temporarily? |
| **Puppet** | What price source does the pool use? |
| **Free Rider** | Can you buy NFTs with the marketplace's own funds? |
| **Climber** | What order does the timelock check things? |

---

## 10.3 Paradigm CTF

**URL:** Historical challenges available on GitHub

Paradigm CTFs feature cutting-edge challenges that go beyond typical vulnerability patterns.

### Notable Challenge Types

| Category | Description | Skills Required |
|----------|-------------|----------------|
| **EVM puzzles** | Exploit opcodes directly | Bytecode, assembly |
| **DeFi challenges** | Complex protocol attacks | Flash loans, AMM math |
| **Cross-chain** | Multi-chain exploitation | Bridge understanding |
| **ZK challenges** | Circuit vulnerabilities | Zero-knowledge basics |
| **Meta-game** | Challenge infrastructure exploitation | Out-of-the-box thinking |

### Study Resources

```
Paradigm CTF Solutions:
- https://github.com/paradigmxyz/paradigm-ctf-2023
- Community writeups on Mirror, Medium, and personal blogs
- cmichel's writeups: cmichel.io
- samczsun's blog: samczsun.com
```

---

## 10.4 Competitive Audit Platforms

### Code4rena

**URL:** [code4rena.com](https://code4rena.com/)

| Aspect | Details |
|--------|---------|
| **Format** | Time-limited competitive audits (3–21 days) |
| **Payout** | Shared prize pool, split among valid unique findings |
| **Severity** | High, Medium (paid), QA/Gas (small reward) |
| **Good for** | Building track record, learning from judging |

**Approaching a Code4rena Contest:**
```
Day 1: Full codebase scan, Slither run, understand architecture
Day 2-3: Deep manual review of core contracts
Day 4-5: Focus on economic attack vectors and edge cases
Day 6-7: Write findings and PoCs
```

### Sherlock

**URL:** [sherlock.xyz](https://sherlock.xyz/)

| Aspect | Details |
|--------|---------|
| **Format** | Time-limited with lead senior auditor (LSA) |
| **Payout** | Based on finding uniqueness and severity |
| **Judging** | LSA judges + escalation process |
| **Good for** | Higher quality judging, clearer rules |

### CodeHawks (Cyfrin)

**URL:** [codehawks.com](https://codehawks.cyfrin.io/)

| Aspect | Details |
|--------|---------|
| **Format** | Competitive audits + First Flights (beginner-friendly) |
| **Payout** | Prize pool per contest |
| **Good for** | Beginners (First Flights), structured learning path |

### Immunefi Bug Bounties

**URL:** [immunefi.com](https://immunefi.com/)

| Aspect | Details |
|--------|---------|
| **Format** | Ongoing bug bounties on live protocols |
| **Payout** | Up to $10M+ for critical findings |
| **Good for** | Real-world impact, highest payouts |

---

## 10.5 Additional Practice Platforms

| Platform | Focus | Difficulty | URL |
|----------|-------|-----------|-----|
| **OnlyPwner** | Smart contract CTF challenges | - | [onlypwner.xyz](https://onlypwner.xyz/) |
| **EVM Puzzles** | Opcode-level puzzles | - | [github/fvictorio/evm-puzzles](https://github.com/fvictorio/evm-puzzles) |
| **Capture the Ether** | Classic Ethereum CTF |  | [capturetheether.com](https://capturetheether.com/) |
| **DeFi Hack Labs** | Reproduce real exploits |  | [github/SunWeb3Sec/DeFiHackLabs](https://github.com/SunWeb3Sec/DeFiHackLabs) |
| **Mr Steal Yo Crypto** | DeFi attack challenges | - | [mrstealyocrypto.xyz](https://mrstealyocrypto.xyz/) |
| **Secureum RACE** | Quiz-style solidity security |  | [secureum.substack.com](https://secureum.substack.com/) |

---

## 10.6 Building Your Own Vulnerable Lab

### Quick Foundry-Based Lab

```bash
# Create a lab project
forge init vuln-lab && cd vuln-lab

# Create vulnerable contracts
mkdir -p src/challenges

# Run your lab against test exploits
forge test -vvvv
```

```solidity
// src/challenges/ReentrancyChallenge.sol
contract ReentrancyChallenge {
    mapping(address => uint256) public balances;
    bool public solved;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // Intentionally vulnerable
    function withdraw() external {
        uint256 balance = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success);
        balances[msg.sender] = 0;
    }

    function isSolved() external view returns (bool) {
        return address(this).balance == 0 && solved;
    }
}
```

---

## 10.7 CTF Pattern Recognition Cheat Sheet

| Pattern You See | Vulnerability | Solution Direction |
|----------------|---------------|-------------------|
| External call before state update | Reentrancy | Write attack contract with receive() callback |
| `tx.origin` in require | Access control bypass | Call through intermediary contract |
| `block.timestamp` / `block.number` in logic | Timestamp/block manipulation | As validator or use `vm.warp`/`vm.roll` |
| `delegatecall` to user input | Code execution hijack | Point to your malicious contract |
| Private state variable | False sense of privacy | Read storage directly via `getStorageAt` |
| Solidity < 0.8.0 + arithmetic | Integer overflow | Trigger wrap-around |
| `transfer` / `send` for ETH | Gas limitation + DoS | Contract without receive() to DoS |
| No slippage check on swap | Price manipulation | Flash loan + dump to manipulate price |
| Upgradeable without initializer guard | Uninitialized impl | Call initialize() on implementation directly |
| ERC20 with callbacks (ERC-777) | Reentrancy via token hooks | Transfer tokens to trigger hook re-entry |

---

## 10.8 Speed Auditing Techniques for Competitions

### Time-Optimized Workflow

```
Minutes 0-30: RAPID SCAN
├── Read README/docs (understand the protocol intent)
├── List all external/public functions
├── Run Slither (background)
├── Run Aderyn (background)
└── Note down contract sizes and inheritance

Minutes 30-120: DEEP DIVE ON CRITICAL CONTRACTS
├── Follow the money (deposit → internal logic → withdrawal)
├── Check all access control
├── Look for external calls (reentrancy surface)
├── Check oracle interactions
└── Review token handling (fee-on-transfer, rebasing)

Minutes 120-180: ATTACK BRAINSTORMING
├── Can I manipulate an oracle with a flash loan?
├── Can I reenter during any callback?
├── Can I bypass access control?
├── Are there rounding errors in share calculations?
├── What happens at extreme values (0, type(uint256).max)?
└── What if two functions are called in the same transaction?

Minutes 180+: POC WRITING
├── Write Foundry test for each finding
├── Classify severity
└── Submit
```

### Quick Wins Checklist (Things to Check First)

- [ ] Missing access control on state-changing functions
- [ ] Unchecked return values on `call`, `transfer`, `transferFrom`
- [ ] `approve` followed by `transferFrom` without balance check
- [ ] Oracle price without staleness validation
- [ ] Division before multiplication (precision loss)
- [ ] Missing `initializer` modifier on upgradeable contracts
- [ ] `delegatecall` to user-controlled address
- [ ] `selfdestruct` accessible
- [ ] Missing `nonReentrant` on functions with external calls + state updates

> **Key Takeaway:** The best way to get fast at finding bugs is volume. Solve every Ethernaut level, complete Damn Vulnerable DeFi, then jump into competitive audits. After your first 5–10 contests, you'll develop pattern recognition that makes you exponentially faster. The difference between finding 1 bug and finding 10 bugs in a contest is not 10x skill — it's having seen the patterns before.

---

*← [Previous: MEV & Mempool](./MEV_AND_MEMPOOL.md) | [Next: Reporting & Disclosure →](./REPORTING_AND_RESPONSIBLE_DISCLOSURE.md)*


<script>
document.addEventListener("DOMContentLoaded", function() {
  const checkboxes = document.querySelectorAll('.task-list-item-checkbox, input[type="checkbox"]');
  checkboxes.forEach(function(cb) {
    cb.removeAttribute('disabled');
    cb.style.cursor = 'pointer';
  });
});
</script>
