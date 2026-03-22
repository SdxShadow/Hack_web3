---
title: "Module 07 — DeFi Protocol Attacks: AMMs, Lending, Bridges & More"
description: "Advanced analysis of DeFi-specific attack vectors: AMM sandwich attacks and JIT liquidity exploitation, lending protocol collateral manipulation, yield aggregator share inflation, stablecoin depeg attacks, bridge validator compromise, NFT marketplace exploits, governance takeovers, and perpetual DEX funding rate manipulation — with real-world case studies from $70M to $624M exploits."
keywords: "DeFi protocol attacks, AMM sandwich attack, flash loan attack, oracle manipulation, bridge exploit, Beanstalk hack, Mango Markets exploit, Ronin bridge, Wormhole exploit, lending protocol attack, yield aggregator vulnerability, JIT liquidity exploitation, collateral manipulation, stablecoin depeg death spiral, perpetual DEX funding rate attack"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 07 — DeFi Protocol Attacks | Web3 Hacker Guide"
og_description: "Deep analysis of every DeFi attack vector: AMM exploits, lending manipulation, bridge hacks, stablecoin death spirals, and governance attacks with billion-dollar case studies."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/07_DEFI_PROTOCOL_ATTACKS"
schema_type: "TechArticle"
difficulty: " Advanced →  Expert"
module: 7
tags: [DeFi-attacks, AMM, flash-loan, bridge-exploit, governance-attack, stablecoin, Beanstalk, Wormhole, Ronin, oracle-manipulation]
nav_order: 7
parent: "Web3 Hacker & Pentester Guide"
---

# Module 07 — DeFi Protocol Attacks

> **Difficulty:** Advanced →  Expert
>
> DeFi protocols are where the money is — and where the most sophisticated attacks occur. This module dissects the attack surface of every major DeFi primitive, with real-world case studies from exploits totaling billions in losses.

---

## 7.1 AMM Attacks (Uniswap V2/V3/V4)

### Uniswap V2 — Constant Product AMM (x × y = k)

**Attack surfaces:**
- **Sandwich attacks**: Monitor mempool for large swaps, front-run + back-run
- **Price oracle manipulation**: Using reserves as a price source
- **Flash swap exploitation**: Borrow from pool, manipulate, return in same tx

### Sandwich Attack Mechanics

```
Mempool:  Victim wants to swap 100 ETH → USDC

Attacker's bundle (submitted via Flashbots):
  TX 1 (Front-run):  Buy USDC with 50 ETH  → Price goes UP
  TX 2 (Victim):     Victim buys USDC at HIGHER price → Gets less USDC
  TX 3 (Back-run):   Sell USDC for ETH     → Profit from price impact

Profit = Victim's excess price impact - Gas costs
```

### Uniswap V3 — Concentrated Liquidity

**Additional attack surfaces:**
- **JIT (Just-In-Time) Liquidity**: Add concentrated liquidity right before a large swap, earn fees, remove immediately after
- **Tick manipulation**: Large swaps can cross price ticks, causing unexpected behavior
- **Oracle manipulation via TWAP**: V3's built-in oracle is more robust but still manipulable over time

### JIT Liquidity Attack

```
Block N:
1. MEV bot detects large pending swap (100 ETH → USDC)
2. Bot adds concentrated liquidity at the exact price tick
3. Large swap executes → bot earns disproportionate fees
4. Bot removes liquidity in same block

Result: Bot captures fees meant for long-term LPs
Defense: Accept this is part of the design; LPs can use wider ranges for protection
```

### Uniswap V4 — Hooks

**New attack surfaces with V4 hooks:**
- **Malicious hooks**: Custom hooks can front-run swaps, manipulate prices, or steal funds
- **Hook reentrancy**: Hooks execute during pool operations — reentrancy via hook callbacks
- **Flash accounting exploitation**: V4's transient storage for flash accounting introduces new state management risks

---

## 7.2 Lending Protocol Attacks (Aave, Compound, Euler)

### Collateral Manipulation

```
Attack Flow:
1. Deposit collateral (e.g., 1000 ETH worth of token X)
2. Manipulate oracle to inflate token X price
3. Borrow maximum against inflated collateral value
4. Price returns to normal → position is undercollateralized
5. Protocol absorbs bad debt
```

### Interest Rate Manipulation

