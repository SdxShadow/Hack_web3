---
title: "Module 09 — MEV & Mempool Security: Sandwich Attacks, Flashbots & Defense"
description: "Complete guide to Maximal Extractable Value (MEV) and mempool security: mempool monitoring with web3.py and ethers.js, front-running and sandwich attack mechanics, Flashbots bundle submission, Proposer-Builder Separation (PBS), dark pool transactions, time-bandit attacks, white-hat MEV rescue techniques, and MEV defense strategies."
keywords: "MEV, maximal extractable value, sandwich attack, front-running, Flashbots, mempool security, PBS proposer builder separation, dark pool, time-bandit attack, white-hat MEV, MEV defense, mempool monitoring, MEV-Boost architecture, ERC-4337 MEV, intent-based MEV mitigation"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 09 — MEV & Mempool Security | Web3 Hacker Guide"
og_description: "Master MEV: sandwich attacks, Flashbots bundle submission, PBS architecture, dark pools, and white-hat rescue techniques with working code examples."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/09_MEV_AND_MEMPOOL"
schema_type: "TechArticle"
difficulty: " Advanced →  Expert"
module: 9
tags: [MEV, sandwich-attack, Flashbots, mempool, front-running, PBS, dark-pool, white-hat-MEV]
nav_order: 9
parent: "Web3 Hacker & Pentester Guide"
---

# Module 09 — MEV & Mempool Security

> **Difficulty:** Advanced →  Expert
>
> Maximal Extractable Value (MEV) is the profit a block producer can extract by including, excluding, or reordering transactions. Understanding MEV is essential both for attacking and defending DeFi protocols. This module covers mempool monitoring, transaction ordering attacks, Flashbots, and MEV defense strategies.

---

## 9.1 Mempool Monitoring

### What Is the Mempool?

The mempool (memory pool) is the set of pending transactions waiting to be included in a block. On Ethereum, the mempool is public — anyone running a full node can observe it.

### Setting Up Mempool Monitoring

```bash
# Run a local full node (Geth) with mempool access
geth --syncmode "snap" --http --http.api "eth,txpool,debug,net" \
  --ws --ws.api "eth,txpool" --txpool.globalslots 50000

# Or use a provider with mempool access
# Alchemy, QuickNode, and bloXroute offer pending transaction streams
```

### Monitoring with Web3.py

```python
import asyncio
from web3 import Web3

w3 = Web3(Web3.WebsocketProvider("wss://eth-mainnet.g.alchemy.com/v2/KEY"))

async def monitor_mempool():
    pending_filter = w3.eth.filter('pending')
    while True:
        for tx_hash in pending_filter.get_new_entries():
            try:
                tx = w3.eth.get_transaction(tx_hash)
                if tx and tx['to']:
                    # Filter for specific contracts
                    if tx['to'].lower() == UNISWAP_ROUTER.lower():
                        decoded = decode_swap(tx['input'])
                        print(f"Swap detected: {decoded}")
                        # Analyze for sandwich opportunity
            except Exception:
                pass
        await asyncio.sleep(0.1)

asyncio.run(monitor_mempool())
```

### Monitoring with ethers.js

```javascript
const { ethers } = require("ethers");

const provider = new ethers.WebSocketProvider("wss://eth-mainnet.g.alchemy.com/v2/KEY");

provider.on("pending", async (txHash) => {
    const tx = await provider.getTransaction(txHash);
    if (tx && tx.to === UNISWAP_ROUTER) {
        const iface = new ethers.Interface(ROUTER_ABI);
        const decoded = iface.parseTransaction({ data: tx.data, value: tx.value });
        console.log(`Function: ${decoded.name}`);
        console.log(`Amount: ${decoded.args.amountIn}`);
        console.log(`Gas Price: ${tx.gasPrice}`);
    }
});
```

---

## 9.2 Transaction Ordering Attacks

### Front-Running

