---
title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)2 — Advanced Web3 Security: ZK, ERC-4337, Cross-Chain, Restaking"
description: "Advanced Web3 security research covering formal verification with Certora and Halmos, ZK proof vulnerabilities (under-constrained circuits, field arithmetic), ERC-4337 account abstraction attacks, EIP-7702 execution model risks, intent-based solver exploitation, LayerZero/Axelar/Hyperlane cross-chain security, EigenLayer restaking risks, parallel EVM security, and AI+Web3 attack surfaces."
keywords: "ZK security, Circom vulnerability, ERC-4337 account abstraction, EIP-7702, cross-chain security, LayerZero, Axelar, formal verification, Certora, Halmos, EigenLayer restaking, parallel EVM, under-constrained ZK circuits, ZK field arithmetic errors, trusted setup compromise, intent-solver manipulation"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)2 — Advanced Web3 Security | Web3 Hacker Guide"
og_description: "Frontier Web3 security research: ZK vulnerabilities, account abstraction attacks, cross-chain message security, EigenLayer restaking risks, and AI+Web3 attack surfaces."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/12_ADVANCED_TOPICS"
schema_type: "TechArticle"
difficulty: " Expert"
module: 12
tags: [ZK-security, ERC-4337, account-abstraction, EIP-7702, cross-chain, LayerZero, formal-verification, restaking, advanced-topics]
nav_order: 12
parent: "Web3 Hacker & Pentester Guide"
---

# Module 12 — Advanced Topics

> **Difficulty:** Expert
>
> This module covers the cutting edge of Web3 security — areas where few auditors have deep expertise. Mastering these topics positions you at the frontier of blockchain security research, where the most impactful (and lucrative) findings are discovered.

---

## 12.1 Formal Verification

### What Is Formal Verification?

Formal verification uses mathematical proofs to verify that a program satisfies a specification for **all possible inputs**, not just random or chosen test cases. Unlike fuzzing (which tests many inputs) or symbolic execution (which explores paths), formal verification provides mathematical certainty.

### Tools Comparison

| Tool | Approach | Language | Strength |
|------|----------|----------|----------|
| **Certora Prover** | SMT-based model checking | CVL (Certora Verification Language) | Most mature, broadest adoption |
| **Halmos** | Symbolic bounded model checking | Solidity (Foundry tests) | Easy for Foundry users, lightweight |
| **HEVM** | Symbolic execution | Solidity (ds-test/Foundry) | Deep path exploration |
| **Kontrol (K framework)** | Rewrite-based verification | K specifications | Most powerful, steepest learning curve |

### Certora Prover — Writing Specs

```cvl
// token_spec.cvl

methods {
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function transfer(address, uint256) external returns (bool);
    function allowance(address, address) external returns (uint256) envfree;
}

// INVARIANT: Total supply equals sum of all balances
// (Certora tracks this via ghost variables)
ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

hook Sstore balances[KEY address user] uint256 newBalance (uint256 oldBalance) {
    sumBalances = sumBalances + newBalance - oldBalance;
}

invariant totalSupplyMatchesSumBalances()
    to_mathint(totalSupply()) == sumBalances;

// RULE: Transfer preserves total supply
rule transferPreservesTotalSupply(address to, uint256 amount) {
    env e;
    uint256 supplyBefore = totalSupply();
    transfer(e, to, amount);
    assert totalSupply() == supplyBefore, "Supply changed!";
}

// RULE: Transfer doesn't affect third-party balances
rule transferDoesNotAffectOthers(address to, uint256 amount, address other) {
    env e;
    require other != e.msg.sender && other != to;
    uint256 otherBefore = balanceOf(other);
    transfer(e, to, amount);
    assert balanceOf(other) == otherBefore, "Third party affected!";
}

// RULE: Nobody can decrease another's balance without approval
rule noUnauthorizedBalanceDecrease(address user) {
    env e;
    require e.msg.sender != user;

    uint256 balBefore = balanceOf(user);
    uint256 allowanceBefore = allowance(user, e.msg.sender);

    // Call any function
    calldataarg args;
    f(e, args);

    uint256 balAfter = balanceOf(user);
    assert balAfter >= balBefore ||
           (balBefore - balAfter <= allowanceBefore),
           "Unauthorized balance decrease!";
}
```

### Halmos — Symbolic Testing in Foundry

```solidity
// Halmos uses symbolic inputs with Foundry syntax
// Run: halmos --contract TokenTest

contract TokenTest is Test {
    Token token;

    function setUp() public {
        token = new Token();
    }

    // Halmos explores ALL possible values of `to` and `amount`
    function check_transferPreservesSupply(address to, uint256 amount) public {
        uint256 supplyBefore = token.totalSupply();

        // Constrain inputs to valid ranges
        vm.assume(amount <= token.balanceOf(address(this)));
        vm.assume(to != address(0));

        token.transfer(to, amount);

        assert(token.totalSupply() == supplyBefore);
    }
}
```