```
Attack Flow (on utilization-based rate models):
1. Supply massive amount to lending pool → utilization drops → borrow rate drops
2. Borrow at artificially low rate
3. Withdraw supply → utilization spikes → rate increases for other borrowers
4. Other borrowers pay inflated rates

Profit: Borrow at low rate, lend elsewhere at market rate
```

### Bad Debt Creation

| Method | Description | Example |
|--------|-------------|---------|
| Oracle manipulation | Inflate collateral value, borrow max, let it default | Mango Markets |
| Illiquid collateral | Deposit token with thin liquidity, inflate price via small trade | CRV on Aave (near-exploit Nov 2022) |
| Cascading liquidations | Trigger liquidation cascade that depletes reserves | Multiple protocols during market crashes |

### Real-World Case: Euler Finance (March 2023) — $197M

```
Root cause: Flawed donation mechanism in eToken
Attack flow:
1. Flash loan → deposit to get eTokens
2. Donate eTokens to reserves (increases reserves without changing debt)
3. This creates an underwater position with health factor < 1
4. Self-liquidate at a profit
5. Repeat with increasing leverage

Key insight: The donate function didn't have proper accounting checks
```

---

## 7.3 Yield Aggregator Attacks

### Share Price Inflation

```
Attack on vault share pricing:
1. Deposit minimal amount → get 1 share
2. Donate large amount directly to vault
3. Share price = totalAssets / totalShares = (1 + donation) / 1
4. Next depositor's shares = deposit * totalShares / totalAssets
5. If deposit < donation, shares = 0 (rounds down)
6. Attacker redeems their 1 share for totalAssets

This is the ERC-4626 inflation attack ([Module 03, Section 3.24](./SMART_CONTRACT_VULNERABILITIES.md#324-donation--inflation-attack-on-vault-shares-erc-4626))
```

### Strategy Manipulation

```
Attack on yield strategy:
1. Identify vault's strategy (e.g., deposits into Curve, Aave, etc.)
2. Manipulate the downstream protocol's state
3. Trigger vault's harvest/rebalance function during manipulated state
4. Vault makes suboptimal trades at manipulated prices

Defense: Implement slippage protection in strategy functions
```

---

## 7.4 Stablecoin Attacks

### Depeg Attack Vectors

| Vector | Mechanism | Example |
|--------|-----------|---------|
| **Collateral drain** | Exploit lending against stablecoin collateral when reserves are thin | Iron Finance ($2B, June 2021) |
| **Oracle manipulation** | Make stablecoin appear worth more/less than $1 | UST depeg amplified by oracle lag |
| **Redemption bank run** | Massively redeem algorithmic stablecoins, breaking the peg mechanism | UST/LUNA ($40B, May 2022) |
| **Governance attack** | Change stablecoin parameters via governance exploit | Modify collateral ratios |

### Algorithmic Stablecoin Vulnerabilities

```
Death spiral pattern (UST/LUNA model):
1. Stablecoin starts to depeg (e.g., $0.98)
2. Arbitrageurs burn stable for governance token (mint LUNA)
3. Governance token price drops from sell pressure
4. Less confidence → more redemptions → more governance token minting
5. Hyperinflation of governance token → peg breaks entirely
```

---

## 7.5 Bridge Attacks

Bridges are the highest-value targets in Web3. They hold locked funds representing assets on other chains.

### Attack Taxonomy

| Attack Type | Description | Historical Exploits |
|-------------|-------------|-------------------|
| **Validator compromise** | Attacker obtains enough validator keys to forge attestations | Ronin ($624M), Harmony ($100M) |
| **Message forgery** | Bypass message verification to mint unbacked tokens | Wormhole ($326M) |
| **Replay across chains** | Same message accepted on multiple destination chains | Various |
| **Smart contract bug** | Vulnerability in the bridge's smart contracts | Nomad ($190M) |
| **Signature threshold** | Compromising M-of-N signers (when M is low) | Harmony Horizon (2-of-5) |

### Real-World Case: Nomad Bridge (August 2022) — $190M

```
Root cause: Trusted root initialized to 0x00 during upgrade

The verification logic:
  require(confirmAt[_root] != 0, "Not confirmed");
  // Since confirmAt[0x00] was set to a non-zero value during init,
  // ANY message with root 0x00 passed verification!

Impact: Anyone could copy a valid bridge transaction, change the recipient
to their own address, and the bridge would accept it. The exploit became
a "crowd-sourced" attack — hundreds of addresses participated.

Key lesson: Always verify initialization values of upgraded contracts
```