```
Attacker observes: Victim's pending swap of 100 ETH → USDC

Attacker's strategy:
1. Submit same swap BEFORE victim with HIGHER gas price
2. Attacker gets the better price
3. Victim gets worse price (more slippage)

Profitable when:
  attacker_profit > gas_cost_differential
  
The attacker's profit = price impact caused by front-running
```

### Back-Running

```
Attacker observes: Large DEX trade that creates an arbitrage opportunity

Strategy:
1. Wait for the large trade to execute
2. Submit arbitrage trade IMMEDIATELY AFTER in the same block
3. Profit from the price dislocation created by the large trade

Example:
- Large buy on Uniswap pushes ETH/USDC price up
- Attacker buys ETH on SushiSwap (still at old price)
- Sells on Uniswap (at new higher price)
- Profit = Price difference - Gas
```

### Sandwich Attacks — Detailed Mechanics

```solidity
// Sandwich attack implementation (for educational purposes)

// Step 1: Decode victim's pending swap
function analyzeVictimTx(bytes calldata data) internal returns (
    address tokenIn,
    address tokenOut,
    uint256 amountIn,
    uint256 amountOutMin  // This is the victim's slippage tolerance!
) {
    // Decode Uniswap V2 Router swapExactTokensForTokens
    (amountIn, amountOutMin, address[] memory path,,) =
        abi.decode(data[4:], (uint256, uint256, address[], address, uint256));
    tokenIn = path[0];
    tokenOut = path[path.length - 1];
}

// Step 2: Calculate optimal front-run amount
// The attacker wants to push the price just enough that the victim's
// amountOutMin is barely satisfied — extracting maximum value
function calculateOptimalFrontRun(
    uint256 victimAmountIn,
    uint256 victimAmountOutMin,
    uint112 reserveIn,
    uint112 reserveOut
) internal pure returns (uint256 frontRunAmount) {
    // Binary search or analytical formula for optimal amount
    // That pushes price to victim's slippage limit
}

// Step 3: Bundle as atomic transaction via Flashbots
// TX 1: Front-run (buy token)
// TX 2: Victim's swap (executes at worse price)
// TX 3: Back-run (sell token for profit)
```

### MEV Revenue Statistics

| MEV Type | Daily Revenue (approx) | Complexity |
|----------|----------------------|------------|
| Sandwich attacks | $1–5M | Medium |
| Arbitrage | $500K–2M | Low-Medium |
| Liquidations | $100K–1M | Medium |
| JIT liquidity | $100K–500K | High |
| Long-tail MEV | Variable | High |

---

## 9.3 Flashbots

### Architecture

```
Traditional flow:
  User → Public Mempool → Miner/Validator → Block

Flashbots flow:
  Searcher → Flashbots Bundle → MEV-Boost Relay → Block Builder → Proposer → Block

Key difference: Bundles are private — not visible in public mempool
```

### Submitting a Flashbots Bundle

```javascript
const { FlashbotsBundleProvider } = require("@flashbots/ethers-provider-bundle");
const { ethers } = require("ethers");

async function submitBundle() {
    const provider = new ethers.JsonRpcProvider("https://eth-mainnet.g.alchemy.com/v2/KEY");
    const authSigner = new ethers.Wallet(FLASHBOTS_AUTH_KEY);

    const flashbotsProvider = await FlashbotsBundleProvider.create(
        provider,
        authSigner,
        "https://relay.flashbots.net"
    );

    const bundle = [
        {
            signer: attackerWallet,
            transaction: {
                to: UNISWAP_ROUTER,
                data: frontRunCalldata,
                gasLimit: 250000,
                maxFeePerGas: ethers.parseUnits("50", "gwei"),
                maxPriorityFeePerGas: ethers.parseUnits("3", "gwei"),
                type: 2,
            },
        },
        // Victim's raw signed transaction (from mempool)
        { signedTransaction: victimRawTx },
        {
            signer: attackerWallet,
            transaction: {
                to: UNISWAP_ROUTER,
                data: backRunCalldata,
                gasLimit: 250000,
                maxFeePerGas: ethers.parseUnits("50", "gwei"),
                maxPriorityFeePerGas: ethers.parseUnits("3", "gwei"),
                type: 2,
            },
        },
    ];

    const targetBlock = await provider.getBlockNumber() + 1;
    const bundleSubmission = await flashbotsProvider.sendBundle(bundle, targetBlock);

    const resolution = await bundleSubmission.wait();
    if (resolution === 0) {
        console.log("Bundle included!");
    } else {
        console.log("Bundle not included (block passed)");
    }
}
```

