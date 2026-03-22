---
title: "Module 01 — Blockchain Fundamentals for Security Researchers"
description: "Deep dive into EVM architecture, opcode-level analysis, Solidity storage internals, transaction lifecycle, gas mechanics, ABI encoding, consensus security, L2 architecture, and cross-chain bridge attack surfaces for Web3 security researchers."
keywords: "EVM internals, Ethereum opcodes, Solidity storage layout, delegatecall vulnerability, blockchain fundamentals, smart contract security, Layer 2 security, bridge exploits, ABI encoding, transaction lifecycle, gas optimization security, EVM memory vs storage, EIP-1559 gas mechanics, rollup security architecture, optimistic vs ZK rollup risks"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 01 — Blockchain Fundamentals | Web3 Hacker Guide"
og_description: "Complete EVM internals guide for security researchers: opcodes, storage layout, ABI encoding, L2 architecture, and bridge vulnerabilities with real-world exploit context."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/01_BLOCKCHAIN_FUNDAMENTALS"
schema_type: "TechArticle"
difficulty: " Beginner →  Intermediate"
module: 1
tags: [EVM, opcodes, Solidity, storage, blockchain, L2, bridges, delegatecall, ABI, gas]
nav_order: 1
parent: "Web3 Hacker & Pentester Guide"
---

# Module 01 — Blockchain Fundamentals for Security Researchers

> **Difficulty:** Beginner →  Intermediate
>
> Before you can break smart contracts, you must deeply understand how the Ethereum Virtual Machine executes code, how transactions flow through the network, and how data is stored on-chain. This module covers every foundational concept a security researcher needs.

---

## 1.1 EVM Architecture

The **Ethereum Virtual Machine (EVM)** is a deterministic, stack-based, 256-bit virtual machine that executes smart contract bytecode. Understanding its internals is non-negotiable for security researchers — every vulnerability ultimately maps to EVM behavior.

### Stack Machine Model

The EVM operates on a **last-in, first-out (LIFO) stack** with a maximum depth of **1024 items**, where each item is a 256-bit (32-byte) word.

```
┌─────────────────────────────────────┐
│            EVM Execution            │
├─────────┬───────────┬───────────────┤
│  Stack  │  Memory   │   Storage     │
│ (LIFO)  │ (byte[])  │ (key→value)   │
│ 1024    │ volatile  │ persistent    │
│ items   │ per call  │ per contract  │
│ 256-bit │ linear    │ 256→256 bit   │
└─────────┴───────────┴───────────────┘
```

### Key EVM Data Locations

| Location | Persistence | Cost | Access Pattern |
|----------|------------|------|----------------|
| **Stack** | Call-scoped | Cheapest | Push/pop (LIFO) |
| **Memory** | Call-scoped | Cheap (linear expansion cost) | Byte-addressable, linear |
| **Storage** | Permanent | Expensive (20K gas write, 5K modify) | 256-bit key → 256-bit value |
| **Calldata** | Transaction-scoped | Read-only, cheap | Byte-addressable, immutable |
| **Returndata** | Call-scoped | After external call | Byte-addressable |
| **Code** | Permanent | Via `CODECOPY` | Contract bytecode |

### Critical Opcodes for Security Researchers