### Real-World Case: Ronin Bridge (March 2022) — $624M

```
Root cause: 5-of-9 validator threshold, but Axie Infinity DAO controlled 4 validators
+ 1 was compromised via social engineering (fake job offer PDF)

Attack flow:
1. Attacker compromised 5 validator keys through social engineering
2. Forged withdrawal messages for 173,600 ETH + 25.5M USDC
3. Signed with compromised keys
4. Bridge processed the forged withdrawal

Key lesson: Validator set security is paramount — diversity of key holders,
hardware security modules, operational security
```

---

## 7.6 NFT Marketplace Attacks

### Royalty Bypass

```solidity
// ERC-2981 royalties are NOT enforced at the protocol level
// Marketplaces voluntarily respect royaltyInfo(), but:
// 1. Users can transfer NFTs directly (no royalty payment)
// 2. Wrapper contracts can obscure transfers
// 3. Some marketplaces (Blur, Sudoswap) don't enforce royalties

// Attack: Create wrapper that purchases NFT without paying royalties
contract RoyaltyBypass {
    function purchaseWithoutRoyalty(address marketplace, bytes calldata data) external {
        // Direct low-level call to marketplace, ignoring royalty payment
        marketplace.call(data);
    }
}
```

### Bid Manipulation

```
Attack on English auction:
1. Place high bid near auction end
2. In the same block, cancel/front-run legitimate bids
3. If last-second bidding is allowed, grief by bidding 1 wei above
4. Block the auction finalization via gas griefing in receive()
```

### Metadata Exploits

```
- IPFS CID spoofing: Different metadata for same CID if hash collision found (extremely unlikely)
- HTTP metadata: If tokenURI uses HTTP, owner can change the image/properties post-sale
- SVG injection: On-chain SVG NFTs can contain JavaScript in metadata viewers
```

---

## 7.7 Governance Attacks

### Flash Loan Governance

```
Pre-conditions: Governance uses current token balance (not snapshots)

Attack:
1. Flash loan governance tokens
2. Create + vote on malicious proposal (drain treasury, change admin)
3. If proposal executes immediately (no timelock) → funds are drained
4. Return flash-loaned tokens

Defense: Use ERC20Votes with snapshot-based voting (getPastVotes)
```

### Timelock Bypass

```
Scenarios where timelocks can be circumvented:
1. Emergency functions that bypass timelock (admin key + emergency = no delay)
2. Governance proposal to modify the timelock itself (set delay to 0)
3. Multiple proposals that individually look harmless but combined are malicious
```

### Vote Manipulation

```
Beyond flash loans:
- Bribe attacks: Pay voters off-chain to vote a certain way (Dark DAOs)
- Vote buying markets: Protocols like Convex/Votium enable voting power markets
- Delegation exploits: Accumulate delegated voting power, use it maliciously
```

---

## 7.8 Liquid Staking Attacks

### Validator Manipulation

```
Attack surface:
- MEV extraction by liquid staking operators (reordering validator duties)
- Slashing exploitation: Force a validator to get slashed to profit from insurance
- Withdrawal delay exploitation: Lock up funds, manipulate price during unbonding period
```

### Oracle Manipulation for LSTs

```
stETH, rETH, cbETH all have "exchange rates" vs ETH:
- If a lending protocol uses the wrong exchange rate source → manipulation
- If the rate is updated on-chain by an oracle → staleness / manipulation
- If the rate assumes 1:1 with ETH → depeg during market stress causes bad debt
```

---

## 7.9 Perpetual DEX Attacks

### Funding Rate Manipulation

```
Attack:
1. Open large long position on perpetual DEX
2. This increases the funding rate (longs pay shorts)
3. Open corresponding short position on a different venue
4. Collect elevated funding payments on the short
5. Close positions when funding normalizes

Defense: Cap funding rates, use longer TWAP for rate calculation
```

### Liquidation Cascades

```
Attack on thin markets:
1. Identify highly leveraged positions on perpetual DEX
2. Large market dump → triggers liquidation of leveraged longs
3. Liquidation sales push price further down → cascade of liquidations
4. Buy at the bottom after cascade exhausts
5. Profit from the artificial crash

Defense: Implement partial liquidation, backstop liquidity, circuit breakers
```

---

## 7.10 Real-World Case Studies — Detailed Analysis

### Beanstalk (April 2022) — $182M

