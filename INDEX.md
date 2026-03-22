---
title: "Web3 Hacker & Pentester Guide — Master Index"
description: "The definitive advanced guide for blockchain security researchers, smart contract auditors, and Web3 penetration testers. Covers EVM internals, DeFi attacks, MEV, ZK security, Account Abstraction, and professional bug bounty methodology."
keywords: "web3 security, smart contract audit, blockchain pentesting, solidity vulnerability, DeFi attack, MEV, reentrancy, flash loan exploit, Immunefi bug bounty, EVM hacking, account abstraction vulnerability, zero knowledge proof security, ZK-SNARK bugs, smart contract security researcher framework, Code4rena audit strategies, Sherlock audit guide, blockchain security roadmap, web3 ethical hacking cyber security"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Web3 Hacker & Pentester Guide"
og_description: "Publication-grade reference for Web3 security — 16 modules from EVM fundamentals to advanced exploit development, DeFi attacks, and professional bug bounty strategy."
og_type: "website"
og_image: "/assets/images/og-banner.png"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/"
schema_type: "TechArticle"
difficulty: "All Levels"
tags: [web3, security, pentesting, blockchain, solidity, DeFi, MEV, bug-bounty, smart-contracts, EVM]
nav_order: 0
permalink: /
---

<p align="center">
  <img src="./assets/images/logo.png" alt="Web3 Hacker Logo" width="300" />
</p>

<h1 align="center"> Web3 Hacker & Pentester Guide</h1>
<p align="center">
  <img src="https://img.shields.io/badge/Modules-16-blueviolet?style=for-the-badge" alt="16 Modules"/>
  <img src="https://img.shields.io/badge/Level-Beginner%20to%20Expert-green?style=for-the-badge" alt="All Levels"/>
  <img src="https://img.shields.io/badge/Last%20Updated-March%202026-orange?style=for-the-badge" alt="Updated"/>
  <img src="https://img.shields.io/badge/License-MIT-blue?style=for-the-badge" alt="MIT License"/>
  <a href="https://www.instagram.com/sdxshadow/"><img src="https://img.shields.io/badge/Instagram-sdxshadow-E4405F?style=for-the-badge&logo=instagram&logoColor=white" alt="Instagram"/></a>
</p>

---

## ️ Legal & Ethical Notice

> [!CAUTION]
> This guide is intended **exclusively for authorized security research, ethical auditing, and educational purposes**.
>
> The authors assume **no liability** for misuse of any information in this guide. By reading this material you agree to use it only for lawful purposes.


---

## Guide Overview

This guide progresses from **foundational EVM internals** through **advanced exploit engineering**, covering every layer of the Web3 attack surface. Each module is self-contained but builds upon concepts introduced earlier — follow the roadmap for structured learning, or jump directly to any module as a reference.

