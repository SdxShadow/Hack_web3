---
title: "Module 03 — Smart Contract Vulnerabilities: Complete Reference"
description: "Comprehensive reference for all smart contract vulnerability classes: reentrancy, integer overflow, access control flaws, flash loan attacks, oracle manipulation, delegatecall exploits, front-running, signature replay, DoS vectors, and more — each with vulnerable code, exploit PoC, fixed code, and real-world exploit references."
keywords: "smart contract vulnerabilities, reentrancy exploit, flash loan attack, oracle manipulation, delegatecall vulnerability, signature replay, access control flaw, integer overflow, DeFi exploits, Solidity security, EVM vulnerabilities, uninitialized proxy bug, cross-function reentrancy, read-only reentrancy, TWAP manipulation cost, AMM sandwiching, AMM front-running attacks, flashmint exploits"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 03 — Smart Contract Vulnerabilities | Web3 Hacker Guide"
og_description: "Complete vulnerability reference with PoC exploits: reentrancy, flash loans, oracles, access control, signature replay, DoS, proxy exploits, and 20+ more vulnerability classes."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/03_SMART_CONTRACT_VULNERABILITIES"
schema_type: "TechArticle"
difficulty: " Intermediate →  Advanced"
module: 3
tags: [vulnerabilities, reentrancy, flash-loans, oracles, access-control, solidity, DeFi-security, exploits, PoC]
nav_order: 3
parent: "Web3 Hacker & Pentester Guide"
---

# Module 03 — Smart Contract Vulnerabilities

> **Difficulty:** Intermediate →  Advanced
>
> This is the core module. Every vulnerability class known to affect smart contracts is documented here with: technical explanation, vulnerable code, fixed code, real-world exploit reference, and detection method. This is your primary reference during audits.

---

## 3.1 Reentrancy

### Difficulty: Intermediate

Reentrancy occurs when a contract makes an external call before updating its state, allowing the called contract to re-enter the original function and exploit the stale state.

### Types of Reentrancy

| Type | Description | Difficulty |
|------|-------------|------------|
| **Single-function** | Re-entering the same function |  Beginner |
| **Cross-function** | Re-entering a different function that shares state |  Intermediate |
| **Cross-contract** | Re-entering via a different contract in the same protocol |  Advanced |
| **Read-only reentrancy** | Re-entering a view function that returns stale state used by another protocol |  Advanced |

### Vulnerable Code — Single-Function Reentrancy

```solidity
// VULNERABLE: State update AFTER external call
contract VulnerableBank {
    mapping(address => uint256) public balances;

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // [NO] External call BEFORE state update
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        // State update happens AFTER the call
        // If msg.sender is a contract, its receive() can call withdraw() again
        balances[msg.sender] = 0;
    }

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
}
```

### Attack Contract

```solidity
contract ReentrancyAttacker {
    VulnerableBank public target;

    constructor(address _target) {
        target = VulnerableBank(_target);
    }

    function attack() external payable {
        target.deposit{value: msg.value}();
        target.withdraw();
    }

    receive() external payable {
        if (address(target).balance >= target.balances(address(this))) {
            target.withdraw(); // Re-enter!
        }
    }
}
```

### Fixed Code

```solidity
// FIXED: Checks-Effects-Interactions pattern + reentrancy guard
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract SecureBank is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function withdraw() external nonReentrant {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // [YES] Effect BEFORE interaction
        balances[msg.sender] = 0;

        // Interaction AFTER effect
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
    }
}
```

### Read-Only Reentrancy

This subtle variant doesn't steal from the re-entered contract directly. Instead, the attacker re-enters a **view function** that returns stale state, and a **different protocol** reads that stale value.

```solidity
// Protocol A — Vulnerable to read-only reentrancy
contract CurvePool {
    // This view function reads balances that haven't been updated yet
    // during the callback in remove_liquidity
    function get_virtual_price() external view returns (uint256) {
        // Returns price based on current balances
        // During reentrancy, balances are stale → price is manipulated
        return _calculate_price();
    }

    function remove_liquidity(uint256 _amount) external {
        // Burns LP tokens → sends ETH → THEN updates balances
        // During the ETH send, get_virtual_price() returns stale value
        _burn(msg.sender, _amount);
        payable(msg.sender).call{value: ethAmount}(""); // Callback here!
        _update_balances(); // Too late — attacker already read stale price
    }
}

// Protocol B — Relies on Protocol A's view function
contract LendingProtocol {
    function getCollateralValue(address user) public view returns (uint256) {
        uint256 lpBalance = lpToken.balanceOf(user);
        uint256 price = curvePool.get_virtual_price(); // Reads stale value!
        return lpBalance * price / 1e18;
    }
}
```

**Real-world exploit:** Curve/Vyper reentrancy (July 2023) — Multiple Curve pools exploited through reentrancy in Vyper 0.2.15-0.3.0, with the compiler's reentrancy lock being buggy. ~$70M lost across multiple pools.

### Detection Methods

- **Slither**: `slither . --detect reentrancy-eth,reentrancy-no-eth,reentrancy-benign`
- **Manual**: Look for external calls before state updates (CEI violations)
- **Echidna**: Write properties asserting that balances are consistent
- **Pattern**: Any `call`, `transfer`, `send`, or ERC-777 `tokensReceived` callback before state update

---

## 3.2 Integer Overflow / Underflow

### Difficulty: Beginner

Prior to Solidity 0.8.0, arithmetic operations silently wrapped on overflow/underflow. Post-0.8.0, they revert by default — but `unchecked {}` blocks and inline assembly bypass this.

### Vulnerable Code (Pre-0.8.x)