```
Type: Flash loan governance attack
Chain: Ethereum mainnet

Attack flow:
1. Flash loan $1B+ in stablecoins from Aave
2. Swap for enough BEAN governance tokens to pass BIP-18
3. BIP-18 was a malicious proposal that transferred all Beanstalk assets
   to the attacker's wallet
4. Execute the proposal (no timelock — emergency governance pathway)
5. Repay flash loans
6. Donate $250K to Ukraine fund (attacker's statement)

Root cause:
- No snapshot-based voting (used current balance)
- Emergency governance path had no timelock
- Single-transaction proposal + vote + execute

Key lesson: ALWAYS use snapshot-based voting + mandatory timelock
```

### Mango Markets (October 2022) — $114M

```
Type: Oracle manipulation + over-borrowing
Chain: Solana

Attack flow:
1. Attacker opened large MNGO perp position on Mango
2. Spot-bought MNGO on secondary markets → price pumped
3. Mango's oracle reflected the pumped price
4. Unrealized PnL on the perp position increased dramatically
5. Used inflated position value as collateral to borrow all available assets
6. Left the protocol with $114M in bad debt

Root cause:
- Oracle used spot price with insufficient TWAP smoothing
- No borrow caps per user
- Unrealized PnL treated as available collateral

Key lesson: Separate unrealized PnL from borrowable collateral
```

### Curve/Vyper Reentrancy (July 2023) — ~$70M

```
Type: Reentrancy via compiler bug
Chain: Ethereum mainnet

Root cause:
- Vyper compiler versions 0.2.15, 0.2.16, 0.3.0 had a bug in the
  reentrancy lock implementation
- The @nonreentrant decorator failed to properly compile, leaving pools
  without reentrancy protection
- Affected pools: alETH/ETH, msETH/ETH, pETH/ETH on Curve

Attack flow:
1. Add liquidity to affected Curve pool
2. Call remove_liquidity() which sends ETH via raw call
3. In the receive() callback, call add_liquidity() (reentrant!)
4. Pool pricing is based on stale state → profit from mispricing
5. Remove newly added liquidity at profit

Key lesson: Compiler bugs exist. Verify bytecode reentrancy guards.
Even non-Solidity languages have critical bugs.
```

### Wormhole (February 2022) — $326M

```
Type: Signature verification bypass
Chain: Solana side of Ethereum↔Solana bridge

Root cause:
- The Wormhole guardian network validates cross-chain messages
- The Solana contract used a deprecated function to verify guardian signatures
- The deprecated function didn't properly validate the guardian set

Attack:
1. Attacker crafted a fake VAA (Verified Action Approval)
2. Bypassed signature verification on Solana
3. Minted 120,000 wETH on Solana (unbacked by any ETH on Ethereum)
4. Bridged 93,750 wETH back to Ethereum → received real ETH

Key lesson: Audit all signature verification paths.
Use current, non-deprecated verification methods.
```

---

## Summary — DeFi Attack Surface by Protocol Type

| Protocol Type | Top Attack Vectors | Defense Strategy |
|--------------|-------------------|------------------|
| **AMMs** | Sandwich, oracle manipulation, JIT liquidity | TWAP oracles, slippage limits, private txs |
| **Lending** | Collateral manipulation, bad debt, oracle staleness | Chainlink, borrow caps, liquidation buffers |
| **Yield Aggregators** | Share inflation, strategy manipulation | Minimum shares, virtual offsets, slippage in strategies |
| **Stablecoins** | Depeg cascades, collateral drain | Over-collateralization, circuit breakers |
| **Bridges** | Validator compromise, message forgery | Decentralized validator sets, fraud proofs, ZK proofs |
| **NFT Marketplaces** | Royalty bypass, bid manipulation | On-chain royalty enforcement, auction mechanics |
| **Governance** | Flash loan voting, bribe attacks | Snapshot voting, timelock, vote escrow |
| **Perp DEXs** | Funding rate manipulation, liquidation cascades | TWAP funding, partial liquidation, backstops |

> **Key Takeaway:** The most profitable DeFi exploits are economic in nature — they don't rely on code bugs but on flawed economic assumptions. Understanding AMM math, lending mechanics, and oracle design is more important than memorizing Solidity anti-patterns. When auditing DeFi protocols, always ask: "If I had $1 billion for one transaction, what could I break?"

---

*← [Previous: Exploit Development](./EXPLOIT_DEVELOPMENT.md) | [Next: Web3 dApp Pentesting →](./WEB3_DAPP_PENTESTING.md)*