```bash
# Run Halmos
halmos --contract TokenTest --function check_ --solver-timeout-assertion 120
```

---

## 12.2 Zero-Knowledge Proof Vulnerabilities

### ZK Architecture Overview

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Private    │───→│   Circuit    │───→│    Proof     │
│   Inputs     │    │  (Constraint │    │  (ZK proof   │
│   (witness)  │    │   System)    │    │   output)    │
└──────────────┘    └──────────────┘    └──────────────┘
                                              │
                                              
                                        ┌──────────────┐
                                        │   Verifier   │
                                        │  (On-chain)  │
                                        │  Returns T/F │
                                        └──────────────┘
```

### Common ZK Vulnerabilities

| Vulnerability | Description | Impact |
|--------------|-------------|--------|
| **Under-constrained circuits** | Missing constraints allow invalid witnesses to generate valid proofs | Fake proofs, fund theft |
| **Over-constrained circuits** | Too many constraints reject valid inputs | DoS, locked funds |
| **Trusted setup compromise** | SNARK ceremony participant retains toxic waste | Forge arbitrary proofs |
| **Soundness errors** | Prover can convince verifier of false statements | Complete system compromise |
| **Zero-knowledge breaks** | Proof leaks information about private inputs | Privacy violation |
| **Arithmetic field errors** | Operations wrap around the field prime, not 2^256 | Unexpected behavior |
| **Nondeterminism** | Same inputs produce different proofs nondeterministically | Verification failures |

### Circom-Specific Issues

```circom
// Example: Under-constrained multiplier
template Multiplier() {
    signal input a;
    signal input b;
    signal output c;

    // [NO] Under-constrained: No constraint linking a, b, c
    // A malicious prover can set c to any value
    c <-- a * b;  // Assignment only, no constraint!
}

// [YES] Fixed: Add constraint
template MultiplierFixed() {
    signal input a;
    signal input b;
    signal output c;

    c <== a * b;  // Assignment AND constraint (a * b === c)
}
```

### ZK Field Arithmetic Pitfalls

```
In ZK circuits, arithmetic is over a prime field (e.g., BN254):
  p = 21888242871839275222246405745257275088548364400416034343698204186575808495617

This means:
- There are no negative numbers (subtraction wraps around p)
- Division is multiplication by modular inverse
- Comparison operators don't exist natively — must be built from constraints
- A value of 0 and a value of p are the same (both represent 0)

Common bug:
  "Check if x < 100" requires range proofs, not simple comparison
  Without range proof, a prover can use p-1 (which is "less than" 0 in field math)
```

---

## 12.3 ZK Circuit Auditing Basics

### Halo2 Auditing

```rust
// Halo2 circuit example — common patterns to audit
use halo2_proofs::{circuit::*, plonk::*};

impl<F: Field> Circuit<F> for MyCircuit<F> {
    fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config {
        let advice = meta.advice_column();
        let selector = meta.selector();

        meta.create_gate("multiply", |meta| {
            let s = meta.query_selector(selector);
            let a = meta.query_advice(advice, Rotation::cur());
            let b = meta.query_advice(advice, Rotation::next());
            let c = meta.query_advice(advice, Rotation(2));

            // Constraint: s * (a * b - c) == 0
            // When selector is active: a * b MUST equal c
            vec![s * (a * b - c)]
        });
        // Audit points:
        // 1. Are all necessary gates constrained?
        // 2. Is the selector activated for all required rows?
        // 3. Are rotations correct (off-by-one)?
        // 4. Are lookups properly ranged?
    }
}
```

### ZK Audit Checklist

- [ ] All intermediate signals are constrained (no `<--` without `===`)
- [ ] Range checks are applied where needed (prevent field wrapping)
- [ ] Public inputs are properly verified
- [ ] Trusted setup is valid (if SNARK-based)
- [ ] No information leakage through proof structure
- [ ] Circuit matches the specification
- [ ] Edge cases: zero values, maximum values, boundary conditions
- [ ] No nondeterministic behavior in witness generation

---

## 12.4 Account Abstraction (ERC-4337)

### Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   User       │───→│  Bundler     │───→│  EntryPoint  │
│  (dApp/SDK)  │    │  (off-chain) │    │  (on-chain)  │
│  creates     │    │  validates & │    │  executes     │
│  UserOp      │    │  bundles     │    │  UserOps      │
└──────────────┘    └──────────────┘    └──────────────┘
                                              │
                                              
                                        ┌──────────────┐
                                        │  Smart       │
                                        │  Contract    │
                                        │  Wallet      │
                                        └──────────────┘
```

### ERC-4337 Attack Surface