```solidity
// Solidity < 0.8.0 — No automatic overflow checks
pragma solidity 0.7.6;

contract VulnerableToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        // [NO] If balances[msg.sender] < amount, this UNDERFLOWS to a huge number
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

    function batchTransfer(address[] memory recipients, uint256 value) external {
        // [NO] If recipients.length * value overflows, totalAmount becomes small
        uint256 totalAmount = recipients.length * value;
        require(balances[msg.sender] >= totalAmount);
        balances[msg.sender] -= totalAmount;
        for (uint i = 0; i < recipients.length; i++) {
            balances[recipients[i]] += value;
        }
    }
}
```

### Fixed Code

```solidity
// FIXED: Use Solidity >= 0.8.0 (automatic checks) or SafeMath for older versions
pragma solidity ^0.8.20;

contract SafeToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        // [YES] Reverts on underflow in Solidity 0.8+
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}

// For older versions:
// import "@openzeppelin/contracts/utils/math/SafeMath.sol";
// using SafeMath for uint256;
// balances[msg.sender] = balances[msg.sender].sub(amount);
```

### Post-0.8.x Bypass — `unchecked` Blocks

```solidity
// Watch out for developers using unchecked for "gas optimization"
function riskyDecrement(uint256 x) internal pure returns (uint256) {
    unchecked {
        return x - 1; // [NO] Wraps if x == 0!
    }
}
```

**Real-world exploit:** BEC Token (April 2018) — `batchTransfer` function had an integer overflow allowing attackers to create tokens from nothing. The `recipients.length * value` multiplication overflowed, bypassing the balance check.

### Detection Methods

- **Slither**: Detects some overflow patterns in pre-0.8.x code
- **Manual**: Search for `unchecked` blocks, inline assembly arithmetic, and pre-0.8.x contracts
- **Mythril**: Symbolic execution can find overflow conditions
- **Pattern**: `unchecked { x - y }` where `y > x` is possible

---

## 3.3 Access Control Flaws

### Difficulty: Beginner

Missing or improper access control is the most common vulnerability class in competitive audits. It ranges from missing `onlyOwner` modifiers to subtle role-based access control logic errors.

### Vulnerable Code — Missing Modifier

```solidity
contract VulnerableVault {
    address public owner;
    uint256 public feeRate;

    constructor() { owner = msg.sender; }

    // [NO] No access control — anyone can change the fee rate!
    function setFeeRate(uint256 _newRate) external {
        feeRate = _newRate;
    }

    // [NO] No access control — anyone can steal all funds!
    function emergencyWithdraw(address to) external {
        payable(to).transfer(address(this).balance);
    }
}
```

### Vulnerable Code — tx.origin Authentication

```solidity
contract VulnerableWallet {
    address public owner;

    // [NO] tx.origin checks the original transaction sender, not the immediate caller
    // An attacker can trick the owner into calling a malicious contract that then
    // calls this function — tx.origin will still be the owner!
    function transfer(address to, uint256 amount) external {
        require(tx.origin == owner, "Not owner"); // [NO] Use msg.sender!
        payable(to).transfer(amount);
    }
}
```

### Fixed Code

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

contract SecureVault is AccessControl {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant FEE_MANAGER_ROLE = keccak256("FEE_MANAGER_ROLE");

    uint256 public feeRate;

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(FEE_MANAGER_ROLE, msg.sender);
    }

    // [YES] Proper role-based access control
    function setFeeRate(uint256 _newRate) external onlyRole(FEE_MANAGER_ROLE) {
        require(_newRate <= 1000, "Fee too high"); // Max 10%
        feeRate = _newRate;
    }

    function emergencyWithdraw(address to) external onlyRole(ADMIN_ROLE) {
        payable(to).transfer(address(this).balance);
    }
}
```

**Real-world exploit:** Parity Multisig Wallet (Nov 2017) — An unprotected `initWallet()` function allowed anyone to take ownership of the library contract and then self-destruct it, freezing ~$150M across 587 wallets.

### Detection Methods
- **Slither**: `slither . --detect unprotected-upgrade,suicidal,arbitrary-send-eth`
- **Manual**: Audit every `external`/`public` function for missing access control
- **Pattern**: Search for state-changing functions without modifiers

---

## 3.4 Front-Running / MEV

### Difficulty: Intermediate

Front-running exploits the public mempool: an attacker observes a pending transaction and submits their own transaction with a higher gas price to execute first.

### Types of Front-Running

| Type | Mechanism | Example |
|------|-----------|---------|
| **Displacement** | Attacker's tx replaces victim's | Front-running a name registration |
| **Insertion (Sandwich)** | Attacker places txs before AND after victim | DEX sandwich attacks |
| **Suppression** | Attacker fills blocks to delay victim's tx | Delaying liquidations |

### Vulnerable Code — Front-Runnable Approval

```solidity
// [NO] ERC20 approve() is front-runnable
// If Alice approves Bob from 100 to 50 tokens:
// 1. Bob sees the pending approve(50) tx in mempool
// 2. Bob front-runs with transferFrom(Alice, Bob, 100) — uses old allowance
// 3. Alice's approve(50) executes
// 4. Bob calls transferFrom(Alice, Bob, 50) — uses new allowance
// Result: Bob steals 150 instead of 50

// FIXED: Use increaseAllowance/decreaseAllowance or set to 0 first
function safeApprove(IERC20 token, address spender, uint256 amount) internal {
    token.approve(spender, 0); // [YES] Set to 0 first
    token.approve(spender, amount);
}
```

### Sandwich Attack on DEX

```solidity
// Attacker monitors mempool for large swap:
// Victim: swap 100 ETH → USDC on Uniswap