| # | Module | Focus Area | Difficulty |
|---|--------|------------|------------|
| 00 | **[Index (this file)](./INDEX.md)** | Navigation, prerequisites, roadmap | — |
| 01 | [Blockchain Fundamentals](./modules/BLOCKCHAIN_FUNDAMENTALS.md) | EVM deep dive, opcodes, storage, L2s, bridges |  Beginner |
| 02 | [Recon & OSINT](./modules/RECON_AND_OSINT.md) | On-chain intelligence, graph analysis, forensics |  Beginner |
| 03 | [Smart Contract Vulnerabilities](./modules/SMART_CONTRACT_VULNERABILITIES.md) | All vuln classes with PoCs, detection, and real-world refs |  Intermediate |
| 04 | [Audit Methodology](./modules/AUDIT_METHODOLOGY.md) | Professional audit workflow, static+dynamic+formal |  Intermediate |
| 05 | [Tools & Frameworks](./modules/TOOLS_AND_FRAMEWORKS.md) | Security tooling deep-dive, configuration, chaining |  Intermediate |
| 06 | [Exploit Development](./modules/EXPLOIT_DEVELOPMENT.md) | Professional PoC architecture, flash loan templates |  Advanced |
| 07 | [DeFi Protocol Attacks](./modules/DEFI_PROTOCOL_ATTACKS.md) | AMM, lending, perps, LSTs, cross-chain attack vectors |  Advanced |
| 08 | [Web3 dApp Pentesting](./modules/WEB3_DAPP_PENTESTING.md) | Frontend, wallet, RPC, EIP-712 phishing, WalletConnect |  Intermediate |
| 09 | [MEV & Mempool](./modules/MEV_AND_MEMPOOL.md) | Sandwich, backrun, intents, ERC-4337 MEV, Flashbots V3 |  Advanced |
| 10 | [CTF & Wargames](./modules/CTF_AND_WARGAMES.md) | Ethernaut, Damn Vulnerable DeFi, Paradigm CTF |  Beginner |
| 11 | [Reporting & Disclosure](./modules/REPORTING_AND_RESPONSIBLE_DISCLOSURE.md) | Reports, CVSS-B, Immunefi negotiation, disclosure timelines |  Intermediate |
| 12 | [Advanced Topics](./modules/ADVANCED_TOPICS.md) | ZK security, ERC-4337 AA, EIP-7702, formal verification |  Expert |
| 13 | [Missing Vuln Classes](./modules/MISSING_VULN_CLASSES.md) | Transient storage, multicall, compiler bugs, weird ERC-20s |  Advanced |
| 14 | [Bug Bounty Playbook](./modules/BUG_BOUNTY_PLAYBOOK.md) | Target selection, sprint methodology, income strategy |  Advanced |
| 15 | [Exploit Recreations](./modules/EXPLOIT_RECREATIONS.md) | Step-by-step PoC labs for real historical exploits |  Advanced |
| 16 | [Master Audit Checklist](./modules/AUDIT_CHECKLIST_MASTER.md) | Per-function, per-protocol, economic security checklists | All Levels |

---

## Prerequisites

### Required Knowledge

| Domain | What You Need |
|--------|--------------|
| **Solidity** | Read and write smart contracts — intermediate proficiency minimum |
| **EVM** | Understanding of opcodes, gas, storage layout |
| **JS/TypeScript** | Scripting, Hardhat/Foundry test suites, dApp interaction |
| **Linux CLI** | Terminal fluency, shell scripting, package managers |
| **Networking** | HTTP, WebSocket, JSON-RPC protocol understanding |
| **Cryptography** | Hashing, ECDSA, Merkle trees, secp256k1, BLS basics |
| **Git** | Version control, reading commit histories and diffs |

### Recommended Background

- Experience with at least one traditional pentesting methodology (OWASP, PTES, OSSTMM)
- Basic understanding of financial instruments: AMMs, lending protocols, derivatives, staking
- Prior exposure to one competitive audit platform (Code4rena, Sherlock, Cantina)
- Familiarity with at least one block explorer (Etherscan, Arbiscan)

### Hardware & Software Requirements

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| **RAM** | 8 GB | 32 GB | Archive node + fuzzer = memory-intensive |
| **Storage** | 100 GB SSD | 4 TB NVMe | Full archive node + fork snapshots |
| **CPU** | 4 cores | 16+ cores | Medusa/Echidna fuzzing is CPU-bound |
| **OS** | Ubuntu 22.04 / macOS 13+ | Ubuntu 22.04 LTS | Best tool compatibility |
| **GPU** | — | NVIDIA (for ZK proving) | Optional, aids ZK circuit work |

---

## 🧪 Lab Setup — Complete Environment

### Step 1: Core Toolchain

```bash
# ── Foundry (primary framework) ──────────────────────────────
curl -L https://foundry.paradigm.xyz | bash && foundryup
forge --version   # Forge 0.2.0+
cast --version
anvil --version

# ── Node.js 20 LTS ────────────────────────────────────────────
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version    # v20+

# ── Hardhat ───────────────────────────────────────────────────
npm install -g hardhat

# ── Python 3.11 ───────────────────────────────────────────────
sudo apt-get install python3.11 python3.11-venv python3-pip

# ── Rust (for Rust-based tools) ───────────────────────────────
curl https://sh.rustup.rs -sSf | sh
rustup update stable
```