```
── Arithmetic ──
ADD, SUB, MUL, DIV, MOD     // No overflow checks pre-0.8.x!
ADDMOD, MULMOD               // Modular arithmetic
EXP                          // Exponentiation (gas-intensive)
SIGNEXTEND                   // Sign extension

── Comparison & Logic ──
LT, GT, SLT, SGT, EQ        // Comparisons
ISZERO, AND, OR, XOR, NOT    // Bitwise logic

── Storage & Memory ──
SLOAD (slot)                 // Read storage: 2100 gas (cold), 100 gas (warm)
SSTORE (slot, value)         // Write storage: 20000 gas (new), 5000 gas (modify)
MLOAD, MSTORE, MSTORE8       // Memory operations
CALLDATALOAD, CALLDATASIZE    // Read calldata
RETURNDATASIZE, RETURNDATACOPY // After external calls

── Control Flow ──
JUMP, JUMPI, JUMPDEST        // Control flow
REVERT, RETURN, STOP         // Execution termination
INVALID                      // Consume all remaining gas

── External Calls ──
CALL         (gas, to, value, inOff, inLen, outOff, outLen) // External call
STATICCALL   (gas, to, inOff, inLen, outOff, outLen)        // Read-only call
DELEGATECALL (gas, to, inOff, inLen, outOff, outLen)        // Preserves msg.sender & storage
CALLCODE     (gas, to, value, inOff, inLen, outOff, outLen) // Deprecated, use delegatecall

── Contract Creation ──
CREATE       // Deploy contract: address = keccak256(sender, nonce)
CREATE2      // Deterministic: address = keccak256(0xFF, sender, salt, initCodeHash)

── Block & Transaction Info ──
BLOCKHASH, COINBASE, TIMESTAMP, NUMBER, DIFFICULTY/PREVRANDAO
GASPRICE, GASLIMIT, CHAINID, SELFBALANCE, BASEFEE
CALLER (msg.sender), ORIGIN (tx.origin), CALLVALUE (msg.value)

── Self-Destruct ──
SELFDESTRUCT // Destroy contract, force-send ETH to target
             // Deprecated post-Dencun (EIP-6780): only works in same tx as creation
```

> **Security Insight:** The distinction between `CALL`, `DELEGATECALL`, and `STATICCALL` is the root cause of entire vulnerability classes. `DELEGATECALL` preserves the caller's `msg.sender` and storage context — this is what makes proxy patterns work, and also what makes delegatecall vulnerabilities devastating.

### Gas Costs — What Matters for Exploits

| Operation | Gas Cost | Security Implication |
|-----------|----------|---------------------|
| `SSTORE` (0 → non-zero) | 20,000 | DoS via storage inflation |
| `SSTORE` (non-zero → zero) | Refund 4,800 | Gas token exploits (historical) |
| `SLOAD` (cold) | 2,100 | First access is expensive |
| `SLOAD` (warm) | 100 | Subsequent access is cheap |
| `CALL` with value | 9,000 + 2,300 stipend | Reentrancy window at 2,300 gas |
| `LOG0`–`LOG4` | 375 + 375*topics + 8*bytes | Event-heavy contracts cost more |
| `CREATE` | 32,000 + deployment cost | Factory pattern gas overhead |

---

## 1.2 Ethereum Account Types

### Externally Owned Accounts (EOA)

- Controlled by a private key (secp256k1 ECDSA)
- Has a **nonce** (transaction counter) and **balance**
- Cannot hold code
- Initiates transactions (only EOAs can start a tx — prior to ERC-4337)

### Contract Accounts

- Controlled by code (bytecode deployed at the address)
- Has a **nonce** (incremented by `CREATE`), **balance**, **code hash**, **storage root**
- Cannot initiate transactions autonomously
- Address derived from creator address + nonce (`CREATE`) or from salt + init code hash (`CREATE2`)

```
EOA Address:  keccak256(publicKey)[12:]  // Last 20 bytes of pubkey hash
CREATE:       keccak256(rlp([sender, nonce]))[12:]
CREATE2:      keccak256(0xFF ++ sender ++ salt ++ keccak256(initCode))[12:]
```

### Nonce Mechanics

| Account Type | Nonce Incremented By | Security Relevance |
|-------------|---------------------|-------------------|
| EOA | Each outgoing tx | Prevents replay attacks |
| Contract | Each `CREATE` call | Predictable contract addresses |

> **Security Insight:** `CREATE2` addresses are deterministic and can be precomputed. An attacker can `SELFDESTRUCT` a contract at a known address and re-deploy different code there (pre-Dencun). This is the basis for metamorphic contract attacks.

---

## 1.3 Transaction Lifecycle

Understanding how a transaction moves from the user's wallet to block inclusion is critical for MEV, front-running, and censorship analysis.