// Step 1: Attacker FRONT-RUNS — buys USDC before victim
// This raises the USDC price

// Step 2: Victim's swap executes at WORSE price

// Step 3: Attacker BACK-RUNS — sells USDC after victim
// Locks in profit from the price impact

// Defense: Use Flashbots Protect, set tight slippage, deadline parameter
```

**Real-world impact:** MEV bots extract millions daily. In 2023, over $1.4 billion was extracted via MEV across Ethereum alone.

### Detection Methods
- **Manual**: Look for transactions vulnerable to ordering-dependence
- **Pattern**: Approval changes, oracle updates, large swaps without slippage protection

---

## 3.5 Flash Loan Attacks

### Difficulty: Advanced

Flash loans allow borrowing millions in tokens with zero collateral — repayment happens in the same transaction. They enable attackers to temporarily wield massive capital for price manipulation, governance attacks, and collateral manipulation.

### Attack Anatomy

```
Single Transaction:
1. Borrow $100M via flash loan (Aave, dYdX, etc.)
2. Use funds to manipulate price oracle or governance
3. Exploit the manipulated state for profit
4. Repay flash loan + fee
5. Keep profit

Total cost: Gas fee + flash loan fee (~0.05%)
```

### Vulnerable Code — Flash Loan Oracle Manipulation

```solidity
contract VulnerableLending {
    IUniswapV2Pair public priceFeed;
    IERC20 public collateralToken;
    IERC20 public borrowToken;

    // [NO] Uses spot price from Uniswap — manipulable via flash loan
    function getPrice() public view returns (uint256) {
        (uint112 reserve0, uint112 reserve1, ) = priceFeed.getReserves();
        return (uint256(reserve1) * 1e18) / uint256(reserve0);
    }

    function borrow(uint256 collateralAmount, uint256 borrowAmount) external {
        collateralToken.transferFrom(msg.sender, address(this), collateralAmount);
        uint256 collateralValue = collateralAmount * getPrice() / 1e18;
        require(collateralValue >= borrowAmount * 150 / 100, "Undercollateralized");
        borrowToken.transfer(msg.sender, borrowAmount);
    }
}
```

### Fixed Code

```solidity
contract SecureLending {
    AggregatorV3Interface public chainlinkOracle;

    // [YES] Uses Chainlink price feed — not manipulable by flash loans
    function getPrice() public view returns (uint256) {
        (, int256 price,, uint256 updatedAt,) = chainlinkOracle.latestRoundData();
        require(price > 0, "Invalid price");
        require(block.timestamp - updatedAt < 3600, "Stale price"); // 1 hour
        return uint256(price);
    }
}
```

**Real-world exploit:** Beanstalk (April 2022) — $182M stolen. Attacker took a flash loan, acquired enough governance tokens to pass a malicious proposal that drained the treasury, and repaid the loan — all in one transaction.

### Detection Methods
- **Manual**: Identify oracle dependencies — are they spot prices or TWAPs?
- **Pattern**: Any protocol that reads prices from AMM reserves without TWAP
- **Echidna/Foundry**: Fuzz with flash-loan-sized inputs

---

## 3.6 Oracle Manipulation

### Difficulty: Advanced

Oracles feed external or cross-contract data into smart contracts. Manipulating this data is one of the most profitable exploit categories.

### Oracle Types & Risks

| Oracle Type | Manipulation Risk | Cost to Manipulate |
|------------|------------------|-------------------|
| **Spot price (AMM reserves)** | Very High | Flash loan (near-zero) |
| **TWAP (time-weighted average)** | Medium | Sustained manipulation over time blocks |
| **Chainlink** | Low | Would require compromising majority of nodes |
| **Band Protocol** | Low-Medium | Similar decentralized oracle network |
| **Custom oracle (centralized)** | Very High | Compromise the oracle operator |

### Vulnerable Code — Spot Price Oracle

```solidity
// [NO] Using Uniswap spot reserves as price oracle
function getTokenPrice() external view returns (uint256) {
    (uint112 r0, uint112 r1, ) = uniswapPair.getReserves();
    return uint256(r1) * 1e18 / uint256(r0);
    // Manipulable: a single large swap changes reserves instantly
}
```

### Fixed Code — Chainlink with Staleness Check

```solidity
// [YES] Using Chainlink with proper validation
function getTokenPrice() external view returns (uint256) {
    (
        uint80 roundId,
        int256 price,
        ,
        uint256 updatedAt,
        uint80 answeredInRound
    ) = priceFeed.latestRoundData();

    require(price > 0, "Negative price");
    require(updatedAt > 0, "Round not complete");
    require(answeredInRound >= roundId, "Stale price");
    require(block.timestamp - updatedAt < 3600, "Price too old");

    return uint256(price);
}
```

### Chainlink Gotchas

| Issue | Description |
|-------|-------------|
| **Staleness** | Price hasn't been updated in too long (oracle may be down) |
| **L2 Sequencer down** | On L2s, check sequencer uptime before using Chainlink prices |
| **Deviation threshold** | Chainlink only updates if price deviates >X% — stale in low-volatility |
| **minAnswer/maxAnswer** | Some feeds have circuit breakers — price clamps at min/max |

**Real-world exploit:** Mango Markets (Oct 2022) — $114M. Attacker manipulated the price of MNGO token on a spot oracle by inflating it through coordinated trading, then borrowed against inflated collateral.

---

## 3.7 Delegatecall Vulnerabilities

### Difficulty: Advanced

`DELEGATECALL` executes the callee's code in the context of the caller — the caller's storage, `msg.sender`, and `msg.value` are preserved. This is the foundation of proxy patterns, but misuse leads to devastating exploits.

### Vulnerable Code — Storage Collision

```solidity
// Proxy contract
contract Proxy {
    address public implementation;  // Storage slot 0
    address public owner;           // Storage slot 1

    function upgrade(address _impl) external {
        require(msg.sender == owner);
        implementation = _impl;
    }

    fallback() external payable {
        (bool s, ) = implementation.delegatecall(msg.data);
        require(s);
    }
}