| Vector | Description | Impact |
|--------|-------------|--------|
| **Validation griefing** | Wasteful `validateUserOp` that passes off-chain but reverts on-chain | DoS on bundlers |
| **Signature replay** | UserOp replayed on different chain or with different EntryPoint | Fund theft |
| **Paymaster exploitation** | Draining paymaster's deposit by creating expensive-to-validate-but-failing UserOps | Paymaster fund drain |
| **Storage access violations** | Validation phase accesses forbidden storage (banned opcodes) | Bundle rejection |
| **Factory determinism** | CREATE2 wallet deployment can be front-run | Wallet takeover |
| **Module vulnerabilities** | Malicious modules in modular account implementations | Fund theft via module |

### Paymaster Attacks

```solidity
// Paymaster sponsors gas for users
// Attack: Create many UserOps that pass paymaster validation
// but fail during execution, wasting paymaster's gas deposit

interface IPaymaster {
    function validatePaymasterUserOp(
        PackedUserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData);

    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost,
        uint256 actualUserOpFeePerGas
    ) external;
}

// Security checks for paymasters:
// 1. Rate limit UserOps per sender
// 2. Require signature from trusted signer
// 3. Set per-UserOp gas limits
// 4. Validate sender reputation
```

---

## 12.5 EIP-7702 — New Execution Model

### What EIP-7702 Changes

EIP-7702 allows EOAs to temporarily delegate to smart contract code per-transaction, blurring the line between EOAs and contract accounts.

```
Before EIP-7702:
  EOA → can only send simple transactions
  Contract Account → has code logic

After EIP-7702:
  EOA → can authorize a contract's code to execute on its behalf
       for the duration of a transaction
```

### Security Implications

| Risk | Description |
|------|-------------|
| **Delegation to malicious code** | User authorizes a contract that drains their funds |
| **Authorization replay** | Signed authorization replayed on another chain |
| **Interaction with existing contracts** | Contracts checking `msg.sender.code.length == 0` for EOA detection break |
| **Revocation complexity** | How to revoke delegation once authorized? |
| **Phishing amplification** | Easier to trick users into signing dangerous authorizations |

---

## 12.6 Intent-Based Systems & Solver Manipulation

### How Intent Systems Work

```
Traditional: User creates exact transaction → submits to mempool
Intent-based: User describes desired outcome → solver finds optimal execution

┌──────────┐    ┌──────────┐    ┌──────────┐
│   User   │───→│  Intent  │───→│  Solver  │
│ "Swap X  │    │  Pool    │    │  Finds   │
│  for Y   │    │          │    │  best    │
│  at best │    │          │    │  path    │
│  price"  │    │          │    │          │
└──────────┘    └──────────┘    └──────────┘
```

### Solver Attack Vectors

| Attack | Description |
|--------|-------------|
| **Solver collusion** | Solvers coordinate to offer worse execution | 
| **Preferential execution** | Solver front-runs intent or delays execution to benefit themselves |
| **Intent interpretation** | Ambiguous intents exploited by solvers |
| **Order flow capture** | Solvers capture valuable order flow, create oligopoly |
| **Censorship** | Solvers selectively refuse to fill certain intents |

---

## 12.7 Cross-Chain Message Security

### LayerZero

```
Architecture:
  Source Chain → Oracle + Relayer → Destination Chain

Security model:
- Oracle (Chainlink/custom) provides block headers
- Relayer provides transaction proofs
- APPLICATION can configure oracle and relayer independently

Attack vectors:
- Oracle and relayer collusion (if both are compromised)
- Application misconfiguration (wrong oracle/relayer)
- Message replay across LayerZero deployments
- Gas estimation attacks (insufficient gas on destination)
```

### Axelar

```
Architecture:
  Source Chain → Axelar Network (PoS validators) → Destination Chain

Security model:
- Validator set secured by staked AXL tokens
- Threshold signature scheme (multisig-like)

Attack vectors:
- Validator collusion (if threshold compromised)
- Cross-chain message forgery (if validation bypassed)
- Gas payment manipulation on destination chain
```

### Hyperlane

```
Architecture:
  Source Chain → ISM (Interchain Security Module) → Destination Chain

Security model:
- Modular: applications choose their own security model
- ISM options: Multisig, Optimistic, ZK, etc.

Attack vectors:
- ISM misconfiguration (too-low threshold)
- Interoperability between different ISM types
- Destination chain gas griefing
```

### Cross-Chain Security Audit Checklist

- [ ] Message validation: How are cross-chain messages verified?
- [ ] Replay protection: Can the same message be processed twice?
- [ ] Gas estimation: What happens if destination execution runs out of gas?
- [ ] Timeout/expiry: What happens if a message is never delivered?
- [ ] Ordering: Are messages processed in order? Does order matter?
- [ ] Access control: Who can send cross-chain messages?
- [ ] Refund mechanism: What happens on failure?