### Step 2: Static Analysis

```bash
# Slither — 80+ vulnerability detectors (Python)
pip3 install slither-analyzer crytic-compile

# Aderyn — Fast AST-based analysis (Rust)
cargo install aderyn

# Wake — Advanced vulnerability detection framework
pip3 install eth-wake

# Solhint — Linter
npm install -g solhint

# Semgrep + web3 ruleset
pip3 install semgrep
semgrep --config=p/smart-contracts .
```

### Step 3: Dynamic Analysis & Fuzzing

```bash
# Echidna — Property-based fuzzer (Haskell, industry standard)
wget https://github.com/crytic/echidna/releases/latest/download/echidna-linux.tar.gz
tar -xzf echidna-linux.tar.gz && sudo mv echidna /usr/local/bin/

# Medusa — Parallelized Echidna alternative (much faster)
go install github.com/crytic/medusa@latest

# Halmos — Symbolic bounded model checker (Foundry-compatible)
pip3 install halmos

# Kontrol — K-framework formal verification
pip3 install kontrol

# Mythril — Symbolic execution for transaction-level analysis
pip3 install mythril
```

### Step 4: Bytecode & Reverse Engineering

```bash
# Heimdall-rs — Advanced decompilation, disassembly, CFG
cargo install heimdall-rs

# Pyrex/Panoramix — Python decompiler
pip3 install panoramix-decompiler

# EVM diff — Compare two contracts bytecode-level
npm install -g evm-bytecode-utils
```

### Step 5: Local Blockchain with Fork Testing

```bash
# Anvil fork — snapshot for reproducible testing
anvil \
  --fork-url https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY \
  --fork-block-number 21900000 \
  --chain-id 31337 \
  --accounts 10 \
  --balance 100000 \
  --port 8545

# Ganache alternative
npm install -g ganache

# Tenderly CLI — live simulation
npm install -g @tenderly/cli
tenderly login
```

### Step 6: RPC Providers