```
┌──────────┐    ┌─────────┐    ┌──────────────┐    ┌────────────┐    ┌───────────┐
│  Wallet  │───→│ RPC Node│───→│   Mempool    │───→│  Validator │───→│   Block   │
│  signs   │    │ validate│    │  (pending)   │    │  (builder) │    │ inclusion │
│  tx      │    │ nonce,  │    │  propagated  │    │  orders by │    │ finalized │
│          │    │ gas,    │    │  to peers    │    │  priority  │    │           │
│          │    │ balance │    │              │    │  fee / MEV │    │           │
└──────────┘    └─────────┘    └──────────────┘    └────────────┘    └───────────┘
                                     │
                                     
                              ┌──────────────┐
                              │  MEV Bots /  │
                              │  Searchers   │
                              │  monitor &   │
                              │  front-run   │
                              └──────────────┘
```

### Transaction Fields

```json
{
  "type": "0x02",           // EIP-1559 transaction
  "nonce": "0x0",           // Sender's tx count
  "to": "0xContract...",    // Recipient (null for contract creation)
  "value": "0x0",           // ETH transferred (in wei)
  "data": "0xa9059cbb...", // Calldata (function selector + args)
  "maxFeePerGas": "30 gwei",
  "maxPriorityFeePerGas": "2 gwei",
  "gasLimit": "21000",
  "chainId": "1",
  "v": "0x1c", "r": "0x...", "s": "0x..."  // ECDSA signature
}
```

### Mempool Security Implications

1. **Transactions are public before inclusion** — anyone monitoring the mempool can see pending txs
2. **Front-running**: Submitting a higher-gas-price tx to get included before the victim
3. **Sandwich attacks**: Surrounding a victim's swap with buy+sell txs
4. **Private mempools** (Flashbots Protect, MEV-Share) route txs directly to builders, bypassing public mempool

---

## 1.4 Gas Mechanics — EIP-1559

Post-EIP-1559, Ethereum uses a dual-fee model:

```
Total Fee = Gas Used × (Base Fee + Priority Fee)

Base Fee:     Protocol-determined, burned. Adjusts ±12.5% per block based on utilization.
Priority Fee: User-set tip to the validator. Incentivizes inclusion.
Max Fee:      Maximum total fee user is willing to pay.
Actual Fee:   min(maxFeePerGas, baseFee + maxPriorityFeePerGas)
```

### Gas Security Considerations

| Attack Vector | Description |
|--------------|-------------|
| **Gas griefing** | Consuming excessive gas in callback functions to cause the caller's tx to run out of gas |
| **Unbounded loops** | Iterating over growing arrays — DoS when array becomes large enough that tx exceeds block gas limit |
| **Return bomb** | Returning excessively large returndata to consume caller's memory expansion gas |
| **Insufficient gas forwarding** | Using `transfer()` / `send()` which forward only 2,300 gas — not enough for complex `receive()` functions |
| **Block stuffing** | Filling blocks with high-gas txs to delay time-sensitive operations |

---

## 1.5 ABI Encoding / Decoding

The **Application Binary Interface (ABI)** defines how functions and their parameters are encoded in calldata. Understanding this is essential for analyzing raw transactions and crafting exploit payloads.

### Function Selector

```
selector = keccak256("transfer(address,uint256)")[0:4]
         = 0xa9059cbb
```

The first 4 bytes of the keccak256 hash of the function signature identify which function to call.

### Argument Encoding (ABI v2)

```
Static types (uint256, address, bool):  Padded to 32 bytes, placed inline
Dynamic types (bytes, string, arrays):  Offset pointer inline, data at end

Example: transfer(address to, uint256 amount)
Calldata:
  0xa9059cbb                                                     // selector
  000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045 // to (padded to 32 bytes)
  0000000000000000000000000000000000000000000000000de0b6b3a7640000 // amount = 1e18
```

### ABI Encoding Gotchas for Auditors

- **Collision risk**: Different function signatures can produce the same 4-byte selector (birthday paradox over 2^32 space). Tools like 4byte.directory catalog known collisions.
- **Non-standard encoding**: Some contracts hand-craft calldata, bypassing the ABI encoder — watch for raw `abi.encodePacked()` which can cause hash collisions.
- **Tuple encoding**: Complex nested structs can be confusing — always decode with `cast calldata-decode` or `4byte.directory`.

```bash
# Decode calldata using cast
cast calldata-decode "transfer(address,uint256)" 0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa9604500000000000000000000000000000000000000000000000000de0b6b3a7640000

# Lookup a selector
cast 4byte 0xa9059cbb
# Output: transfer(address,uint256)
```