// Implementation contract
contract Implementation {
    // [NO] STORAGE COLLISION: 'admin' occupies slot 0 — same as 'implementation' in Proxy!
    address public admin;        // Slot 0 — overwrites proxy.implementation!
    uint256 public value;        // Slot 1 — overwrites proxy.owner!

    function setAdmin(address _admin) external {
        admin = _admin;  // Actually writes to proxy.implementation → breaks proxy!
    }
}
```

### Fixed Code — EIP-1967 Storage Slots

```solidity
// [YES] Fixed: Use EIP-1967 random storage slots to avoid collision
contract SecureProxy {
    // Slot = keccak256("eip1967.proxy.implementation") - 1
    bytes32 private constant IMPL_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    bytes32 private constant ADMIN_SLOT =
        0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;

    function _setImplementation(address impl) internal {
        assembly { sstore(IMPL_SLOT, impl) }
    }

    function _getImplementation() internal view returns (address impl) {
        assembly { impl := sload(IMPL_SLOT) }
    }
}
```

**Real-world exploit:** Parity Wallet Hack #1 (July 2017) — The initialization function of a library contract was callable by anyone via `delegatecall`, allowing the attacker to take ownership and drain ~$30M.

---

## 3.8 Uninitialized Storage Pointers

### Difficulty: Intermediate

In older Solidity versions (< 0.5.0), local variables of struct type defaulted to storage — creating a pointer to slot 0, which could overwrite critical state.

### Vulnerable Code

```solidity
pragma solidity 0.4.25;

contract UninitializedStorage {
    address public owner;     // Slot 0
    uint256 public totalDeposit;  // Slot 1

    struct User {
        address addr;         // Would point to slot 0
        uint256 balance;      // Would point to slot 1
    }

    mapping(uint256 => User) public users;

    function register(uint256 id) external {
        // [NO] In Solidity < 0.5.0, this creates a storage pointer to slot 0!
        User user;
        user.addr = msg.sender;     // Overwrites owner (slot 0)!
        user.balance = 0;           // Overwrites totalDeposit (slot 1)!
        users[id] = user;
    }
}
```

### Fixed Code

```solidity
pragma solidity ^0.8.20;