| Provider | Free Tier | Best For | Signup |
|----------|-----------|----------|--------|
| **Alchemy** | 300M CU/month | Mainnet forks, large datasets | [alchemy.com](https://alchemy.com) |
| **Infura** | 100K req/day | Multi-chain support | [infura.io](https://infura.io) |
| **QuickNode** | 10M credits/month | Low-latency MEV work | [quicknode.com](https://quicknode.com) |
| **Tenderly** | Unlimited simulations | Debug traces, simulation | [tenderly.co](https://tenderly.co) |
| **Chainstack** | 25M credits/month | Archive node access | [chainstack.com](https://chainstack.com) |
| **dRPC** | Generous free tier | Multi-chain, fast response | [drpc.org](https://drpc.org) |

### Step 7: Browser, Wallet & Proxy Tools

```bash
# Browser wallet (install as browser extension)
# - MetaMask: https://metamask.io
# - Rabby: https://rabby.io (shows tx simulation)
# - Frame: https://frame.sh (desktop wallet, advanced tx crafting)

# Burp Suite (Web3 dApp pentesting)
# Community Edition: https://portswigger.net/burp/communitydownload

# mitmproxy (lightweight HTTP interception)
pip3 install mitmproxy

# Wireshark for RPC traffic analysis
sudo apt-get install wireshark

# Postman / Insomnia (API/RPC testing)
snap install postman
```

---

## ️ Skill Progression Roadmap

### Phase 1: Foundation (Weeks 1–4)
> **Goal: Understand how blockchain works at the protocol level**

1. Complete **[Module 01](./modules/BLOCKCHAIN_FUNDAMENTALS.md)** — EVM architecture, opcodes, storage, L2 security models
2. Complete **[Module 02](./modules/RECON_AND_OSINT.md)** — On-chain OSINT, target profiling, blockchain forensics
3. Complete **[Module 10](./modules/CTF_AND_WARGAMES.md)** — Begin Ethernaut challenges (levels 0–15)
4. Set up full lab environment (Foundry + Slither + Anvil)
5. Read 5 past audit reports from Trail of Bits, OpenZeppelin, or Spearbit
6. Deploy and interact with contracts on Sepolia/Holešky testnet

### Phase 2: Core Security Skills (Weeks 5–10)
> **Goal: Identify and exploit all standard vulnerability classes**

1. Complete **[Module 03](./modules/SMART_CONTRACT_VULNERABILITIES.md)** — Master every vulnerability class with PoC reproduction
2. Complete **[Module 04](./modules/AUDIT_METHODOLOGY.md)** — Implement professional audit workflow end-to-end
3. Complete **[Module 05](./modules/TOOLS_AND_FRAMEWORKS.md)** — Configure and chain: Slither → Echidna → Foundry Fuzz → Halmos
4. Finish Ethernaut (all levels), begin Damn Vulnerable DeFi (DVDF v3)
5. Write your first 5 PoC exploits in Foundry (including flash loan)
6. Participate in a Secureum RACE quiz

### Phase 3: Advanced Practice (Weeks 11–18)
> **Goal: Exploit complex DeFi protocols and develop professional-grade PoCs**

1. Complete **[Module 06](./modules/EXPLOIT_DEVELOPMENT.md)** — Exploit Development (flash loan templates, PoC architecture)
2. Complete **[Module 07](./modules/DEFI_PROTOCOL_ATTACKS.md)** — DeFi Protocol Attacks (AMMs, lending, perps, LSTs)
3. Complete **[Module 08](./modules/WEB3_DAPP_PENTESTING.md)** — dApp Pentesting (frontend, RPC, wallet, EIP-712)
4. Complete **[Module 09](./modules/MEV_AND_MEMPOOL.md)** — MEV & Mempool (sandwich, arbitrage, intent-based MEV)
5. Complete **[Module 13](./modules/MISSING_VULN_CLASSES.md)** — Missing Vuln Classes (transient storage, multicall, compiler bugs)
6. Complete Damn Vulnerable DeFi V3, begin Paradigm CTF challenges
7. Participate in your first competitive audit (Code4rena, Sherlock, or Cantina)

### Phase 4: Expert (Weeks 19+)
> **Goal: Become a professional security researcher or senior auditor**

1. Complete **[Module 11](./modules/REPORTING_AND_RESPONSIBLE_DISCLOSURE.md)** — Professional Reporting, CVSS-B scoring, Immunefi submissions
2. Complete **[Module 12](./modules/ADVANCED_TOPICS.md)** — Advanced Topics (ZK security, ERC-4337, EIP-7702)
3. Complete **[Module 14](./modules/BUG_BOUNTY_PLAYBOOK.md)** — Bug Bounty Playbook (income strategy, target selection)
4. Complete **[Module 15](./modules/EXPLOIT_RECREATIONS.md)** — Recreate 5+ historical exploits from scratch
5. Use **[Module 16](./modules/AUDIT_CHECKLIST_MASTER.md)** — Master Audit Checklist on every engagement
6. Submit your first Immunefi bug bounty finding
7. Pursue solo audits or join an audit firm (Spearbit, Trust, Pashov, Cyfrin)
8. Build custom security tools (Slither detectors, Echidna properties, Halmos specs)
9. Publish original security research (blogs, conference talks, whitepapers)
10. Mentor other security researchers and contribute to community education

---

## Master Tool Reference

### Static Analysis & Linting

| Tool | Language | Detectors | Best For | Install |
|------|----------|-----------|----------|---------|
| **Slither** | Python | 80+ | Comprehensive automated analysis | `pip3 install slither-analyzer` |
| **Aderyn** | Rust | 50+ | Fast CI/CD integration | `cargo install aderyn` |
| **Wake** | Python | 40+ | Framework with Python scripting | `pip3 install eth-wake` |
| **Solhint** | JS | Style+Security | Code quality gates | `npm install -g solhint` |
| **Semgrep** | Python | Custom rules | Custom pattern matching | `pip3 install semgrep` |
| **4naly3er** | Python | Template-based | Audit report generation | Clone from GitHub |

### Dynamic Analysis & Fuzzing

| Tool | Language | Mode | Best For | Speed vs Echidna |
|------|----------|------|----------|-----------------|
| **Foundry Fuzz** | Solidity | Stateless | Quick property checks | Ships with Forge |
| **Echidna** | Haskell | Stateful | Deep property-based testing | Baseline |
| **Medusa** | Go | Stateful | Parallelized, 3-5× faster | 3-5× |
| **Halmos** | Python | Symbolic | Formal SMT-based verification | N/A |
| **Manticore** | Python | Symbolic | Full bytecode symbolic exec | N/A |
| **Kontrol** | K-lang | Formal | Full formal verification | N/A |

### Bytecode Analysis

| Tool | Type | Capabilities | Use Case |
|------|------|-------------|----------|
| **Heimdall-rs** | CLI | Decompile, disassemble, CFG | Unverified contracts |
| **Dedaub** | Web | High-quality decompilation | Complex contracts |
| **EtherVM** | Web | Quick online decompilation | Fast checks |
| **Panoramix** | Python | Python-output decompilation | Automated analysis |
| **Tenderly Debugger** | Web | Step-through opcode trace | Tx forensics |

### On-Chain Intelligence

| Tool | Type | Best For |
|------|------|---------|
| **Etherscan / Arbiscan** | Web | Contract source, tx explorer |
| **Phalcon** (BlockSec) | Web | Attack tx replay and analysis |
| **Tenderly Simulator** | Web | Pre-execution state simulation |
| **Dune Analytics** | Web | Custom on-chain SQL queries |
| **Arkham Intelligence** | Web | Entity labeling, fund tracing |
| **Nansen** | Web | Wallet behavior, DEX analytics |
| **Breadcrumbs** | Web | Fund flow visualization |
| **4byte.directory** | Web | Function selector lookup |
| **OpenChain** | Web | ABI decoding of any selector |

### Scripting & Chain Interaction

| Tool | Language | Best For |
|------|----------|---------|
| **Cast** (Foundry) | CLI | Chain queries, tx crafting, ABI encoding |
| **ethers.js v6** | TypeScript | Modern dApp scripting |
| **viem** | TypeScript | High-performance, type-safe EVM lib |
| **web3.py** | Python | Python automation, research scripts |
| **Brownie** | Python | Legacy Python Hardhat equivalent |

---

## Essential Reading & Resources

### Canonical References

| Resource | Type | Focus |
|----------|------|-------|
| [SWC Registry](https://swcregistry.io/) | Reference | Smart Contract Weakness Classification |
| [Rekt News](https://rekt.news/) | Post-mortems | DeFi exploit analysis |
| [DeFiHackLabs](https://github.com/SunWeb3Sec/DeFiHackLabs) | PoC Library | 200+ real exploit reproductions in Foundry |
| [Solodit](https://solodit.xyz/) | Database | Audit finding aggregator (all platforms) |
| [Immunefi Classified](https://immunefi.com/bug-bounty/) | Platform | Live bug bounty programs |
| [Code4rena Findings](https://code4rena.com/reports) | Reports | Competitive audit outcomes |
| [Sherlock Findings](https://app.sherlock.xyz/audits) | Reports | Audit reports with severity reasoning |
| [Trail of Bits Blog](https://blog.trailofbits.com/) | Research | Cutting-edge tool and vuln research |
| [Spearbit Blog](https://spearbit.com/) | Research | Protocol-specific deep dives |

### Essential Books

- *Mastering Ethereum* — Andreas Antonopoulos & Gavin Wood (free online)
- *Smart Contract Security Field Guide* — Trail of Bits
- *Ethereum EVM Illustrated* — Takenobu T. (visual EVM reference)

### Courses & Educational Content

| Resource | Platform | Level |
|----------|----------|-------|
| **Cyfrin Updraft Web3 Security** | cyfrin.io | Beginner→Advanced |
| **Secureum Bootcamp** | secureum.xyz | Intermediate→Advanced |
| **Owen Thurm's Smart Contract Hacking** | YouTube | Intermediate |
| **Andy Li's Audit Techniques** | YouTube | Intermediate |
| **Patrick Collins — Foundry Course** | YouTube | Beginner |
| **Rareskills Blog** | rareskills.io | All levels — deep technical |

### Communities

| Community | Platform | Focus |
|-----------|----------|-------|
| **Secureum Discord** | Discord | Audit discussions, RACE quizzes |
| **Immunefi Discord** | Discord | Bug bounty hunters community |
| **DeFi Security Summit** | Annual | On-site conference |
| **EthSecurity Telegram** | Telegram | Researcher network |
| **Spearbit Discord** | Discord | Advanced researcher community |
| **Code4rena Discord** | Discord | Competitive audit community |

---

## Gap Analysis — Advanced Topics Added

The original modules were strong but these critical areas were added for senior-level work:

| Advanced Topic | Module |
|---------------|--------|
| Transient storage (EIP-1153) attack vectors | 13 §13.1 |
| ERC-721/1155 callback reentrancy patterns | 13 §13.2 |
| Permit/EIP-2612 front-running & phishing | 13 §13.3 |
| Multicall `msg.value` reuse (Uniswap V3 pattern) | 13 §13.4 |
| Read-only reentrancy (deep dive) | 13 §13.5 |
| Complete weird ERC-20 reference (16 types) | 13 §13.6 |
| Solidity/Vyper compiler bugs (live CVEs) | 13 §13.7 |
| Return bomb & gas stipend attacks | 13 §13.8 |
| Chainlink full validation (minAnswer, L2 sequencer, deviation) | 13 §13.9 |
| TWAP manipulation cost analysis | 13 §13.10 |
| ERC-4626 vault rounding direction errors | 13 §13.11 |
| Governance timelock bypass patterns | 13 §13.12 |
| Cross-chain replay attacks (chainId spoofing) | 13 §13.13 |
| Inline assembly pitfalls & MSTORE8 clobbering | 13 §13.14 |
| Staking reward distribution bugs | 13 §13.15 |
| Donation-based price manipulation (ERC-4626) | 13 §13.16 |
| ECDSA signature malleability (OpenSSL bypass) | 13 §13.17 |
| Liquidation mechanism vulnerabilities | 13 §13.18 |
| Flashbots bundle ordering attacks | 13 §13.19 |
| Proxy upgrade attack patterns (UUPS, Beacon) | 13 §13.20 |
| Target selection & sprint methodology | 14 |
| Immunefi submission strategy & negotiation | 14 |
| Income trajectory & reputation building | 14 |
| Real exploit recreations (DAO, Harvest, Beanstalk, Nomad, Euler) | 15 |
| Per-function audit checklist | 16 |
| Per-protocol checklists (AMM, lending, perps, LST, AA) | 16 |
| Economic security checklist | 16 |
| ZK circuit security & constraint bugs | 12 |
| ERC-4337 Account Abstraction attack surface | 12 |
| EIP-7702 (EOA delegation) security implications | 12 |
| Intent-based MEV & solver manipulation | 09 |
| WalletConnect v2 security model | 08 |
| EIP-712 phishing attack anatomy | 08 |

---

> [!NOTE]
> **This is a living document.** Web3 security evolves rapidly — new vulnerability classes emerge as new primitives (Account Abstraction, intents, ZK rollups, restaking) ship to production. Revisit modules periodically and supplement with real-time exploit analysis from [Rekt News](https://rekt.news/) and [BlockSec alerts](https://blocksec.com/phalcon/alert).

---

<div align="center">

**[Start with Module 01 — Blockchain Fundamentals →](./modules/BLOCKCHAIN_FUNDAMENTALS.md)**

*Built for serious researchers. Use responsibly.*

</div>

---

*Next: [01 — Blockchain Fundamentals →](./modules/BLOCKCHAIN_FUNDAMENTALS.md)*