---

## 1.6 Solidity Storage Internals

### Storage Layout

Solidity uses a flat **256-bit key → 256-bit value** storage model. State variables are assigned to sequential **slots** starting from slot 0.

```solidity
contract StorageLayout {
    uint256 public a;           // Slot 0
    uint256 public b;           // Slot 1
    address public owner;       // Slot 2 (20 bytes, left-padded)
    bool public paused;         // Slot 2 (packed with owner — 1 byte)
    uint128 public x;           // Slot 3
    uint128 public y;           // Slot 3 (packed with x)
    mapping(address => uint256) public balances;  // Slot 4 (base slot)
    uint256[] public arr;       // Slot 5 (length stored here)
}
```

### Variable Packing

Variables smaller than 32 bytes are packed into the **same slot** if they fit. They are packed right-to-left (lower-order bits first).

```
Slot 2: [12 bytes padding][20 bytes owner][1 byte paused]
Slot 3: [16 bytes x][16 bytes y]
```

### Mapping Storage

```
// For mapping at slot p with key k:
slot = keccak256(h(k) . p)     // . = concatenation, h() = pad to 32 bytes

// Nested mapping[k1][k2] at slot p:
slot = keccak256(h(k2) . keccak256(h(k1) . p))
```

### Dynamic Array Storage

```
// For array at slot p:
// arr.length is stored at slot p
// arr[i] is stored at: keccak256(p) + i
```

### Reading Storage Directly

```bash
# Read slot 0 of a contract
cast storage 0xContractAddress 0

# Read a specific mapping value: balances[0xUser] where mapping is at slot 4
cast index address 0xUserAddress 4 | xargs cast storage 0xContractAddress

# Using forge inspect for layout
forge inspect ContractName storage-layout
```

> **Security Insight:** Storage layout knowledge is essential for exploiting proxy storage collisions. When a proxy uses `DELEGATECALL` to an implementation, both share the same storage — if their layouts conflict, state corruption occurs. This is why EIP-1967 reserves specific slots (e.g., `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc` for implementation address).

---

## 1.7 Bytecode & Disassembly

### Contract Deployment

When a contract is deployed, the **init code** (constructor bytecode) runs once and returns the **runtime bytecode** — the code stored on-chain.

```
Deployment Tx Data = [init code (constructor)] → executes → returns [runtime bytecode]
On-chain code = runtime bytecode only
```

### Reading Raw Bytecode

```bash
# Get deployed bytecode
cast code 0xContractAddress --rpc-url https://eth-mainnet.g.alchemy.com/v2/KEY

# Disassemble bytecode
cast disassemble $(cast code 0xContractAddress)

# Use heimdall for decompilation
heimdall decompile 0xContractAddress --rpc-url https://eth-mainnet.g.alchemy.com/v2/KEY
```

### Bytecode Structure

```
[runtime bytecode]
├── Function dispatcher (switch on selector)
│   ├── 0xa9059cbb → transfer()
│   ├── 0x70a08231 → balanceOf()
│   └── fallback
├── Function bodies
├── Free memory pointer setup (start at 0x80)
└── Metadata hash (Solidity compiler appends CBOR-encoded metadata)
    // Last ~43 bytes: a264697066735822... (IPFS hash of metadata)
```

### Why Bytecode Analysis Matters

1. **Unverified contracts** — Many deployed contracts never upload source to Etherscan. Decompilation is the only option.
2. **Compiler bugs** — The Solidity compiler has had bugs that produce incorrect bytecode despite correct source code.
3. **Inline assembly** — `assembly {}` blocks bypass Solidity safety checks — visible only in bytecode.
4. **Obfuscated logic** — Some malicious contracts intentionally obscure logic (honeypots, rug pulls).

---

## 1.8 Consensus Mechanisms & Security Implications

### Proof of Work (PoW) — Historical

| Property | Detail |
|----------|--------|
| **Security model** | Hash power majority (51% attack) |
| **Block time** | ~13s (variable) |
| **Finality** | Probabilistic (6+ confirmations) |
| **MEV** | Miners order txs |
| **Attack cost** | Hardware + electricity |

### Proof of Stake (PoS) — Ethereum Post-Merge