contract FixedStorage {
    address public owner;
    uint256 public totalDeposit;

    struct User {
        address addr;
        uint256 balance;
    }

    mapping(uint256 => User) public users;

    function register(uint256 id) external {
        // [YES] In 0.8+, explicit storage/memory keyword is required
        User memory user = User({addr: msg.sender, balance: 0});
        users[id] = user;
    }
}
```

### Detection Methods
- **Slither**: `slither . --detect uninitialized-storage`
- **Manual**: Look for local struct variables without explicit `memory` keyword in pre-0.5.0 code

---

## 3.9 Signature Replay Attacks

### Difficulty: Intermediate

Off-chain signatures enable gasless transactions and meta-transactions. Without proper replay protection, a valid signature can be reused.

### Vulnerable Code

```solidity
contract VulnerableRelay {
    mapping(address => uint256) public balances;

    // [NO] No replay protection! Same signature can be submitted multiple times
    function withdraw(uint256 amount, bytes memory signature) external {
        bytes32 hash = keccak256(abi.encodePacked(amount));
        address signer = ECDSA.recover(hash, signature);

        require(balances[signer] >= amount, "Insufficient balance");
        balances[signer] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

### Fixed Code — With EIP-712

```solidity
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract SecureRelay is EIP712 {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public nonces;  // [YES] Replay protection

    bytes32 public constant WITHDRAW_TYPEHASH =
        keccak256("Withdraw(address owner,uint256 amount,uint256 nonce,uint256 deadline)");

    constructor() EIP712("SecureRelay", "1") {}

    function withdraw(
        uint256 amount,
        uint256 deadline,
        uint8 v, bytes32 r, bytes32 s
    ) external {
        require(block.timestamp <= deadline, "Expired"); // [YES] Deadline

        bytes32 structHash = keccak256(abi.encode(
            WITHDRAW_TYPEHASH,
            msg.sender,
            amount,
            nonces[msg.sender]++, // [YES] Nonce incremented
            deadline
        ));

        bytes32 hash = _hashTypedDataV4(structHash); // [YES] Includes chainId
        address signer = ECDSA.recover(hash, v, r, s);

        require(signer == msg.sender, "Invalid signature");
        require(balances[signer] >= amount, "Insufficient");
        balances[signer] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

### Replay Attack Vectors

| Missing Protection | Attack |
|-------------------|--------|
| No nonce | Replay same signature multiple times |
| No chainId | Replay signature on different chains (mainnet sig used on Polygon) |
| No deadline | Signature valid forever — attacker can submit later when conditions favor them |
| No contract address | Signature meant for Contract A replayed on Contract B |

**Real-world exploit:** Wintermute Op token theft (June 2022) — 20M OP tokens were stolen because the Gnosis Safe deployment process on Optimism allowed replay of mainnet deployment signatures.

---

## 3.10 Unsafe External Calls

### Difficulty: Beginner

Low-level calls (`call`, `delegatecall`, `staticcall`) return a boolean success indicator. Failing to check this return value means silent failures.

### Vulnerable Code

```solidity
contract UnsafeCaller {
    // [NO] Return value of call is not checked
    function sendETH(address to, uint256 amount) external {
        payable(to).call{value: amount}(""); // May fail silently!
    }

    // [NO] Return value of ERC20 transfer not checked
    function sendToken(IERC20 token, address to, uint256 amount) external {
        token.transfer(to, amount); // Some tokens return false instead of reverting
    }
}
```

### Fixed Code

```solidity
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract SafeCaller {
    using SafeERC20 for IERC20;

    // [YES] Check return value
    function sendETH(address to, uint256 amount) external {
        (bool success, ) = payable(to).call{value: amount}("");
        require(success, "ETH transfer failed");
    }

    // [YES] Use SafeERC20 for tokens
    function sendToken(IERC20 token, address to, uint256 amount) external {
        token.safeTransfer(to, amount); // Reverts on failure
    }
}
```

### Detection Methods
- **Slither**: `slither . --detect unchecked-lowlevel,unchecked-transfer`
- **Manual**: Search for `.call{`, `.transfer(`, `.send(` without return value check

---

## 3.11 Denial of Service (DoS)

### Difficulty: Intermediate

DoS attacks prevent legitimate users from interacting with a contract. Unlike traditional web DoS, on-chain DoS can be permanent.

### Types of DoS

| Type | Mechanism | Permanence |
|------|-----------|------------|
| **Gas griefing** | Force expensive operations in callbacks | Per-transaction |
| **Unbounded loops** | Iterate over growing arrays until gas limit exceeded | Permanent (as array grows) |
| **Forced ETH via selfdestruct** | Break `address(this).balance == 0` checks | Permanent |
| **Push vs Pull** | Send loop fails for one recipient → blocks all | Permanent (until removed) |
| **Block gas limit** | Array too large: no single tx can process it | Permanent |

### Vulnerable Code — Unbounded Loop

```solidity
contract VulnerableAirdrop {
    address[] public recipients;

    function addRecipient(address r) external {
        recipients.push(r);
    }

    // [NO] Loops over unbounded array — will exceed gas limit as array grows
    function distribute(IERC20 token, uint256 amountEach) external {
        for (uint i = 0; i < recipients.length; i++) {
            token.transfer(recipients[i], amountEach); // What if one reverts?
        }
    }
}
```

### Fixed Code — Pull Pattern + Batched Processing

```solidity
contract SecureDistribution {
    mapping(address => uint256) public claimable;

    // [YES] Push: Admin sets claimable amounts (bounded by admin's gas budget)
    function setClaimable(address[] calldata users, uint256[] calldata amounts) external onlyOwner {
        require(users.length == amounts.length);
        require(users.length <= 100, "Batch too large"); // [YES] Bounded
        for (uint i = 0; i < users.length; i++) {
            claimable[users[i]] += amounts[i];
        }
    }

    // [YES] Pull: Users claim their own tokens
    function claim(IERC20 token) external {
        uint256 amount = claimable[msg.sender];
        require(amount > 0, "Nothing to claim");
        claimable[msg.sender] = 0;
        token.safeTransfer(msg.sender, amount);
    }
}
```

### Force-Feeding ETH

```solidity
// [NO] This check can be broken by selfdestruct force-feeding ETH
contract VulnerableGame {
    function isGameOver() public view returns (bool) {
        return address(this).balance == 7 ether; // Can be bypassed!
    }
}

// Attacker can force-send ETH via:
// 1. selfdestruct — sends remaining balance to target (pre-Dencun only in same tx)
// 2. Mining reward / validator reward (set target as coinbase)
// 3. Pre-calculated CREATE2 address — send ETH before contract deployment
```

**Real-world exploit:** GovernMental Ponzi (2016) — A growing array of investors eventually made the `withdraw()` function exceed the block gas limit, permanently locking 1,100 ETH.

---

## 3.12 Self-Destruct / Force-Feeding ETH

### Difficulty: Beginner

`SELFDESTRUCT` destroys a contract and forcefully sends its remaining ETH to a target address, bypassing any `receive()` or `fallback()` function. Post-Dencun (EIP-6780), `SELFDESTRUCT` only works within the same transaction as contract creation.

```solidity
// Attacker contract (pre-Dencun):
contract ForceFeeder {
    constructor(address payable target) payable {
        selfdestruct(target); // [YES] Still works in same tx as creation (EIP-6780)
    }
}

// Now ETH can be forcefully sent even post-Dencun by creating + destructing in one tx
```

### What Breaks
- Any invariant checking `address(this).balance == X`
- Games relying on exact balance thresholds
- Contracts that assume no ETH can arrive without going through `receive()`/`fallback()`

---

## 3.13 Timestamp Dependence

### Difficulty: Beginner

`block.timestamp` is set by the block proposer and can be manipulated within the ~12-second slot window on PoS Ethereum.

### Vulnerable Code

```solidity
// [NO] Using timestamp for randomness or precise timing
contract TimestampVulnerable {
    function isWinner() external view returns (bool) {
        return block.timestamp % 15 == 0; // Manipulable!
    }

    function isExpired(uint256 deadline) external view returns (bool) {
        // ! Validator can slightly manipulate this
        // Acceptable for deadlines with large windows (hours/days)
        // Not acceptable for precise sub-minute timing
        return block.timestamp >= deadline;
    }
}
```

### Guidance
- **Acceptable**: Using `block.timestamp` for deadlines with windows > 15 minutes
- **Dangerous**: Using `block.timestamp` for randomness, exact-second timing, or lotteries
- **Note**: On PoS, timestamps are more constrained (must be exactly `12 * slotNumber + genesisTime`)

---

## 3.14 Block Number Dependence

### Difficulty: Beginner

Using `block.number` for time estimation is unreliable across chains with different block times (L2s often have variable block times).

```solidity
// [NO] Assuming 12-second block time for duration estimation
uint256 public constant BLOCKS_PER_DAY = 7200; // Only valid on mainnet post-Merge

// On Arbitrum, Optimism, or other L2s, block time varies significantly
// This could be off by hours or days
```

---

## 3.15 Randomness Manipulation

### Difficulty: Intermediate

On-chain randomness is a fundamental challenge. Block variables are predictable to miners/validators.

### Vulnerable Code

```solidity
// [NO] Blockhash-based randomness — predictable to validators
function getRandomNumber() external view returns (uint256) {
    return uint256(keccak256(abi.encodePacked(
        block.timestamp,
        block.prevrandao,     // PREVRANDAO replaces DIFFICULTY post-Merge
        msg.sender
    )));
}
// Validators know block.prevrandao and block.timestamp before block production
// They can reorder txs or skip proposing to influence the outcome
```

### Secure Randomness

```solidity
// [YES] Use Chainlink VRF (Verifiable Random Function)
import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBaseV2.sol";

contract SecureRandom is VRFConsumerBaseV2 {
    function requestRandom() external returns (uint256 requestId) {
        requestId = COORDINATOR.requestRandomWords(
            keyHash,
            subscriptionId,
            requestConfirmations,
            callbackGasLimit,
            numWords
        );
    }

    function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
        // Use randomWords[0] — cryptographically secure and verifiable
    }
}
```

**Note**: `PREVRANDAO` (post-Merge) provides 256 bits of "randomness," but the proposer knows the value before proposing the block and can choose not to propose (1-bit bias attack). It's acceptable for low-value outcomes but not for high-value lotteries.

---

## 3.16 Precision Loss & Rounding Errors

### Difficulty: Intermediate

Solidity has no floating-point arithmetic. All math uses integer division, which truncates (rounds toward zero). This creates exploitable rounding errors, especially in DeFi.

### Vulnerable Code

```solidity
// [NO] Division before multiplication — precision loss
function calculateFee(uint256 amount, uint256 feePercent) external pure returns (uint256) {
    return amount / 100 * feePercent; // If amount = 99, result = 0 regardless of fee!
}

// [NO] Small deposits can round to 0 shares
function deposit(uint256 assets) external returns (uint256 shares) {
    shares = assets * totalSupply / totalAssets; // If totalAssets >> totalSupply * assets, shares = 0
    // User deposits tokens but gets 0 shares — funds lost
}
```

### Fixed Code

```solidity
// [YES] Multiply before divide
function calculateFee(uint256 amount, uint256 feePercent) external pure returns (uint256) {
    return amount * feePercent / 100; // 99 * 5 / 100 = 4 (correct)
}

// [YES] Add minimum share check + round up for deposits
function deposit(uint256 assets) external returns (uint256 shares) {
    shares = assets * totalSupply / totalAssets;
    require(shares > 0, "Deposit too small");
}

// [YES] Round up against the user for withdrawals (protocol-favorable rounding)
function previewRedeem(uint256 shares) public view returns (uint256 assets) {
    assets = shares * totalAssets / totalSupply; // Round down — user gets less
}
```

### Detection Methods
- **Manual**: Look for division before multiplication, division by large numbers, and share/asset conversion formulas
- **Foundry**: Fuzz with very small and very large values to catch edge cases

---

## 3.17 Rug Pull Mechanics & Honeypot Patterns

### Difficulty: Beginner

Rug pulls are malicious contracts designed to steal funds from users. Honeypots attract buyers but prevent selling.

### Common Rug Pull Patterns

```solidity
// Pattern 1: Hidden mint function
contract RugToken {
    // Owner can mint unlimited tokens and dump on users
    function _mint(address to, uint256 amount) internal { /* ... */ }
    function secretMint() external onlyOwner {
        _mint(owner, totalSupply() * 100); // 100x inflation
    }
}

// Pattern 2: Blacklist prevents selling
contract HoneypotToken {
    mapping(address => bool) private _isBlacklisted;
    function transfer(address to, uint256 amount) public override returns (bool) {
        require(!_isBlacklisted[msg.sender], ""); // Users can buy but not sell
        return super.transfer(to, amount);
    }
}

// Pattern 3: Fee manipulation
contract FeeRugToken {
    uint256 public sellFee = 0; // Starts at 0%
    function setSellFee(uint256 _fee) external onlyOwner {
        sellFee = _fee; // Owner sets to 99% after launch
    }
}

// Pattern 4: Proxy rug — owner upgrades implementation to drain
// Uses upgradeable proxy pattern — swaps implementation to a drainer contract
```

### Detection Red Flags
- Unverified contract source code
- Owner can mint, pause, blacklist, or modify fees without timelock
- Liquidity not locked (no lock contract holding LP tokens)
- Recently created deployer address funded by mixing services
- `approve()` to unknown addresses in constructor

---

## 3.18 Governance Attack Vectors

### Difficulty: Advanced

DeFi governance can be exploited through flash loans to temporarily acquire voting power.

### Vulnerable Code

```solidity
contract VulnerableGovernor {
    IERC20Votes public token;

    function propose(/* ... */) external returns (uint256) {
        // [NO] No minimum holding period — flash loan can meet threshold
        require(token.getVotes(msg.sender) >= proposalThreshold, "Below threshold");
        // ... create proposal
    }

    function castVote(uint256 proposalId, uint8 support) external {
        // [NO] Uses CURRENT balance, not snapshot
        uint256 weight = token.balanceOf(msg.sender);
        // ... record vote with weight
    }
}
```

### Fixed Code

```solidity
contract SecureGovernor {
    function propose(/* ... */) external returns (uint256) {
        // [YES] Uses past snapshot — flash loan doesn't affect historical balances
        require(
            token.getPastVotes(msg.sender, block.number - 1) >= proposalThreshold,
            "Below threshold"
        );
    }

    function castVote(uint256 proposalId, uint8 support) external {
        // [YES] Uses snapshot at proposal creation time
        uint256 weight = token.getPastVotes(msg.sender, proposals[proposalId].startBlock);
    }
}
```

**Real-world exploit:** Beanstalk (April 2022) — Attacker flash-loaned governance tokens, passed a malicious BIP (Beanstalk Improvement Proposal), drained $182M, and repaid the flash loan — all in a single transaction.

---

## 3.19 Proxy Storage Collisions

### Difficulty: Advanced

When a transparent proxy delegates to an implementation, both share the same storage space. If the proxy and implementation define variables in overlapping slots, state corruption occurs.

### Function Selector Clash

```solidity
// If proxy has: function admin() — selector 0xf851a440
// And implementation has: function clashingfunction() — selector 0xf851a440
// The proxy intercepts the call instead of forwarding to implementation!

// OpenZeppelin TransparentUpgradeableProxy prevents this by:
// - Restricting admin functions to admin-only
// - Non-admin calls always delegate to implementation
```

### EIP-1967 Gap Pattern

```solidity
// In OpenZeppelin's upgradeable contracts:
contract UpgradeableBase {
    // Reserve storage slots to prevent future versions from colliding
    uint256[50] private __gap; // [YES] Subclasses can add variables above the gap
}

contract UpgradeableChild is UpgradeableBase {
    uint256 public newVariable; // Uses slot immediately after parent's variables
    uint256[49] private __gap; // Gap shrinks by 1 for each new variable
}
```

---

## 3.20 Upgradeable Contract Pitfalls

### Difficulty: Advanced

### Initialization Flaws

```solidity
// [NO] Constructor doesn't run on implementation in proxy pattern!
contract VulnerableImplementation {
    address public owner;

    // This constructor only runs during deployment, NOT when used as proxy implementation
    constructor() {
        owner = msg.sender;
    }
}

// [YES] Use initializer pattern
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract SecureImplementation is Initializable {
    address public owner;

    function initialize(address _owner) external initializer {
        owner = _owner; // Runs once when proxy calls it
    }
}
```

### Uninitialized Implementation Risk

```solidity
// If the implementation contract is NOT initialized directly,
// an attacker can call initialize() on the implementation itself (not the proxy).
// Then they own the implementation and can call selfdestruct (pre-Dencun)
// or other privileged functions.

// [YES] Fix: Call _disableInitializers() in implementation constructor
constructor() {
    _disableInitializers(); // Prevents initialization of the implementation itself
}
```

**Real-world exploit:** Wormhole Uninitialized Proxy (2022) — The implementation contract was not initialized, potentially allowing an attacker to take ownership and upgrade it maliciously.

---

## 3.21 Non-Standard Token Issues

### Difficulty: Intermediate

Not all ERC-20 tokens behave identically. Assumptions about standard behavior lead to vulnerabilities.

| Token Type | Behavior | Risk |
|-----------|----------|------|
| **Fee-on-transfer** (USDT, STA) | Recipient receives less than `amount` | Accounting mismatch — protocol credits more than received |
| **Rebasing** (stETH, AMPL) | Balances change without transfers | Share calculations break |
| **ERC-777** (imBTC) | `tokensReceived` callback on transfer | Reentrancy vector |
| **Double-entry tokens** (some bridged TUSD) | Two addresses point to same balance | Double-counting in protocols |
| **Pausable** (USDC, USDT) | Transfers can be frozen | DoS for protocols relying on the token |
| **Blacklistable** (USDC, USDT) | Specific addresses blocked | Frozen funds in DeFi contracts |
| **Non-bool-returning** (USDT on mainnet) | `transfer()` returns nothing | Reverts if caller expects `bool` return |
| **Decimals ≠ 18** (USDC = 6, WBTC = 8) | Arithmetic assumptions break | Precision errors in price calculations |

### Handling Fee-on-Transfer Tokens

```solidity
// [NO] Assumes amount received == amount sent
function deposit(IERC20 token, uint256 amount) external {
    token.transferFrom(msg.sender, address(this), amount);
    balances[msg.sender] += amount; // Credits full amount, but less was received!
}

// [YES] Measure actual balance change
function deposit(IERC20 token, uint256 amount) external {
    uint256 balanceBefore = token.balanceOf(address(this));
    token.transferFrom(msg.sender, address(this), amount);
    uint256 received = token.balanceOf(address(this)) - balanceBefore;
    balances[msg.sender] += received; // Credits actual received amount
}
```

---

## 3.22 Sandwich Attacks & AMM Price Impact

### Difficulty: Advanced

Sandwich attacks exploit the deterministic price impact of AMM swaps. The attacker:
1. Detects a large pending swap in the mempool
2. Front-runs with a buy (pushing the price up)
3. Victim's swap executes at a worse price
4. Attacker back-runs with a sell (capturing the squeezed profit)

### Defense in Smart Contracts

```solidity
// [YES] Require slippage protection in swap functions
function swap(
    uint256 amountIn,
    uint256 amountOutMin, // [YES] Minimum acceptable output
    uint256 deadline      // [YES] Transaction expiry
) external {
    require(block.timestamp <= deadline, "Expired");
    uint256 amountOut = _calculateSwap(amountIn);
    require(amountOut >= amountOutMin, "Slippage exceeded");
    // ... execute swap
}
```

---

## 3.23 Price Impact Manipulation in Lending Protocols

### Difficulty: Advanced

Lending protocols that use collateral value based on AMM prices are vulnerable to flash-loan-powered price manipulation.

```
Attack Flow:
1. Flash loan large amount of Token A
2. Dump Token A on DEX → price of Token A crashes
3. Liquidate users who have Token A as collateral (now undercollateralized)
4. Buy Token A back at crashed price
5. Repay flash loan
```

### Defense
- Use TWAP oracles (multi-block)
- Implement liquidation cooldown periods
- Cap maximum position sizes
- Use Chainlink price feeds (not spot prices)

---

## 3.24 Donation / Inflation Attack on Vault Shares (ERC-4626)

### Difficulty: Advanced

This attack exploits the share-based accounting of ERC-4626 vaults. The attacker inflates the share price by "donating" assets directly to the vault, making subsequent depositors' shares round down to zero.

### Attack Flow

```
1. Attacker deposits 1 wei of asset → receives 1 share
2. Attacker donates 1e18 tokens directly to vault (transfer, not deposit)
3. Now: totalAssets = 1e18 + 1, totalSupply = 1
4. Share price: 1 share = ~1e18 tokens
5. Victim deposits 5e17 tokens (0.5 ETH):
   shares = 5e17 * 1 / (1e18 + 1) = 0 shares (rounds down!)
6. Victim's 0.5 ETH is now owned by the attacker's 1 share
```

### Vulnerable Code

```solidity
// Standard ERC-4626 deposit without protection
function deposit(uint256 assets, address receiver) public returns (uint256 shares) {
    shares = assets * totalSupply() / totalAssets();
    // [NO] If totalAssets is inflated and totalSupply is 1, shares rounds to 0
    _mint(receiver, shares);
    asset.transferFrom(msg.sender, address(this), assets);
}
```

### Fixed Code

```solidity
// [YES] Defense 1: Virtual shares and assets offset (OpenZeppelin approach)
function _convertToShares(uint256 assets, Math.Rounding rounding) internal view returns (uint256) {
    return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
}

// [YES] Defense 2: Require minimum initial deposit
function deposit(uint256 assets, address receiver) public returns (uint256 shares) {
    if (totalSupply() == 0) {
        require(assets >= MIN_DEPOSIT, "Below minimum");
        shares = assets - DEAD_SHARES; // Burn some shares to address(dead)
        _mint(address(0xdead), DEAD_SHARES); // Dead shares prevent inflation
    } else {
        shares = assets * totalSupply() / totalAssets();
    }
    require(shares > 0, "Zero shares");
    _mint(receiver, shares);
    asset.transferFrom(msg.sender, address(this), assets);
}
```

**Real-world exploit:** Multiple ERC-4626 vaults have been exploited via this pattern. The Yearn V3 and OpenZeppelin libraries now include built-in mitigations.

---

## Summary — Vulnerability Detection Quick Reference

| # | Vulnerability | Slither Detector | Key Pattern |
|---|--------------|-----------------|-------------|
| 1 | Reentrancy | `reentrancy-eth` | External call before state update |
| 2 | Integer overflow | (manual for `unchecked`) | `unchecked {}`, pre-0.8.x |
| 3 | Access control | `unprotected-upgrade` | Missing modifiers on state-changing functions |
| 4 | Front-running | (manual) | Order-dependent operations |
| 5 | Flash loan | (manual) | Spot price oracles |
| 6 | Oracle manipulation | (manual) | AMM reserve reads |
| 7 | Delegatecall | `controlled-delegatecall` | Storage layout mismatches |
| 8 | Uninitialized storage | `uninitialized-storage` | Pre-0.5.0 struct locals |
| 9 | Signature replay | (manual) | Missing nonce/chainId/deadline |
| 10 | Unsafe calls | `unchecked-lowlevel` | `.call` without return check |
| 11 | DoS | (manual) | Unbounded loops, push pattern |
| 12 | Self-destruct | `suicidal` | `address(this).balance` checks |
| 13 | Timestamp | `timestamp` | `block.timestamp` in conditionals |
| 14 | Block number | (manual) | Block-time assumptions |
| 15 | Randomness | `weak-prng` | `blockhash`, `prevrandao` |
| 16 | Precision loss | (manual) | Division before multiplication |
| 17 | Rug pulls | (manual) | Unverified, mint, blacklist |
| 18 | Governance | (manual) | Flash-loan-able voting |
| 19 | Proxy collision | (manual) | Storage slot overlap |
| 20 | Upgradeable | `uninitialized-state` | Missing `initializer` |
| 21 | Token quirks | (manual) | Fee-on-transfer, rebasing |
| 22 | Sandwich | (manual) | Missing slippage checks |
| 23 | Lending manipulation | (manual) | Spot price collateral valuation |
| 24 | Vault inflation | (manual) | First depositor share rounding |

> **Key Takeaway:** Most high-severity findings in competitive audits are NOT reentrancy or overflow — those are well-known and often caught by tools. The highest-paying findings in 2024–2025 are: logic errors in economic models, cross-contract interaction issues, incorrect assumptions about external token behavior, and oracle manipulation. Train your eye to think about economic invariants, not just code patterns.

---

*← [Previous: Recon & OSINT](./RECON_AND_OSINT.md) | [Next: Audit Methodology →](./AUDIT_METHODOLOGY.md)*
