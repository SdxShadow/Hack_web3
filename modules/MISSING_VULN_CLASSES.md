---
title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)3 — Missing Vulnerability Classes & Advanced Attack Patterns"
description: "Advanced vulnerability classes not covered elsewhere: EIP-1153 transient storage attacks, ERC-721/1155 callback reentrancy, permit phishing, multicall msg.value reuse, read-only reentrancy, weird ERC-20, Solidity compiler bugs, return bomb, Chainlink edge cases, TWAP manipulation, vault rounding, governance timelock bypass, cross-chain replay, assembly pitfalls, staking bugs, signature malleability, liquidation exploits, and proxy upgrade patterns."
keywords: "transient storage security, EIP-1153, read-only reentrancy, weird ERC-20, permit phishing, multicall vulnerability, Chainlink edge cases, TWAP manipulation, vault rounding, governance bypass, signature malleability, ERC-721 ERC-1155 callback reentrancy, Solidity compiler zero-day bugs, return bomb gas griefing, cross-chain replay attack"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "[Module 1](./BLOCKCHAIN_FUNDAMENTALS.md)3 — Missing Vulnerability Classes | Web3 Hacker Guide"
og_description: "Advanced vulnerability classes for top-1% auditors: transient storage, read-only reentrancy, multicall exploits, Chainlink edge cases, and 15+ more critical patterns."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/13_MISSING_VULN_CLASSES"
schema_type: "TechArticle"
difficulty: " Advanced →  Expert"
module: 13
tags: [transient-storage, EIP-1153, read-only-reentrancy, weird-ERC-20, permit-phishing, Chainlink, TWAP, vault-rounding, proxy-upgrade]
nav_order: 13
parent: "Web3 Hacker & Pentester Guide"
---



---

*← [Previous: Advanced Topics](./ADVANCED_TOPICS.md) | [Next: Bug Bounty Playbook →](./BUG_BOUNTY_PLAYBOOK.md)*
anced |
| Vault rounding direction | §13.11 |  Advanced |
| Governance timelock bypass | §13.12 |  Advanced |
| Cross-chain replay | §13.13 |  Advanced |
| Assembly pitfalls | §13.14 |  Advanced |
| Staking reward bugs | §13.15 |  Intermediate |
| Donation price manipulation | §13.16 |  Advanced |
| Signature malleability | §13.17 |  Intermediate |
| Liquidation mechanism bugs | §13.18 |  Advanced |
| Bundle ordering attacks | §13.19 |  Expert |
| Proxy upgrade patterns | §13.20 |  Advanced |dvanced |
| ERC-721/1155 callback reentrancy | §13.2 |  Intermediate |
| Permit front-running & phishing | §13.3 |  Intermediate |
| Multicall msg.value reuse | §13.4 |  Advanced |
| Read-only reentrancy (deep) | §13.5 |  Advanced |
| Weird ERC-20 complete reference | §13.6 |  Intermediate |
| Compiler bugs | §13.7 |  Intermediate |
| Return bomb / gas griefing | §13.8 |  Intermediate |
| Chainlink edge cases (complete) | §13.9 |  Advanced |
| TWAP manipulation cost analysis | §13.10 |  Adventation
// If the beacon is compromised, ALL proxies are compromised simultaneously

// Attack surface:
// 1. Beacon owner key compromise → upgrade all proxies at once
// 2. Beacon contract bug → all proxies affected
// 3. No timelock on beacon upgrades → instant mass upgrade

// Defense: Timelock on beacon upgrades, multisig beacon owner
```

---

## Summary — Gap Coverage

| Vulnerability Class | Covered In | Difficulty |
|--------------------|-----------|------------|
| Transient storage bugs | §13.1 |  Atation is deployed without a proxy and initialized by an attacker,
// they can call upgradeTo(maliciousContract) then selfdestruct the implementation

// Post-Dencun (EIP-6780): selfdestruct only works in same tx as creation
// But the upgrade-to-malicious-implementation attack still works

// [YES] Fix: _disableInitializers() in implementation constructor
constructor() {
    _disableInitializers();
}
```