---

## 12.8 Restaking Protocol Risks (EigenLayer)

### Architecture

```
┌─────────────────────────────────────────────┐
│              EigenLayer                      │
│  ┌────────────┐  ┌────────────────────────┐ │
│  │  Stakers   │  │  AVS (Actively         │ │
│  │  Deposit   │──│  Validated Services)   │ │
│  │  ETH/LSTs  │  │  Use restaked security │ │
│  └────────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### Restaking Risk Surface

| Risk | Description | Severity |
|------|-------------|----------|
| **Cascading slashing** | Restaked ETH slashed across multiple AVSes simultaneously | Critical |
| **Operator centralization** | Few operators handle majority of restaked ETH | High |
| **Smart contract risk** | Bug in EigenLayer or AVS contracts drains restaked ETH | Critical |
| **Withdrawal delay exploitation** | Attackers exploit the unbonding period for attacks | Medium |
| **AVS validation bugs** | Incorrect validation logic in AVS → unfair slashing | High |
| **Economic attacks** | Cost-of-corruption vs revenue analysis across AVSes | Variable |

---

## 12.9 Parallel EVM (Monad, Sei)

### What Changes with Parallel Execution

```
Sequential EVM:
  TX 1 → TX 2 → TX 3 → TX 4 (one after another)

Parallel EVM:
  TX 1 ─┐
  TX 2 ─┤─→ Executed simultaneously if no conflicts
  TX 3 ─┤
  TX 4 ─┘

Conflict detection: If TX 1 and TX 3 touch the same storage slot,
they must be serialized. Otherwise, they can run in parallel.
```

### Security Implications

| Issue | Description |
|-------|-------------|
| **State access conflicts** | Transactions accessing same storage may have nondeterministic ordering |
| **MEV in parallel context** | Ordering within parallel batches creates new MEV opportunities |
| **Reentrancy across parallel txs** | Concurrent state modifications may violate invariants |
| **Different gas semantics** | Parallel execution may change effective gas costs |
| **Cross-transaction dependencies** | Assumptions about transaction ordering may break |

---

## 12.10 AI Agent + Web3 Interaction Surface

### Emerging Attack Surface

As AI agents increasingly interact with Web3 systems (autonomous trading, portfolio management, DAO participation), new attack vectors emerge:

| Vector | Description |
|--------|-------------|
| **Prompt injection via on-chain data** | Malicious contract names, token symbols, or metadata that manipulate AI decision-making |
| **Agent key management** | AI agents holding private keys — single point of compromise |
| **Adversarial inputs** | Crafted on-chain states designed to trigger specific AI behaviors |
| **Oracle manipulation for AI** | Manipulating data feeds that AI agents rely on for decisions |
| **Social engineering AI** | Governance proposals or forum posts crafted to influence AI-driven voting |
| **Autonomous agent rug pulls** | AI agents with treasury access making unauthorized transfers |

### Defense Considerations

```
1. AI agents should operate with MINIMUM necessary permissions
2. Multi-sig or time-lock on all fund movements
3. Human-in-the-loop for transactions above threshold
4. Input sanitization for all on-chain data fed to AI
5. Monitoring and circuit breakers for autonomous agents
6. Separation of hot wallet (operations) and cold wallet (treasury)
```

---

## Summary — Frontier Research Areas

| Topic | Maturity | Audit Demand | Learning Priority |
|-------|----------|-------------|-------------------|
| Formal verification | Mature | High (for critical protocols) | High |
| ZK vulnerabilities | Growing | Rapidly increasing | Critical |
| ERC-4337 | Maturing | Moderate | High |
| EIP-7702 | Early | Emerging | Medium |
| Intent systems | Early | Emerging | Medium |
| Cross-chain messaging | Mature | High | High |
| Restaking | Growing | High | High |
| Parallel EVM | Early | Low (for now) | Low-Medium |
| AI + Web3 | Very early | Emerging | Low-Medium |

> **Key Takeaway:** The highest-paying security work is at the frontier. ZK circuit auditing, cross-chain message security, and account abstraction are areas where demand far exceeds supply of qualified auditors. If you can audit Circom circuits or write Certora specs for complex DeFi protocols, you're in the top 1% of Web3 security researchers. Invest in these skills now — they'll compound as these technologies mature and handle billions of dollars.

---

*← [Previous: Reporting & Disclosure](./REPORTING_AND_RESPONSIBLE_DISCLOSURE.md) | [Back to Index →](./INDEX.md)*


<script>
document.addEventListener("DOMContentLoaded", function() {
  const checkboxes = document.querySelectorAll('.task-list-item-checkbox, input[type="checkbox"]');
  checkboxes.forEach(function(cb) {
    cb.removeAttribute('disabled');
    cb.style.cursor = 'pointer';
  });
});
</script>
