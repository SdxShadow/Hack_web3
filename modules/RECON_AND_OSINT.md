---
title: "Module 02 — Reconnaissance & OSINT for Web3 Security"
description: "Complete on-chain OSINT methodology for Web3 penetration testers: Etherscan deep dives, proxy detection, admin tracing, deployer analysis, dependency mapping, Dune Analytics queries, and blockchain forensics techniques."
keywords: "web3 OSINT, blockchain reconnaissance, on-chain intelligence, proxy detection, EIP-1967, Etherscan API, Dune Analytics, smart contract recon, DeFi security research, blockchain forensics, token distribution analysis, dependency mapping blockchain, admin trace OSINT, subgraph security analysis, transaction pattern recognition"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 02 — Web3 Recon & OSINT | Web3 Hacker Guide"
og_description: "Systematic on-chain intelligence gathering for Web3 security researchers — from Etherscan profiling to blockchain graph analysis and admin key tracing."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/02_RECON_AND_OSINT"
schema_type: "TechArticle"
difficulty: " Beginner →  Intermediate"
module: 2
tags: [OSINT, recon, blockchain, Etherscan, proxy, DeFi, forensics, web3-security]
nav_order: 2
parent: "Web3 Hacker & Pentester Guide"
---

# Module 02 — Reconnaissance & OSINT for Web3

> **Difficulty:** Beginner →  Intermediate
>
> Reconnaissance is the first phase of any penetration test. In Web3, the transparent nature of public blockchains gives us extraordinary visibility — every transaction, every contract interaction, every token transfer is permanently recorded. This module teaches you how to systematically extract intelligence from on-chain and off-chain sources before touching a single line of Solidity.

---

## 2.1 On-Chain OSINT

### Etherscan Deep Dive

Etherscan is far more than a block explorer. For a pentester, it's the primary intelligence platform.

**What to Extract from Etherscan:**

| Data Point | Where to Find | Security Relevance |
|-----------|---------------|-------------------|
| Contract source code | "Contract" tab → "Code" | Review verified source; check compiler version, optimizer settings |
| Read/Write functions | "Contract" tab → "Read/Write" | Identify admin functions, state queries |
| Transaction history | "Transactions" tab | Deployment sequence, admin calls, suspicious patterns |
| Internal transactions | "Internal Txns" tab | `DELEGATECALL`, `CREATE`, ETH movements |
| Token transfers | "Token Transfers" tab | Token flow analysis, whale movements |
| Events/logs | "Events" tab | Ownership changes, parameter updates, pauses |
| Contract creator | Top of contract page | Trace back to deployer wallet, find related contracts |
| Similar contracts | "Contract Diff" feature | Compare against known-vulnerable versions |

```bash
# Using Etherscan API for automated recon
export ETHERSCAN_API_KEY="YOUR_KEY"

# Get contract source code
curl "https://api.etherscan.io/api?module=contract&action=getsourcecode&address=0xCONTRACT&apikey=$ETHERSCAN_API_KEY"

# Get all transactions for an address
curl "https://api.etherscan.io/api?module=account&action=txlist&address=0xADDRESS&startblock=0&endblock=99999999&sort=asc&apikey=$ETHERSCAN_API_KEY"

# Get contract ABI
curl "https://api.etherscan.io/api?module=contract&action=getabi&address=0xCONTRACT&apikey=$ETHERSCAN_API_KEY"
```

### Tenderly — Transaction Debugging

Tenderly provides **step-by-step transaction execution traces**, including state changes, gas usage per opcode, and call stack visualization.

**Key features for recon:**
- **Transaction Simulator** — Simulate transactions against live or forked state
- **State Diff** — See exactly which storage slots changed and their before/after values
- **Call Trace** — Visual call graph showing all internal calls, delegatecalls, and external calls
- **Gas Profiler** — Identify gas-heavy operations (potential DoS vectors)

```bash
# Tenderly CLI — export project for simulation
tenderly export --project YOUR_PROJECT 0xTransactionHash

# Fork a network state at a specific block
tenderly fork --network mainnet --block-number 18500000
```

### Dune Analytics — Custom On-Chain Queries

Dune lets you write SQL queries against decoded blockchain data. For auditors, this is invaluable for:

```sql
-- Find all admin function calls on a contract
SELECT
    block_time,
    tx_hash,
    "from" AS caller,
    function_name
FROM ethereum.decoded_contracts
WHERE contract_address = 0xTARGET
  AND function_name IN ('setOwner', 'transferOwnership', 'pause', 'upgradeTo')
ORDER BY block_time DESC
LIMIT 100;

-- Track token distribution concentration
SELECT
    "to" AS holder,
    SUM(value) / 1e18 AS balance
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xTOKEN_ADDRESS
GROUP BY "to"
ORDER BY balance DESC
LIMIT 50;
```

### Phalcon (BlockSec) — Attack Transaction Analysis

Phalcon specializes in **dissecting exploit transactions**. When analyzing a hack:

1. Enter the transaction hash at [phalcon.blocksec.com](https://phalcon.blocksec.com)
2. Phalcon shows the **invocation flow** — every call, with decoded function names, arguments, and return values
3. **Balance changes** — which addresses gained/lost which tokens
4. **Fund flow** — visual diagram of token movements

> **Practical Tip:** When studying past exploits for CTF preparation or research, Phalcon is faster than reading Tenderly traces because it pre-labels known contracts and protocols.

---

## 2.2 Contract Verification Analysis

### Verified vs. Unverified Contracts

| Aspect | Verified | Unverified |
|--------|----------|------------|
| Source code | Available on Etherscan | Only bytecode |
| Audit ability | Full Solidity review | Must decompile |
| Trust level | Higher (but compiler bugs exist) | Suspicious by default |
| Analysis tools | Slither, Mythril, manual review | Heimdall-rs, Dedaub, EtherVM |

### Analyzing Unverified Contracts

```bash
# Step 1: Get the bytecode
cast code 0xUnverifiedContract --rpc-url $ETH_RPC

# Step 2: Try Dedaub online decompiler
# Upload bytecode at https://app.dedaub.com/decompile

# Step 3: Use heimdall-rs for advanced decompilation
heimdall decompile 0xUnverifiedContract --rpc-url $ETH_RPC --output ./decompiled/

# Step 4: Look for known selectors
heimdall decode 0xUnverifiedContract --rpc-url $ETH_RPC

# Step 5: Check Sourcify for partial verification
curl "https://sourcify.dev/server/check-all-by-addresses?addresses=0xCONTRACT&chainIds=1"
```

### Compiler Version Analysis

```bash
# Extract compiler version from metadata
cast code 0xContract | tail -c 86
# Last bytes contain CBOR-encoded metadata including Solidity version

# Check for known compiler bugs
# https://docs.soliditylang.org/en/latest/bugs.html
# Critical: Solidity 0.8.13-0.8.15 had optimizer bugs affecting ABI encoding
```

---

## 2.3 Proxy Pattern Detection

Identifying proxy patterns is critical — the code you see on Etherscan might not be the code actually executing.

### Common Proxy Patterns

| Pattern | Detection Method | Storage Slot |
|---------|-----------------|-------------|
| **EIP-1967 Transparent Proxy** | Check slot `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc` | Implementation address |
| **EIP-1967 Admin** | Check slot `0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103` | Admin address |
| **EIP-1967 Beacon** | Check slot `0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50` | Beacon address |
| **EIP-897 (DelegateProxy)** | Call `implementation()` | Usually slot 0 |
| **Minimal Proxy (EIP-1167)** | Bytecode starts with `363d3d373d3d3d363d73` | Hardcoded in bytecode |
| **UUPS (EIP-1822)** | `upgradeTo()` in implementation, not proxy | Same slot as EIP-1967 |
| **Diamond (EIP-2535)** | Multiple facets, `diamondCut()` function | Mapping of selectors → facets |

```bash
# Check for EIP-1967 implementation slot
cast storage 0xProxyAddress 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

# Check for EIP-1967 admin slot
cast storage 0xProxyAddress 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103

# Detect minimal proxy pattern (EIP-1167)
cast code 0xAddress | grep -q "363d3d373d3d3d363d73" && echo "EIP-1167 Minimal Proxy detected"

# Get implementation from OpenZeppelin proxy
cast call 0xProxyAddress "implementation()(address)"
```

### Proxy Detection Script

```javascript
// proxy-detect.js — Automated proxy pattern detection
const { ethers } = require("ethers");
const provider = new ethers.JsonRpcProvider(process.env.ETH_RPC);

async function detectProxy(address) {
    const results = {};

    // EIP-1967 implementation slot
    const implSlot = "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc";
    const implData = await provider.getStorage(address, implSlot);
    if (implData !== ethers.ZeroHash) {
        results.type = "EIP-1967 Transparent/UUPS Proxy";
        results.implementation = ethers.getAddress("0x" + implData.slice(26));
    }

    // EIP-1967 beacon slot
    const beaconSlot = "0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50";
    const beaconData = await provider.getStorage(address, beaconSlot);
    if (beaconData !== ethers.ZeroHash) {
        results.type = "Beacon Proxy";
        results.beacon = ethers.getAddress("0x" + beaconData.slice(26));
    }

    // EIP-1167 minimal proxy detection
    const code = await provider.getCode(address);
    if (code.startsWith("0x363d3d373d3d3d363d73")) {
        results.type = "EIP-1167 Minimal Proxy (Clone)";
        results.implementation = ethers.getAddress("0x" + code.slice(22, 62));
    }

    return results;
}

detectProxy("0xTargetAddress").then(console.log);
```

---

## 2.4 Admin & Owner Address Tracing

### Identifying Privileged Roles

```bash
# Common owner/admin function selectors
cast call 0xContract "owner()(address)"
cast call 0xContract "admin()(address)"
cast call 0xContract "governance()(address)"
cast call 0xContract "guardian()(address)"
cast call 0xContract "pendingOwner()(address)"

# For AccessControl (OpenZeppelin)
# DEFAULT_ADMIN_ROLE = 0x00
cast call 0xContract "getRoleMemberCount(bytes32)(uint256)" 0x0000000000000000000000000000000000000000000000000000000000000000
cast call 0xContract "getRoleMember(bytes32,uint256)(address)" 0x0000000000000000000000000000000000000000000000000000000000000000 0

# For Timelock controllers
cast call 0xContract "getMinDelay()(uint256)"
```

### Multisig Identification

```bash
# Gnosis Safe detection — check for known Safe functions
cast call 0xAddress "getThreshold()(uint256)" 2>/dev/null && echo "Gnosis Safe detected"

# Get Safe owners
cast call 0xSafe "getOwners()(address[])"

# Get threshold
cast call 0xSafe "getThreshold()(uint256)"
```

**What to look for in admin analysis:**
- Single EOA ownership = high centralization risk
- 2-of-3 multisig = still vulnerable to 2 key compromise
- Timelock = admin changes delayed (how long? 24h vs 7 days)
- No `renounceOwnership()` called = owner can still act

---

## 2.5 Deployer Wallet & Related Contract Analysis

### Tracing Contract Relationships

```bash
# Find who deployed a contract
cast receipt 0xDeploymentTxHash --field from

# Find all contracts deployed by an address
# Use Etherscan API:
curl "https://api.etherscan.io/api?module=account&action=txlist&address=0xDeployer&startblock=0&endblock=99999999&sort=asc&apikey=$ETHERSCAN_API_KEY" \
  | jq '.result[] | select(.to == "") | {hash: .hash, contractAddress: .contractAddress}'
```

### Patterns to Look For

| Pattern | What It Indicates |
|---------|------------------|
| Same deployer, multiple contracts | Protocol ecosystem mapping |
| Deployer funds from Tornado Cash / Railgun | Privacy-seeking deployer (could be legitimate or malicious) |
| CREATE2 deployments | Deterministic addresses, possible metamorphic risk |
| Factory pattern deployments | Protocol uses factory for pool/vault creation |
| Deployer renounces then reclaims via backdoor | Potential rug pull setup |

---

## 2.6 Protocol Dependency Mapping

### What to Map

Every DeFi protocol depends on external systems. Map these dependencies to understand the full attack surface.

```
┌─────────────────────────────────────────────────────┐
│                 Target Protocol                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │
│  │  Oracle   │  │   DEX    │  │   Lending Pool    │ │
│  │ Chainlink │  │ Uniswap  │  │   Aave / Comp     │ │
│  │ Band      │  │ Curve    │  │                   │ │
│  │ TWAP      │  │ Balancer │  │                   │ │
│  └──────────┘  └──────────┘  └───────────────────┘ │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │
│  │  Bridge   │  │  Token   │  │   Governance      │ │
│  │ LayerZero │  │  ERC20s  │  │   Timelock        │ │
│  │ Wormhole  │  │  wETH    │  │   Governor        │ │
│  │           │  │  stETH   │  │                   │ │
│  └──────────┘  └──────────┘  └───────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Dependency checklist:**
- [ ] Which oracle(s) does the protocol use? (Chainlink, Uniswap TWAP, custom)
- [ ] What tokens does the protocol interact with? (fee-on-transfer? rebasing? ERC-777?)
- [ ] Does it integrate with other DeFi protocols? (composability risk)
- [ ] Any bridge dependencies? (cross-chain assets)
- [ ] External access control dependencies? (multisig, timelock, governance)
- [ ] What happens if a dependency is compromised? (oracle goes stale, DEX is drained)

---

## 2.7 GitHub & Codebase Recon

### What to Extract from a Protocol's GitHub

```bash
# Clone the repository
git clone https://github.com/protocol/contracts

# Check for audit reports
find . -name "*.pdf" -o -name "*audit*" -o -name "*security*" | head -20

# Look for TODO/FIXME/HACK comments — often mark incomplete security measures
grep -rn "TODO\|FIXME\|HACK\|XXX\|BUG\|SECURITY\|VULNERABLE" contracts/ --include="*.sol"

# Review recent changes to critical files
git log --oneline -20 contracts/core/

# Check for changes since last audit
git log --since="2024-01-01" --oneline contracts/

# Diff between audited commit and current deployment
git diff AUDITED_COMMIT..HEAD -- contracts/

# Find large recent commits (potential feature additions not yet audited)
git log --since="2024-06-01" --stat --diff-filter=M -- "*.sol" | head -100
```

### Audit History Analysis

| Checkpoint | What to Look For |
|-----------|-----------------|
| Audit report findings | Were all findings addressed? Check the fix commits |
| Time since audit | Code deployed long after audit may have unaudited changes |
| Scope coverage | Was the full codebase audited, or only specific contracts? |
| Known limitations | Auditors often list "out of scope" assumptions |
| Multiple audits | Consensus across auditors increases confidence |

---

## 2.8 Token Distribution & Whale Analysis

### Why It Matters

Token distribution affects governance security, liquidity depth, and rug pull risk.

```bash
# Using Etherscan Token Holder page
# https://etherscan.io/token/0xTOKEN#balances

# Top holder concentration
# If top 10 holders control >50% of supply → governance/dump risk

# Check if team tokens are locked
# Look for vesting contracts in top holders
cast call 0xVestingContract "beneficiary()(address)"
cast call 0xVestingContract "releaseTime()(uint256)"
```

### Red Flags in Token Distribution

| Signal | Risk |
|--------|------|
| Single address holds >20% | Centralization / dump risk |
| Unlocked team tokens > circulating supply | Potential rug pull |
| Large tokens in exchange hot wallets | Imminent sell pressure |
| Recent large transfers to new wallets | Possible pre-dump distribution |
| No liquidity locks | LP can be pulled at any time |

---

## 2.9 Social Engineering Vectors in Web3

### Attack Surfaces Specific to Web3

| Vector | Description | Example |
|--------|-------------|---------|
| **Discord compromise** | Hijacked admin accounts post phishing links | Bored Ape Yacht Club Discord hack |
| **Fake governance proposals** | Malicious proposals disguised as benign | Beanstalk governance attack |
| **Compromised npm packages** | Malicious dependencies in project repos | event-stream supply chain attack |
| **Fake verified contracts** | Uploading misleading source code to Etherscan | Honeypot tokens |
| **Impersonation** | Fake team members in Telegram/Discord | Social engineering validator keys |
| **Phishing dApps** | Cloning legitimate frontends | Fake Uniswap sites |
| **DNS hijacking** | Redirecting protocol domains to malicious dApps | Curve Finance DNS hijack |

### Recon on Social Channels

- Monitor Discord announcement channels for sudden permission changes
- Track Telegram group admin additions/removals
- Watch governance forums for unusual proposal activity
- Monitor Twitter/X for compromised team accounts

---

## 2.10 Subgraph Analysis (The Graph Protocol)

### Querying Subgraphs for Recon

The Graph indexes blockchain data into queryable subgraphs. Many DeFi protocols expose their state via subgraphs.

```graphql
# Query a Uniswap V3 subgraph for large swaps (potential whale activity)
{
  swaps(
    first: 10,
    orderBy: amountUSD,
    orderDirection: desc,
    where: { pool: "0xPoolAddress" }
  ) {
    id
    timestamp
    amountUSD
    sender
    recipient
    amount0
    amount1
  }
}

# Query governance proposals
{
  proposals(
    first: 10,
    orderBy: startBlock,
    orderDirection: desc
  ) {
    id
    proposer
    description
    forVotes
    againstVotes
    status
  }
}
```

**Subgraph recon targets:**
- Large recent transactions (whale activity, potential attack setup)
- Governance proposal patterns (flash loan governance setup)
- Liquidity provision/removal patterns (rug pull signals)
- Account activity over time (bot detection)

---

## 2.11 Transaction History Pattern Analysis

### What to Look For

```bash
# Using cast to analyze recent transactions
cast etherscan-source 0xContract --chain mainnet
cast logs --from-block 18500000 --to-block latest --address 0xContract
```

### Suspicious Patterns

| Pattern | Possible Implication |
|---------|---------------------|
| Many small test transactions before large ones | Attacker rehearsing exploit |
| `approve(MAX_UINT256)` to unknown contracts | Phishing approval or exploit setup |
| Flash loan interactions from new address | Potential attack preparation |
| Rapid sequence of governance-related txs | Governance attack in progress |
| Contract creation followed by immediate self-destruct | Metamorphic contract attack |
| Multiple token approvals to same new contract | Drainer contract setup |

---

## 2.12 Tools Reference

| Tool | Purpose | URL |
|------|---------|-----|
| **Etherscan** | Block explorer, contract verification | [etherscan.io](https://etherscan.io) |
| **Tenderly** | Tx debugging, simulation, forking | [tenderly.co](https://tenderly.co) |
| **Phalcon** | Attack transaction analysis | [phalcon.blocksec.com](https://phalcon.blocksec.com) |
| **Dune Analytics** | Custom on-chain SQL queries | [dune.com](https://dune.com) |
| **4byte.directory** | Function selector lookup | [4byte.directory](https://www.4byte.directory) |
| **Dedaub** | Bytecode decompilation | [app.dedaub.com](https://app.dedaub.com) |
| **Sourcify** | Contract verification (alternative to Etherscan) | [sourcify.dev](https://sourcify.dev) |
| **Arkham Intelligence** | Wallet labeling, entity tracking | [arkhamintelligence.com](https://www.arkhamintelligence.com) |
| **Nansen** | Wallet labeling, smart money tracking | [nansen.ai](https://www.nansen.ai) |
| **Heimdall-rs** | Bytecode analysis CLI | [github.com/Jon-Becker/heimdall-rs](https://github.com/Jon-Becker/heimdall-rs) |
| **BlockSec** | Security monitoring, attack detection | [blocksec.com](https://blocksec.com) |
| **cast** (Foundry) | CLI blockchain interaction | Ships with Foundry |

---

## Recon Methodology Checklist

Use this checklist at the start of every engagement:

- [ ] **Contract identification** — List all contracts in scope with addresses
- [ ] **Verification status** — Confirmed verified on Etherscan/Sourcify? If not, decompile
- [ ] **Proxy detection** — Check all EIP-1967 slots, identify implementation contracts
- [ ] **Admin/owner mapping** — Who controls upgrades, pauses, parameter changes?
- [ ] **Multisig analysis** — Threshold, owners, timelock delay?
- [ ] **Dependency mapping** — Oracles, DEXes, bridges, external contracts?
- [ ] **Deployer analysis** — Trace deployer wallet, find related contracts
- [ ] **Audit history** — Previous audits, findings, fix status?
- [ ] **Code changes since audit** — `git diff` from audited commit to deployed version
- [ ] **Token analysis** — Distribution, whale concentration, locking status?
- [ ] **Social channels** — Discord/Telegram admin security, governance activity?
- [ ] **Transaction patterns** — Any suspicious recent activity?

> **Key Takeaway:** In Web3 pentesting, recon is disproportionately valuable compared to traditional web pentesting. Because blockchains are public, you can often identify vulnerabilities entirely through on-chain analysis before reading a single line of source code. Develop the habit of doing thorough recon — it saves hours of manual code review and often reveals attack vectors that pure code auditing misses.

---

*← [Previous: Blockchain Fundamentals](./BLOCKCHAIN_FUNDAMENTALS.md) | [Next: Smart Contract Vulnerabilities →](./SMART_CONTRACT_VULNERABILITIES.md)*


<script>
document.addEventListener("DOMContentLoaded", function() {
  const checkboxes = document.querySelectorAll('.task-list-item-checkbox, input[type="checkbox"]');
  checkboxes.forEach(function(cb) {
    cb.removeAttribute('disabled');
    cb.style.cursor = 'pointer';
  });
});
</script>