### Beacon Proxy Attacks

```solidity
// Beacon proxies: all proxies point to a beacon, beacon points to implemndent,
an attacker can exploit ordering within the bundle.

Example:
Bundle: [TX1: deposit, TX2: borrow, TX3: manipulate oracle, TX4: liquidate]
- TX3 happens AFTER TX2 — the borrow was at the correct price
- TX4 liquidates a different user whose position became underwater due to TX3
- This is legitimate MEV but can be used maliciously
```

---

## 13.20 Proxy Upgrade Attack Patterns

### UUPS Self-Destruct (Pre-Dencun)

```solidity
// UUPS proxies have the upgrade logic in the IMPLEMENTATION
// If the implemen The protocol absorbs the bad debt
- This dilutes all depositors

Attack:
1. Create many small undercollateralized positions
2. Each position creates bad debt when liquidated
3. Aggregate bad debt drains the protocol's reserves

Defense: Minimum position sizes, liquidation buffers, insurance funds
```

---

## 13.19 Flashbots Bundle Ordering Attacks

### Bundle Dependency Exploitation

```
When submitting Flashbots bundles, transactions execute in order.
If a protocol assumes transactions in a bundle are indepecan be exploited

// Attack: Self-liquidation
// 1. Deposit collateral
// 2. Borrow to the edge of liquidation threshold
// 3. Slightly push your own position underwater (via oracle manipulation or donation)
// 4. Liquidate yourself from a different address
// 5. Collect the liquidation bonus

// Defense: Prevent self-liquidation (check borrower != liquidator)
// Or: Ensure liquidation bonus < oracle manipulation cost
```

### Bad Debt Socialization

```
When a position is liquidated but collateral < debt:
-sses the check
}

// [YES] Fixed: Use OpenZeppelin's ECDSA which enforces lower-s
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

function verify(bytes32 hash, bytes memory sig) internal pure returns (address) {
    return ECDSA.recover(hash, sig); // Enforces s <= secp256k1.n/2
}
```

---

## 13.18 Liquidation Mechanism Vulnerabilities

### Liquidation Incentive Manipulation

```solidity
// Protocols offer liquidation bonuses to incentivize liquidators
// Vulnerability: If bonus is too high, it 

```solidity
// ECDSA signatures have two valid forms for any message:
// (v, r, s) and (v', r, s') where s' = secp256k1.n - s
// Both recover to the same address!