### Flashbots Protect

```
For users who want to avoid being sandwiched:

1. Add Flashbots Protect RPC to MetaMask:
   Network Name: Flashbots Protect
   RPC URL: https://rpc.flashbots.net
   Chain ID: 1
   
Benefits:
- Transactions go directly to block builders, bypassing public mempool
- No sandwich attacks (tx isn't visible to searchers)
- Failed transactions don't cost gas (reverted bundles aren't included)
- MEV-Share returns some extracted MEV back to the user
```

---

## 9.4 MEV-Boost and PBS

### Proposer-Builder Separation (PBS)

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Searcher │───→│ Builder  │───→│  Relay   │───→│ Proposer │
│ (finds   │    │ (builds  │    │(connects │    │(proposes │
│  MEV)    │    │  optimal │    │ builder  │    │  block)  │
│          │    │  blocks) │    │ & prop.) │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘

Searcher: Finds MEV opportunities, creates bundles
Builder: Combines bundles into block, maximizes total value
Relay: Trustless intermediary — builder submits blocks, relay validates
Proposer: Selects highest-value block from relays
```

### Security Implications of PBS

| Risk | Description |
|------|-------------|
| **Builder centralization** | Top 3 builders build >90% of blocks on Ethereum |
| **Relay trust** | Relays see unencrypted blocks — can theoretically steal MEV |
| **Censorship** | Builders/relays can censor specific transactions (OFAC compliance) |
| **Proposer MEV** | Proposers can unbundle and steal MEV (mitigated by encrypted block proposals) |

---

## 9.5 Dark Pool Transactions

### Private Transaction Pools

| Protocol | Mechanism | Privacy Level |
|----------|-----------|--------------|
| **Flashbots Protect** | Routes to builders directly | Hides from public mempool |
| **MEV-Share** | Partial tx info shared with searchers | Partial — searchers bid for inclusion |
| **MEV Blocker** | Rebates extracted MEV to users | Hides from sandwich bots |
| **bloXroute** | Private mempool service | Full privacy until inclusion |
| **Eden Network** | Protected transactions for stakers | Staker priority ordering |

---

## 9.6 Time-Bandit Attacks

### Concept

A time-bandit attack involves reorganizing past blocks to steal MEV that was already extracted.

```
Block N:   Victim's DEX trade creates $10M arbitrage opportunity
Block N+1: Honest searcher captures the $10M arbitrage

Time-bandit attack:
1. Attacker with sufficient stake/hash power reorganizes blocks N and N+1
2. New Block N': Attacker captures the $10M arbitrage themselves
3. New Block N+1': Builds on attacker's new history

This is profitable when:
  MEV captured > cost of reorg (staking penalties, hash power cost)