| Property | Detail |
|----------|--------|
| **Security model** | 32 ETH staked per validator |
| **Block time** | 12s (fixed slots) |
| **Finality** | ~12.8 minutes (2 epochs) |
| **MEV** | Proposers order txs (PBS via MEV-Boost) |
| **Attack cost** | 1/3 validators to halt, 2/3 to finalize bad chain |
| **Slashing** | Validators lose stake for equivocation |

### Delegated Proof of Stake (DPoS)

Used by chains like EOS, TRON, BNB Chain (partially). A fixed set of validators elected by token holders.

**Security implications:**
- Fewer validators = easier coordination for censorship
- Vote-buying attacks on delegate elections
- Centralization risks when stake concentrates

### Security Comparison

| Attack | PoW | PoS | DPoS |
|--------|-----|-----|------|
| 51% attack | $$$ (hardware) | $$$ (1/3 stake) | Fewer validators needed |
| Long-range attack | N/A | Possible (mitigated by checkpoints) | Possible |
| Censorship | Costly (minority mining) | Proposer censorship | Easy with few delegates |
| Finality reversion | Possible with hash power | Requires ≥1/3 malicious stake | Easier |
| Time-bandit attack | Profitable if block reward > reorg cost | Economic penalties (slashing) | Lower cost |

---

## 1.9 Layer 2 Architecture

### Optimistic Rollups (Arbitrum, Optimism, Base)

```
┌─────────────────────────────────────────────┐
│              Ethereum L1 (DA Layer)         │
│  ┌────────────────────────────────────────┐ │
│  │  Rollup Contract (state root, batch)  │ │
│  │  Fraud Proof Window: 7 days           │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
                     
                     │ Batched calldata / blobs
┌─────────────────────────────────────────────┐
│              L2 Sequencer                   │
│  Executes txs → compresses → posts to L1   │
│  Trust assumption: sequencer can censor     │
│  but CANNOT steal funds (fraud proofs)      │
└─────────────────────────────────────────────┘
```

**Security considerations:**
- **7-day challenge period** — withdrawals delayed; bridge exploit window
- **Sequencer centralization** — single sequencer can censor/reorder txs
- **Fraud proof liveness** — requires at least one honest challenger
- **Forced inclusion** — users can submit txs directly to L1 if sequencer censors

### ZK-Rollups (zkSync, StarkNet, Polygon zkEVM, Scroll)

```
┌─────────────────────────────────────────────┐
│              Ethereum L1                    │
│  ┌────────────────────────────────────────┐ │
│  │  Verifier Contract (ZK proof check)   │ │
│  │  Instant finality once proof verified │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
                     
                     │ ZK Proof + State diff
┌─────────────────────────────────────────────┐
│              L2 Prover / Sequencer          │
│  Executes txs → generates ZK proof         │
│  Proof validates all state transitions      │
└─────────────────────────────────────────────┘
```

**Security considerations:**
- **Prover bugs** — incorrect circuit constraints could allow invalid state transitions
- **Trusted setup** (for SNARKs) — compromised ceremony = fake proofs
- **Escape hatch** — must allow users to exit if prover goes offline
- **EVM equivalence** — differences from L1 EVM can create unexpected behavior
- **Upgrade mechanisms** — many ZK-rollups retain admin keys to upgrade verifier contracts

### State Channels (Raiden, Lightning Network on BTC)

Off-chain bilateral channels with on-chain dispute resolution.

**Security considerations:**
- Requires watchtower services to prevent fraudulent channel closure
- Channel griefing (forcing expensive on-chain dispute resolution)
- Data availability: losing channel state = losing funds

### Plasma (largely deprecated)

Child chains with periodic commitments to L1. Replaced by rollups due to data availability problems.

---

## 1.10 Cross-Chain Bridges

### Architecture Patterns

| Type | Trust Model | Examples | Risk Level |
|------|------------|----------|------------|
| **Lock-and-mint** | Relies on bridge validators to attest to lock event | Wormhole, Ronin | High — validator compromise = total fund theft |
| **Burn-and-mint** | Token burnt on source chain, minted on destination | LayerZero (OFT) | Medium — depends on oracle/relayer security |
| **Liquidity network** | Liquidity providers on both chains, atomic swaps | Connext, Hop | Lower — no wrapped assets, limited by liquidity |
| **Native rollup bridge** | Uses L1 contracts + fraud/validity proofs | Optimism, Arbitrum native bridge | Lowest — inherits L1 security |