// [NO] Vulnerable: Using raw ecrecover
function verify(bytes32 hash, bytes memory sig) internal pure returns (address) {
    (bytes32 r, bytes32 s, uint8 v) = splitSig(sig);
    return ecrecover(hash, v, r, s);
    // Attacker can submit (v', r, s') — different bytes, same signer
    // If signature is used as a key (mapping[sig] = true), this bypatokens directly to Uniswap V2 pair
// 2. Call sync() to update reserves
// 3. Price is now manipulated
// 4. Any protocol reading the spot price is affected

// This is different from flash loan manipulation:
// - Flash loan: borrow → manipulate → repay (atomic)
// - Donation: permanent price change (until arbitraged away)
// - Cost: the donated tokens (not free like flash loans)
// - Benefit: can persist across blocks (affects TWAP over time)
```

---

## 13.17 Signature Malleability

### ECDSA Malleabilitytake them right before reward distribution
3. Claim disproportionate rewards
4. Unstake and repay flash loan

Defense: Use snapshot-based reward calculation
         Implement minimum staking duration
         Use time-weighted average stake
```

---

## 13.16 Price Manipulation via Donation

### Direct Token Donation to Pools

```solidity
// Uniswap V2: Reserves are tracked via balanceOf()
// If you send tokens directly to the pool (not via swap), reserves are off
// until sync() is called

// Attack:
// 1. Send oken() returns 0 due to integer division
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenStored;
        return rewardPerTokenStored + (rewardRate * (block.timestamp - lastUpdate) * 1e18 / totalStaked);
    }

    // Attack: Stake a massive amount to make rewardPerToken() = 0
    // Other stakers earn 0 rewards while attacker holds the stake
}
```

### Reward Manipulation via Flash Stake

```
Attack:
1. Flash loan governance/staking tokens
2. Srn(0, 32)
    }
}
```

---

## 13.15 Staking & Reward Distribution Bugs

### Reward Calculation Precision

```solidity
// Common pattern: rewards per token accumulated
// Vulnerable to precision loss and manipulation

contract VulnerableStaking {
    uint256 public rewardPerTokenStored;
    uint256 public totalStaked;
    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;

    // [NO] If totalStaked is very large and rewardRate is small,
    // rewardPerTal pure returns (bytes32) {
    assembly {
        // [NO] Upper bits of addr may not be clean (if passed from calldata)
        let packed := or(shl(96, addr), value)
        mstore(0, packed)
        return(0, 32)
    }
}

// [YES] Clean the address first
function packDataSafe(address addr, uint96 value) internal pure returns (bytes32) {
    assembly {
        let cleanAddr := and(addr, 0xffffffffffffffffffffffffffffffffffffffff)
        let packed := or(shl(96, cleanAddr), value)
        mstore(0, packed)
        retuemory data) internal pure returns (bytes32 result) {
    assembly {
        result := mload(data) // [NO] Reads the LENGTH, not the data!
        // data points to the length prefix
        // data + 32 points to the actual bytes
    }
}

// [YES] Correct
function goodAssembly(bytes memory data) internal pure returns (bytes32 result) {
    assembly {
        result := mload(add(data, 32)) // [YES] Skip the 32-byte length prefix
    }
}

// [NO] Dirty bits in packed data
function packData(address addr, uint96 value) intern);
}
```

### Bridge Message Replay

```
Attack: A message processed on Chain A is replayed on Chain B
        (or replayed multiple times on the same chain)

Required protections:
1. Nonce per sender per destination chain
2. Message hash tracking (mark as processed)
3. Chain ID in message payload
4. Destination contract address in message payload
```

---

## 13.14 Solidity Assembly Pitfalls

### Common Inline Assembly Bugs

```solidity
// [NO] Memory corruption via incorrect offset
function badAssembly(bytes mInvalid sig");
    // No chainId in hash → same sig works on Ethereum, Polygon, BSC, etc.
}

// [YES] Include chainId in signed data
function execute(bytes32 dataHash, bytes memory sig) external {
    bytes32 hash = keccak256(abi.encodePacked(
        block.chainid,          // [YES] Chain-specific
        address(this),          // [YES] Contract-specific
        nonces[owner]++,        // [YES] Replay protection
        dataHash
    ));
    address signer = ECDSA.recover(hash, sig);
    require(signer == owner, "Invalid sig""Not guardian");
        token.transfer(to, amount); // No timelock, no multisig
    }
}

// Attack: Compromise the guardian key → instant fund drain
// Defense: Even emergency functions should require multisig
//          or have a short (1-hour) timelock
```

---

## 13.13 Cross-Chain Replay Attacks

### Chain ID Validation

```solidity
// [NO] Signature valid on all chains
function execute(bytes32 hash, bytes memory sig) external {
    address signer = ECDSA.recover(hash, sig);
    require(signer == owner, "Require independent security review for each proposal
         Implement proposal dependency tracking
```

### Emergency Function Abuse

```solidity
// Many protocols have emergency functions that bypass timelock
// These are intended for crisis response but create attack vectors

contract VulnerableProtocol {
    address public guardian;

    // [NO] Guardian can drain funds instantly with no timelock
    function emergencyWithdraw(address to, uint256 amount) external {
        require(msg.sender == guardian,  harvest proportional to deposit

Defense: Implement withdrawal fees, vesting periods, or per-block deposit limits
```

---

## 13.12 Governance Timelock Bypass Patterns

### Proposal Batching Attack

```
Attack: Submit multiple proposals that individually look harmless
        but combined execute a malicious action

Example:
- Proposal A: "Add new token as collateral" (looks benign)
- Proposal B: "Update oracle for new token" (looks benign)
- Combined: New token uses a manipulable oracle → exploit

Defense: ays less
    // Attacker can repeatedly withdraw/deposit to drain vault via rounding
}

// [YES] Correct: Round UP for withdrawals
function previewWithdraw(uint256 assets) public view returns (uint256 shares) {
    return assets.mulDiv(totalSupply(), totalAssets(), Math.Rounding.Ceil);
}
```

### Vault Sandwich Attack

```
Attack on vaults with delayed price updates:
1. Deposit large amount before a profitable harvest
2. Harvest increases vault's totalAssets
3. Withdraw immediately after harvest
4. Profit = share ofulable with flash loans
```

---

## 13.11 Vault / Share Accounting Bugs

### Rounding Direction Errors

```solidity
// ERC-4626 standard: Round DOWN for deposits (user gets fewer shares)
//                    Round UP for withdrawals (user pays more assets)
// This protects the vault from value leakage

// [NO] Wrong rounding direction — rounds in user's favor
function previewWithdraw(uint256 assets) public view returns (uint256 shares) {
    return assets * totalSupply() / totalAssets(); // Rounds DOWN — user pTick);
}
```

### TWAP Manipulation Cost

```
To manipulate a 30-minute TWAP by X%:
- Attacker must hold the price at X% deviation for 30 minutes
- Cost = capital locked × opportunity cost × 30 minutes
- On liquid pairs (ETH/USDC), this costs millions per % of manipulation
- On illiquid pairs (small-cap tokens), manipulation is cheap

Rule of thumb:
- 30-minute TWAP on ETH/USDC: Very expensive to manipulate
- 30-minute TWAP on small-cap token: Potentially cheap
- 1-block TWAP: Essentially a spot price — manipapital over time

function getTWAP(address pool, uint32 secondsAgo) external view returns (uint256 price) {
    uint32[] memory secondsAgos = new uint32[](2);
    secondsAgos[0] = secondsAgo;
    secondsAgos[1] = 0;

    (int56[] memory tickCumulatives, ) = IUniswapV3Pool(pool).observe(secondsAgos);

    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
    int24 arithmeticMeanTick = int24(tickCumulativesDelta / int56(uint56(secondsAgo)));

    price = TickMath.getSqrtRatioAtTick(arithmeticMeanht LUNA was worth $0.10
- Attackers borrowed against "valuable" LUNA collateral
- Protocols suffered massive bad debt

Detection: Check if the feed has minAnswer/maxAnswer set
cast call 0xFeedAddress "minAnswer()(int192)"
cast call 0xFeedAddress "maxAnswer()(int192)"
```

---

## 13.10 Uniswap V3 TWAP Oracle Manipulation

### TWAP Mechanics

```solidity
// Uniswap V3 TWAP: Time-Weighted Average Price over N seconds
// More resistant to flash loan manipulation than spot price
// But still manipulable with sustained cnt256 startedAt,,) = sequencerFeed.latestRoundData();
    require(answer == 0, "Sequencer down"); // 0 = up, 1 = down
    require(block.timestamp - startedAt > GRACE_PERIOD, "Sequencer just restarted");
    // Grace period prevents using prices from right after sequencer restart
}
```

### Chainlink minAnswer/maxAnswer Exploit

```
Real scenario: LUNA crash (May 2022)
- Chainlink LUNA/USD feed had minAnswer = $0.10
- When LUNA crashed to $0.0001, the feed still reported $0.10
- Protocols using this feed thougime (for Arbitrum, Optimism, etc.)
    // Must check sequencer uptime feed before using any price
    _checkSequencerUptime();

    // [YES] Check 6: Price within expected bounds (circuit breaker)
    // Some feeds have minAnswer/maxAnswer — price clamps at these values
    // If real price is outside bounds, feed returns the clamped value
    // This is NOT detectable from the feed itself — use secondary validation

    return uint256(answer);
}

function _checkSequencerUptime() internal view {
    (, int256 answer, uice is positive
    require(answer > 0, "Negative price");

    // [YES] Check 2: Round is complete
    require(updatedAt != 0, "Incomplete round");

    // [YES] Check 3: Answer is from the latest round (not stale)
    require(answeredInRound >= roundId, "Stale price");

    // [YES] Check 4: Price is not too old (heartbeat varies by feed)
    // ETH/USD heartbeat = 3600s, BTC/USD = 3600s, some = 86400s
    require(block.timestamp - updatedAt <= HEARTBEAT + GRACE_PERIOD, "Price too old");

    // [YES] Check 5: L2 sequencer upt256 public counter;
    receive() external payable {
        counter++; // Requires SSTORE — 20,000 gas — transfer() will fail!
    }
}
```

---

## 13.9 Chainlink Oracle Edge Cases

### Complete Chainlink Validation Checklist

```solidity
function getPrice(AggregatorV3Interface feed) internal view returns (uint256) {
    (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    ) = feed.latestRoundData();

    // [YES] Check 1: Pri   require(success, "Call failed");
}
```

### Gas Stipend Attacks

```solidity
// transfer() and send() forward only 2300 gas
// This is NOT enough for:
// - Writing to storage (20,000 gas)
// - Emitting events (375+ gas per topic)
// - Complex receive() logic

// Attack: Make a contract whose receive() requires more than 2300 gas
// Any protocol using transfer() to send ETH to this contract will fail
// This can permanently lock funds if the protocol has no fallback

contract GasGriefingReceiver {
    uint
contract Victim {
    function callExternal(address target) external {
        // [NO] No gas limit on returndata copy
        (bool success, bytes memory data) = target.call{gas: 100000}("");
        // If target returns 1MB, copying it costs enormous gas
        // This can cause the caller's transaction to run out of gas
    }
}

// [YES] Fix: Limit returndata size
function callExternal(address target) external {
    (bool success, ) = target.call{gas: 100000}("");
    // Don't copy returndata — just check success
 ncoded metadata')
"

# Use solc-select to test with specific versions
pip3 install solc-select
solc-select install 0.8.13
solc-select use 0.8.13
```

---

## 13.8 Gas Griefing & Return Bomb

### Return Bomb Attack

```solidity
// Attacker returns massive returndata to consume caller's memory expansion gas

contract ReturnBomber {
    fallback() external {
        assembly {
            // Return 1MB of data — caller pays for memory expansion
            return(0, 1000000)
        }
    }
}

// Victim contract:ge pointer | Storage corruption |
| 0.6.x | `calldata` slicing bug | Incorrect data access |
| Vyper 0.2.15–0.3.0 | Reentrancy lock bug | Reentrancy guard ineffective |

```bash
# Check for known compiler bugs
# https://docs.soliditylang.org/en/latest/bugs.html

# Extract compiler version from deployed bytecode
cast code 0xContract --rpc-url $ETH_RPC | python3 -c "
import sys, json
bytecode = sys.stdin.read().strip()
# Last 43 bytes are CBOR metadata
# Contains compiler version
print('Check last bytes for CBOR-eeived < amount) {
            console.log("Fee on transfer detected:", amount - received);
        }
    }
}
```

---

## 13.7 Solidity Compiler Bugs

### Known Critical Compiler Bugs

| Version | Bug | Impact |
|---------|-----|--------|
| 0.8.13–0.8.15 | ABI encoding bug with nested arrays | Incorrect calldata encoding |
| 0.8.13–0.8.15 | Optimizer bug with `verbatim` | Incorrect code generation |
| 0.7.0–0.8.16 | `abi.encode` with `bytes` in optimizer | Data corruption |
| 0.4.x–0.5.x | Uninitialized storame token |

### Testing for Token Compatibility

```solidity
// Foundry test template for token compatibility
contract TokenCompatibilityTest is Test {
    function test_feeOnTransfer(address token, address from, address to, uint256 amount) public {
        uint256 balBefore = IERC20(token).balanceOf(to);
        vm.prank(from);
        IERC20(token).transfer(to, amount);
        uint256 received = IERC20(token).balanceOf(to) - balBefore;
        // If received < amount, it's fee-on-transfer
        if (reccy |
| **Low decimals** | USDC (6), WBTC (8) | Precision loss in price calculations |
| **High decimals** | YAM (24) | Overflow risk in multiplication |
| **Transfer to zero** | Some tokens revert | DoS if protocol sends to address(0) |
| **Self-transfer** | Some tokens break | Accounting errors on `transfer(self, amount)` |
| **Flash mintable** | Some DeFi tokens | Temporary infinite supply |
| **Deflationary** | BOMB | Balance decreases over time |
| **Multiple entry points** | TUSD | Two addresses for saansfer** | STA, PAXG | Protocol credits more than received |
| **Rebasing** | stETH, AMPL, OHM | Balance changes without transfer events |
| **Approval race condition** | USDT | `approve(X)` then `approve(Y)` — front-run to spend both |
| **Blocklist** | USDC, USDT | Protocol funds frozen if contract is blacklisted |
| **Pausable** | USDC, USDT | Protocol DoS if token is paused |
| **Upgradeable** | USDC | Token behavior can change after audit |
| **ERC-777 hooks** | imBTC | `tokensReceived` callback = reentran that protocol makes external calls before updating state
```

---

## 13.6 Weird ERC-20 Edge Cases (Complete Reference)

### The Weird ERC-20 Repository

[github.com/d-xo/weird-erc20](https://github.com/d-xo/weird-erc20) catalogs all non-standard token behaviors. Every auditor must know these.

| Token Behavior | Example Tokens | Attack Vector |
|---------------|----------------|---------------|
| **Missing return value** | USDT (mainnet), BNB | `transfer()` reverts if caller uses `bool` return |
| **Fee on trllback:
  → Protocol A's balances are NOT yet updated
  → get_virtual_price() returns STALE (inflated) value
  → Attacker borrows from Protocol B using inflated collateral
  → Protocol A finishes updating balances
  → Attacker's position is now undercollateralized
```

### Detection

```bash
# Look for view functions that read state that could be mid-update
# during an external call

# Slither doesn't catch this — manual review required
# Pattern: Protocol reads price/state from another protocol
#         ANDt has special permissions
```

---

## 13.5 Cross-Contract Reentrancy (Read-Only Reentrancy Deep Dive)

### The Pattern

Read-only reentrancy is the most subtle and dangerous reentrancy variant. It doesn't steal from the re-entered contract — it exploits stale state that another protocol reads.

```
Protocol A (Curve pool):
  remove_liquidity() → sends ETH → [callback] → updates balances

Protocol B (Lending protocol):
  Uses Curve's get_virtual_price() as collateral oracle

Attack:
  During Protocol A's ETH ca

**Real-world exploit:** Uniswap V3 Multicall (2021) — This exact pattern was identified. Uniswap's multicall is safe because it uses `payable` only on specific functions, but many forks got this wrong.

### Delegatecall in Multicall Context

```solidity
// When multicall uses delegatecall, msg.sender is preserved
// But if the called function uses msg.sender for access control,
// the multicall contract itself becomes the "caller" in some contexts

// This can bypass access control if the multicall contraccall {
    function multicall(bytes[] calldata data) external payable {
        for (uint i = 0; i < data.length; i++) {
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success);
        }
    }

    function deposit() external payable {
        balances[msg.sender] += msg.value; // [NO] msg.value is the SAME for all calls!
    }
}

// Attack: Call multicall with 1 ETH, but include deposit() 10 times
// Each deposit() sees msg.value = 1 ETH → credited 10 ETH for 1 ETH sent
```
3. The message is actually an EIP-2612 permit for all their tokens
4. Attacker submits the permit + transferFrom in one tx
5. User's tokens are drained without them ever sending a transaction

Defense: Wallets should decode and display permit signatures clearly
```

---

## 13.4 Multicall Vulnerabilities

### msg.value Reuse in Multicall

```solidity
// Critical vulnerability: msg.value is shared across all calls in a multicall
// An attacker can reuse the same ETH value multiple times

contract VulnerableMulti
function depositWithPermit(uint256 amount, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external {
    // If permit was already called (front-run), this try/catch handles it gracefully
    try token.permit(msg.sender, address(this), amount, deadline, v, r, s) {} catch {}
    // The allowance is set either way — proceed with deposit
    token.transferFrom(msg.sender, address(this), amount);
}
```

### Permit Phishing

```
Attack:
1. Attacker creates a malicious dApp
2. User is asked to sign a "login message"sless approvals via signatures
// Vulnerability: permit() can be front-run

// Victim signs: permit(owner=Alice, spender=Bob, value=100, deadline=X)
// Attacker sees this in mempool
// Attacker front-runs with the SAME signature to call permit() first
// Then calls transferFrom() before Bob can

// This is NOT a vulnerability in permit itself — it's a design consideration
// Protocols that use permit() must handle the case where permit() was already called

// [YES] Correct pattern: wrap permit + action atomicallyId still listed at old price!
        market.buy{value: price}(tokenId);
        return this.onERC721Received.selector;
    }
}
```

### ERC-1155 Batch Transfer Callbacks

```solidity
// ERC-1155 safeBatchTransferFrom calls onERC1155BatchReceived
// Multiple tokens transferred in one call = multiple reentrancy opportunities
// Each token in the batch can trigger a different callback path
```

---

## 13.3 Permit / EIP-2612 Vulnerabilities

### Permit Front-Running

```solidity
// EIP-2612 permit() allows gaiggers onERC721Received callback BEFORE state cleanup
        nft.safeTransferFrom(seller, msg.sender, tokenId);

        // State cleanup happens AFTER the callback — reentrancy window!
        delete sellers[tokenId];
        delete prices[tokenId];
        payable(seller).transfer(msg.value);
    }
}

// Attacker's contract:
contract NFTReentrancyAttacker {
    function onERC721Received(address, address, uint256 tokenId, bytes calldata)
        external returns (bytes4)
    {
        // Re-enter buy() — tokenulnerabilities

### ERC-721/1155 Callback Reentrancy

```solidity
// ERC-721 safeTransferFrom calls onERC721Received on the recipient
// This is a reentrancy vector if state isn't updated first

contract VulnerableNFTMarket {
    mapping(uint256 => address) public sellers;
    mapping(uint256 => uint256) public prices;

    function buy(uint256 tokenId) external payable {
        require(msg.value >= prices[tokenId], "Insufficient payment");
        address seller = sellers[tokenId];

        // [NO] NFT transfer trransient storage used as a "flag" that can be pre-set
contract VulnerableWithTransient {
    // Attacker can call setFlag() before calling privilegedAction()
    // in the same transaction
    function setFlag() external {
        assembly { tstore(0, 1) }
    }

    function privilegedAction() external {
        uint256 flag;
        assembly { flag := tload(0) }
        require(flag == 1, "Not authorized"); // [NO] Anyone can set this!
        // ... privileged logic
    }
}
```

---

## 13.2 Callback & Hook Vsactions but NOT between calls in the same tx | Reentrancy if guard logic is flawed |
| **Cross-function transient state** | Transient values set in one function are readable in another within the same tx | Unexpected state sharing |
| **Callback manipulation** | Attacker sets transient values before calling a contract that reads them | Logic bypass |
| **Flash loan interaction** | Transient storage persists through flash loan callbacks | State leakage across protocol boundaries |

```solidity
// Vulnerable: Tonly for the duration of a transaction, then is automatically cleared.

```solidity
// Solidity 0.8.24+ syntax
assembly {
    tstore(slot, value)   // Write to transient storage
    value := tload(slot)  // Read from transient storage
}
```

### Security Implications

| Issue | Description | Impact |
|-------|-------------|--------|
| **Reentrancy guard bypass** | If a reentrancy guard uses transient storage, it resets between transses & Advanced Attack Patterns

> **Difficulty:** Advanced →  Expert
>
> This module covers critical vulnerability classes and attack patterns that are absent from the core guide but appear regularly in competitive audits and real-world exploits. Mastering these is what separates a top-10% auditor from a top-1% auditor.

---

## 13.1 Transient Storage Vulnerabilities (EIP-1153)

### What Is Transient Storage?

EIP-1153 (activated in Dencun, March 2024) introduces `TSTORE`/`TLOAD` opcodes — storage that persists # Module 13 — Missing Vulnerability Cla