```

### Ethereum PoS Mitigation

On PoS Ethereum, time-bandit attacks are mitigated by:
- **Finality**: After 2 epochs (~12.8 minutes), blocks are finalized and irreversible
- **Slashing**: Validators who propose conflicting blocks get slashed
- **Weak subjectivity**: Nodes won't accept reorgs past the finality checkpoint

---

## 9.7 Defending Against MEV

### Protocol-Level Defenses

| Defense | Mechanism | Trade-off |
|---------|-----------|-----------|
| **Commit-reveal** | Users commit to tx, reveal later | Adds latency (2 blocks minimum) |
| **Batch auctions** | Collect orders, execute at uniform price | No price discovery between batches |
| **Encrypted mempools** | Encrypt tx until ordering is finalized (Threshold encryption) | Requires complex cryptographic infrastructure |
| **Slippage limits** | Minimum acceptable output for swaps | User may need to set tight limits |
| **Deadline parameter** | Transaction expires after timestamp | Prevents stale-tx exploitation |
| **Private submission** | Use Flashbots Protect, MEV-Share | Reduces MEV but doesn't eliminate it |

### User-Level Defenses

```
1. Use Flashbots Protect RPC (free, easy setup in MetaMask)
2. Set tight slippage tolerance (0.5-1% instead of default 0.5-3%)
3. Include deadline parameter in all swaps
4. Break large trades into smaller chunks
5. Use DEX aggregators with private order routing (1inch Fusion, CoW Swap)
6. For governance votes: cast votes close to deadline to reduce front-running
```

---

## 9.8 White-Hat MEV for Attack Recovery

### How Ethical Hackers Use MEV Techniques

When a vulnerability is discovered in a live contract, white-hat rescuers use MEV techniques to:

1. **Front-run the attacker** — Exploit the vulnerability themselves to rescue funds before attacker
2. **Use Flashbots bundles** — Submit rescue transaction privately to avoid tipping off the attacker
3. **Coordinate with block builders** — Some builders have "white-hat fast lanes" for emergency rescues

### White-Hat Rescue Example

```javascript
// White-hat rescue via Flashbots
const rescueBundle = [
    {
        signer: whiteHatWallet,
        transaction: {
            to: vulnerableContract,
            data: rescueCalldata, // Calls the vulnerable function to move funds to safe address
            gasLimit: 500000,
            maxFeePerGas: ethers.parseUnits("100", "gwei"), // High priority
            maxPriorityFeePerGas: ethers.parseUnits("50", "gwei"),
        },
    },
];

// Submit as high-priority bundle — builders include it first
// Repeat for multiple upcoming blocks to ensure inclusion
for (let i = 0; i < 10; i++) {
    await flashbotsProvider.sendBundle(rescueBundle, currentBlock + i);
}
```

### Coordination Services

- **Flashbots White-Hat Hotline** — Direct contact for emergency rescues
- **SEAL 911** — Security experts available for emergency war-room coordination
- **Immunefi Emergency Response** — Platform-coordinated vulnerability triage

---

## 9.9 Tools

| Tool | Purpose | URL |
|------|---------|-----|
| **Flashbots** | Private mempool, bundle submission | [flashbots.net](https://flashbots.net) |
| **MEV-Share** | User-controlled MEV redistribution | [mev-share.flashbots.net](https://docs.flashbots.net/flashbots-mev-share/overview) |
| **bloXroute** | High-speed mempool access, private txs | [bloxroute.com](https://bloxroute.com) |
| **Eden Network** | Protected transactions | [edennetwork.io](https://edennetwork.io) |
| **EigenPhi** | MEV transaction analysis | [eigenphi.io](https://eigenphi.io) |
| **MEV-Boost** | PBS relay for validators | [github.com/flashbots/mev-boost](https://github.com/flashbots/mev-boost) |
| **zeromev** | MEV data analytics | [zeromev.org](https://zeromev.org) |
| **libmev** | MEV dashboard | [libmev.com](https://libmev.com) |

> **Key Takeaway:** MEV is not a bug — it's a fundamental property of any system where transaction ordering determines outcomes and ordering authority lies with a single entity. As a security researcher, understand MEV both offensively (to demonstrate front-running risks) and defensively (to recommend mitigations). The best defense is designing protocols where transaction ordering doesn't matter — through commit-reveal schemes, batch auctions, or encrypted mempools.

---

*← [Previous: Web3 dApp Pentesting](./WEB3_DAPP_PENTESTING.md) | [Next: CTF & Wargames →](./CTF_AND_WARGAMES.md)*