### Bridge Attack Surface

```
Source Chain                        Destination Chain
┌──────────┐    ┌─────────────┐    ┌──────────┐
│ Lock/Burn │───→│  Attestation │───→│ Mint/    │
│ Contract  │    │  Layer       │    │ Unlock   │
└──────────┘    │ (validators, │    └──────────┘
                │  oracles,    │
                │  relayers)   │
                └─────────────┘
                       
              Attack vectors:
              1. Validator key compromise
              2. Message forgery
              3. Replay across chains
              4. Signature threshold exploit
              5. Oracle manipulation
```

### Real-World Bridge Exploits

| Exploit | Loss | Root Cause |
|---------|------|-----------|
| **Ronin Bridge** (Mar 2022) | $624M | 5/9 validator keys compromised (social engineering) |
| **Wormhole** (Feb 2022) | $326M | Signature verification bypass in Solana guardian |
| **Nomad** (Aug 2022) | $190M | Trusted root initialized to 0x00 — any message valid |
| **Harmony Horizon** (Jun 2022) | $100M | 2/5 multisig compromise |

> **Key Takeaway:** Bridges are the highest-risk component in Web3. They combine smart contract risk with validator/multisig trust assumptions, making them attractive targets for nation-state-level attackers. As a pentester, always map a protocol's bridge dependencies.

---

## 1.11 IPFS & Decentralized Storage

### How IPFS Integrates with dApps

**IPFS (InterPlanetary File System)** is a content-addressed storage network. Files are identified by their hash (CID — Content Identifier), not by location.

```
Traditional: https://example.com/image.png     → Location-addressed
IPFS:        ipfs://QmX7b3eE5gYT3aW8Cqf...     → Content-addressed
```

### Security Implications

| Issue | Description |
|-------|-------------|
| **NFT metadata mutability** | If `tokenURI()` points to an HTTP gateway instead of IPFS, the owner can change the image/metadata after sale |
| **IPFS pinning dependency** | Content is only available if someone pins it — unpinned content disappears |
| **Gateway trust** | `https://ipfs.io/ipfs/Qm...` routes through a centralized gateway — MITM possible |
| **Content injection** | If a contract stores IPFS hashes on-chain, the deployer can store arbitrary content |
| **Arweave vs IPFS** | Arweave provides permanent storage (pay once, store forever). IPFS requires ongoing pinning. |

```bash
# Retrieve NFT metadata from IPFS
curl https://ipfs.io/ipfs/QmeSjSinHpPnmXmspMjwiXyN6zS4E9zccariGR3jxcaWtq/1

# Pin content (preventing garbage collection)
ipfs pin add QmHash
```

---

## Summary & Key Takeaways

| Concept | Why It Matters for Pentesting |
|---------|------------------------------|
| EVM stack/memory/storage | Understanding exploit mechanics at the opcode level |
| `DELEGATECALL` vs `CALL` | Proxy vulnerabilities, storage collisions |
| Transaction lifecycle | MEV extraction, front-running attacks |
| Storage layout | Proxy collisions, state manipulation, direct storage reads |
| ABI encoding | Crafting exploit payloads, decoding attack transactions |
| Bytecode analysis | Auditing unverified contracts, finding hidden logic |
| Consensus mechanisms | Understanding finality, reorg risks, censorship vectors |
| L2 architecture | Sequencer trust, delayed finality, bridge interactions |
| Cross-chain bridges | Highest-value attack targets in Web3 |

> **Key Takeaway:** Every exploit ultimately reduces to unexpected EVM behavior. The more deeply you understand how the EVM processes opcodes, manages storage, and handles external calls, the more naturally you'll spot vulnerabilities during audits. Invest time in reading raw bytecode and tracing transactions at the opcode level — it separates good auditors from great ones.

---

*← [Previous: Index](./INDEX.md) | [Next: Recon & OSINT →](./RECON_AND_OSINT.md)*
