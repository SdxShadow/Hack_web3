---
title: "Module 05 — Web3 Security Tools & Frameworks Deep Dive"
description: "Hands-on guide to every Web3 security tool: Foundry (forge, cast, anvil), Slither static analysis, Echidna fuzzing, Mythril symbolic execution, Certora formal verification, Heimdall bytecode analysis, Tenderly debugging, and scripting with ethers.js and web3.py."
keywords: "Foundry tutorial, Slither static analysis, Echidna fuzzing, Mythril symbolic execution, Certora formal verification, Heimdall bytecode analysis, Tenderly debugger, web3 security tools, forge test, cast CLI, Medusa fuzzer smart contracts, Halmos restricted symbolic execution, Wake python framework, smart contract CI/CD security"
author: "Web3 Security Research"
date: 2025-01-01
last_modified_at: 2026-03-21
og_title: "Module 05 — Web3 Security Tools | Web3 Hacker Guide"
og_description: "Complete hands-on guide to web3 security tools — Foundry, Slither, Echidna, Mythril, Certora, Heimdall, and Tenderly with real configurations and usage examples."
og_type: "article"
twitter_card: "summary_large_image"
canonical_url: "https://sdxshadow.github.io/Hack_web3/05_TOOLS_AND_FRAMEWORKS"
schema_type: "TechArticle"
difficulty: " Intermediate"
module: 5
tags: [Foundry, Slither, Echidna, Mythril, Certora, Heimdall, web3-tools, fuzzing, static-analysis]
nav_order: 5
parent: "Web3 Hacker & Pentester Guide"
---

# Module 05 — Tools & Frameworks Deep Dive

> **Difficulty:** Intermediate
>
> Mastering your tools is as important as understanding vulnerabilities. This module provides hands-on instructions for every major Web3 security tool — installation, configuration, real-world usage examples, and expert tips.

---

## 5.1 Foundry (Forge, Cast, Anvil)

Foundry is the industry-standard framework for smart contract development and security testing. It's written in Rust, fast, and designed for serious auditors.

### Setup

```bash
# Install
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Initialize a new project
forge init my-audit && cd my-audit

# Install dependencies
forge install OpenZeppelin/openzeppelin-contracts
forge install foundry-rs/forge-std
```

### Forge — Testing & Fuzzing

```bash
# Run all tests
forge test

# Run tests with verbosity (see traces on failure)
forge test -vvvv

# Run a specific test
forge test --mt test_reentrancy -vvvv

# Fuzz testing (runs random inputs)
forge test --mt testFuzz_ --fuzz-runs 10000

# Invariant testing
forge test --mt invariant_ --fuzz-runs 5000

# Fork testing (test against mainnet state)
forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/KEY --fork-block-number 18500000

# Coverage
forge coverage --report summary
forge coverage --report lcov  # For VS Code Coverage Gutters
```

### Foundry Cheatcodes for Pentesters

```solidity
// Essential cheatcodes for exploit PoCs
vm.prank(address)              // Sets msg.sender for the next call
vm.startPrank(address)         // Sets msg.sender for all subsequent calls
vm.deal(address, uint256)      // Sets ETH balance of an address
vm.warp(uint256)               // Sets block.timestamp
vm.roll(uint256)               // Sets block.number
vm.store(address, bytes32, bytes32)  // Write directly to storage slot
vm.load(address, bytes32)      // Read storage slot
vm.expectRevert()              // Next call should revert
vm.expectEmit()                // Assert event emission
vm.createFork(url)             // Create a fork
vm.selectFork(forkId)          // Switch between forks
vm.makePersistent(address)     // Address persists across fork switches
vm.label(address, string)      // Label address in traces for readability
vm.sign(uint256, bytes32)      // Sign with a private key
vm.addr(uint256)               // Derive address from private key
vm.envUint(string)             // Read environment variable
```

### Cast — Live Chain Interaction

```bash
# Read contract state
cast call 0xContract "balanceOf(address)(uint256)" 0xUser --rpc-url $ETH_RPC

# Send a transaction
cast send 0xContract "transfer(address,uint256)" 0xTo 1000000 \
  --private-key $PK --rpc-url $ETH_RPC

# Decode calldata
cast calldata-decode "swap(uint256,uint256,address,bytes)" 0x022c0d9f...

# Look up function selector
cast 4byte 0xa9059cbb

# Get storage slot
cast storage 0xContract 0 --rpc-url $ETH_RPC

# Get contract code
cast code 0xContract --rpc-url $ETH_RPC

# Compute storage slot for mapping
cast index address 0xKey 3  # mapping at slot 3, key = 0xKey

# Trace a transaction
cast run 0xTxHash --rpc-url $ETH_RPC

# Estimate gas for a call
cast estimate 0xContract "transfer(address,uint256)" 0xTo 100 --rpc-url $ETH_RPC

# Convert units
cast to-wei 1.5 ether           # 1500000000000000000
cast from-wei 1000000000000000000  # 1.0
cast --to-base 255 hex            # 0xff
```

### Anvil — Local Chain

```bash
# Start local chain
anvil

# Fork mainnet at a specific block
anvil --fork-url $ETH_RPC --fork-block-number 18500000

# Start with specific accounts
anvil --accounts 10 --balance 10000

# Mine blocks on demand (no auto-mining)
anvil --no-mining

# Impersonate any address (useful for testing admin functions)
cast rpc anvil_impersonateAccount 0xWhaleAddress
cast send 0xContract "adminFunction()" --from 0xWhaleAddress --unlocked
```

---

## 5.2 Hardhat

### Setup & Plugin Ecosystem

```bash
# Initialize
npx hardhat init

# Key plugins for security
npm install @nomicfoundation/hardhat-toolbox
npm install hardhat-gas-reporter
npm install solidity-coverage
npm install @openzeppelin/hardhat-upgrades
```

### Mainnet Forking

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    hardhat: {
      forking: {
        url: "https://eth-mainnet.g.alchemy.com/v2/KEY",
        blockNumber: 18500000, // Pin to specific block for reproducibility
      },
    },
  },
};
```

### Console.log Debugging

```solidity
// Import in Solidity — only works in Hardhat tests
import "hardhat/console.sol";

function complexLogic(uint256 x) internal {
    console.log("Value of x:", x);
    console.log("Sender:", msg.sender);
    console.log("Balance:", address(this).balance);
}
```

```bash
# Run tests with gas reporter
REPORT_GAS=true npx hardhat test

# Run coverage
npx hardhat coverage

# Compile with optimizer details
npx hardhat compile --force
```

---

## 5.3 Slither — Static Analysis

### Installation & Basic Usage

```bash
# Install
pip3 install slither-analyzer

# Run all detectors
slither .

# Run specific detectors
slither . --detect reentrancy-eth,unchecked-lowlevel,arbitrary-send-eth

# Export to JSON for processing
slither . --json slither-output.json

# Print contract summary
slither . --print human-summary

# Print function summary (call graph)
slither . --print function-summary

# Print inheritance graph
slither . --print inheritance-graph

# Print storage layout
slither . --print variable-order
```

### Key Detectors for Auditors

| Detector | Severity | Description |
|----------|----------|-------------|
| `reentrancy-eth` | High | Reentrancy with ETH transfer |
| `reentrancy-no-eth` | Medium | Reentrancy without ETH |
| `arbitrary-send-eth` | High | Functions sending ETH to arbitrary address |
| `controlled-delegatecall` | High | Delegatecall to user-controlled address |
| `suicidal` | High | Functions allowing self-destruct |
| `unprotected-upgrade` | High | Upgradeable proxy without access control |
| `unchecked-lowlevel` | Medium | Unchecked low-level call return value |
| `unchecked-transfer` | Medium | Unchecked ERC20 transfer return value |
| `uninitialized-storage` | High | Uninitialized storage variables |
| `timestamp` | Low | Dangerous use of block.timestamp |
| `weak-prng` | High | Weak randomness source |

### Writing Custom Detectors

```python
# custom_detector.py
from slither.detectors.abstract_detector import AbstractDetector, DetectorClassification
from slither.core.cfg.node import NodeType

class MyCustomDetector(AbstractDetector):
    ARGUMENT = "my-custom-detector"
    HELP = "Detect custom vulnerability pattern"
    IMPACT = DetectorClassification.HIGH
    CONFIDENCE = DetectorClassification.HIGH

    WIKI = "Custom detector for XYZ vulnerability"

    def _detect(self):
        results = []
        for contract in self.compilation_unit.contracts_derived:
            for function in contract.functions:
                # Check for specific pattern
                for node in function.nodes:
                    if self._check_vulnerability(node):
                        info = [function, " has vulnerability pattern\n"]
                        results.append(self.generate_result(info))
        return results
```

### Slither Triage Workflow

```bash
# 1. Run full analysis
slither . 2>&1 | tee slither-full.txt

# 2. Count findings by detector
slither . --json output.json
cat output.json | jq '.results.detectors | group_by(.check) | map({check: .[0].check, count: length})'

# 3. Focus on high-impact findings
slither . --detect reentrancy-eth,arbitrary-send-eth,controlled-delegatecall,suicidal

# 4. Review each finding — classify as:
#   - True positive → write up
#   - False positive → document reason
#   - Informational → note for completeness
```

---

## 5.4 Mythril — Symbolic Execution

### Installation & Usage

```bash
# Install via pip
pip3 install mythril

# Analyze a contract
myth analyze contracts/Vault.sol --solc-json mythril.config.json

# Analyze deployed contract
myth analyze --address 0xContract --rpc infura-mainnet

# Set execution depth
myth analyze contracts/Vault.sol --execution-timeout 300 --max-depth 50

# Check specific SWC
myth analyze contracts/Vault.sol --swc-filter 107,104,101
```

### Mythril Configuration

```json
// mythril.config.json
{
  "remappings": [
    "@openzeppelin/=lib/openzeppelin-contracts/"
  ]
}
```

### Limitations
- Slow for complex contracts (state space explosion)
- Struggles with loops and complex branching
- May timeout on large codebases
- Best for finding specific patterns: reentrancy, integer overflow, unchecked calls

---

## 5.5 Echidna — Property-Based Fuzzing

### Installation

```bash
# Download binary
wget https://github.com/crytic/echidna/releases/download/v2.2.3/echidna-2.2.3-linux.tar.gz
tar -xzf echidna-2.2.3-linux.tar.gz
sudo mv echidna /usr/local/bin/
```

### Writing Echidna Properties

```solidity
contract EchidnaTest {
    Token token;

    constructor() {
        token = new Token();
        token.mint(address(this), 1000e18);
    }

    // Echidna calls random functions, then checks all echidna_ properties
    // If any property returns false, Echidna reports a bug

    function echidna_total_supply_consistent() public view returns (bool) {
        return token.totalSupply() >= token.balanceOf(address(this));
    }

    function echidna_no_free_tokens() public view returns (bool) {
        return token.balanceOf(address(this)) <= 1000e18;
    }

    // Echidna will try to make these return false by calling
    // transfer(), approve(), burn(), etc. with random inputs
}
```

### Configuration

```yaml
# echidna.config.yaml
testMode: "assertion"    # or "property" or "optimization"
testLimit: 50000         # Number of test runs
seqLen: 100              # Max sequence length
shrinkLimit: 5000        # Shrink attempts for counterexample
deployer: "0x10000"
sender: ["0x10000", "0x20000", "0x30000"]
corpusDir: "echidna-corpus"
```

```bash
# Run Echidna
echidna . --contract EchidnaTest --config echidna.config.yaml

# With corpus management (saves interesting inputs)
echidna . --contract EchidnaTest --config echidna.config.yaml --corpus-dir corpus/
```

---

## 5.6 Certora Prover — Formal Verification

### Basics

Certora uses formal verification to **prove** that properties hold for ALL possible inputs, not just random ones.

### Writing CVL Specs

```cvl
// spec.cvl — Certora Verification Language
methods {
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function transfer(address, uint256) external returns (bool);
}

// Rule: Transfer preserves total supply
rule transferPreservesTotalSupply(address to, uint256 amount) {
    env e;
    uint256 supplyBefore = totalSupply();

    transfer(e, to, amount);

    uint256 supplyAfter = totalSupply();
    assert supplyAfter == supplyBefore, "Total supply changed after transfer!";
}

// Invariant: No user has more than total supply
invariant noUserExceedsTotalSupply(address user)
    balanceOf(user) <= totalSupply();
```

```bash
# Run Certora
certoraRun contracts/Token.sol --verify Token:specs/token.cvl \
  --solc solc-0.8.20 \
  --msg "Token transfer verification"
```

---

## 5.7 Manticore — Symbolic Execution

```bash
# Install
pip3 install manticore[native]

# Analyze a contract
manticore contracts/Vault.sol --contract Vault --thorough

# Explore all paths
manticore --contract Vault --workspace manticore-output
```

Manticore explores all execution paths symbolically. It's slower than Mythril but can go deeper on complex contracts.

---

## 5.8 Bytecode Analysis Tools

### Heimdall-rs

```bash
# Install
cargo install heimdall

# Decompile a contract
heimdall decompile 0xContractAddress --rpc-url $ETH_RPC

# Decode function selectors
heimdall decode 0xContractAddress --rpc-url $ETH_RPC

# Disassemble
heimdall disassemble 0xContractAddress --rpc-url $ETH_RPC
```

### EtherVM / Panoramix / Dedaub

- **EtherVM** ([ethervm.io](https://ethervm.io)): Paste bytecode → online decompilation
- **Dedaub** ([app.dedaub.com](https://app.dedaub.com)): Most accurate decompilation, handles edge cases
- **Panoramix**: Backend decompiler used by Etherscan's "Decompile" feature

### When to Use Bytecode Analysis

| Scenario | Recommended Tool |
|----------|-----------------|
| Quick function signature lookup | 4byte.directory, `cast 4byte` |
| Full decompilation of unverified contract | Dedaub, Heimdall-rs |
| Opcode-level debugging | Tenderly debugger, `cast run` |
| Compiler bug verification | Compare source → expected bytecode → actual bytecode |

---

## 5.9 Tenderly

### Transaction Simulation

```bash
# Tenderly CLI
tenderly export --project myproject 0xTxHash

# Simulate a transaction without broadcasting
tenderly simulate --from 0xSender --to 0xContract \
  --input 0xCalldata --value 0 --block 18500000
```

### Fork Testing via API

```javascript
// Create a Tenderly fork via API
const response = await fetch("https://api.tenderly.co/api/v1/account/USERNAME/project/PROJECT/fork", {
    method: "POST",
    headers: {
        "X-Access-Key": TENDERLY_API_KEY,
        "Content-Type": "application/json"
    },
    body: JSON.stringify({
        network_id: "1",
        block_number: 18500000
    })
});
const { simulation_fork } = await response.json();
// Use simulation_fork.rpc_url for testing
```

---

## 5.10 Phalcon / BlockSec

Phalcon is the go-to tool for **analyzing past exploit transactions**.

### Usage Workflow

1. Go to [phalcon.blocksec.com](https://phalcon.blocksec.com)
2. Enter the exploit transaction hash
3. View:
   - **Invocation flow**: Every internal call in tree format
   - **Balance changes**: Which addresses gained/lost tokens
   - **Fund flow**: Visual diagram of token movements
4. Use this to reverse-engineer the exploit for study or PoC recreation

---

## 5.11 Scripting with Web3.py / Ethers.js / Viem

### Ethers.js Example — Reading Contract State

```javascript
const { ethers } = require("ethers");

const provider = new ethers.JsonRpcProvider(process.env.ETH_RPC);

async function readState() {
    // Read storage directly
    const slot0 = await provider.getStorage("0xContract", 0);
    console.log("Slot 0:", slot0);

    // Call a function
    const contract = new ethers.Contract("0xContract", ABI, provider);
    const owner = await contract.owner();
    console.log("Owner:", owner);

    // Get transaction receipt
    const receipt = await provider.getTransactionReceipt("0xTxHash");
    console.log("Gas used:", receipt.gasUsed.toString());
}
```

### Web3.py Example — Monitoring Mempool

```python
from web3 import Web3

w3 = Web3(Web3.WebsocketProvider("wss://eth-mainnet.g.alchemy.com/v2/KEY"))

def handle_pending(tx_hash):
    tx = w3.eth.get_transaction(tx_hash)
    if tx and tx['to'] == '0xTargetContract':
        print(f"Pending tx to target: {tx_hash.hex()}")
        print(f"  From: {tx['from']}")
        print(f"  Value: {tx['value']}")
        print(f"  Input: {tx['input'][:10]}")  # Function selector

# Subscribe to pending transactions
pending_filter = w3.eth.filter('pending')
for tx_hash in pending_filter.get_new_entries():
    handle_pending(tx_hash)
```

---

## Tool Selection Decision Matrix

| Task | Primary Tool | Alternative |
|------|-------------|-------------|
| Writing exploit PoCs | **Foundry** (fork tests) | Hardhat |
| Static analysis (first pass) | **Slither** | Aderyn |
| Fuzzing (property-based) | **Echidna** / Foundry fuzz | Medusa |
| Symbolic execution | **Halmos** (Foundry-native) | Mythril, Manticore |
| Formal verification | **Certora** | Halmos (lighter) |
| Bytecode decompilation | **Dedaub** / **Heimdall** | EtherVM |
| Live chain interaction | **Cast** (Foundry) | ethers.js, web3.py |
| Transaction debugging | **Tenderly** | Cast run |
| Exploit analysis | **Phalcon** | Tenderly |
| Local chain forking | **Anvil** (Foundry) | Hardhat |

> **Key Takeaway:** Foundry is the single most important tool in your arsenal. If you master `forge`, `cast`, and `anvil`, you can handle 90% of pentesting tasks. Add Slither for automated detection and Echidna for deeper fuzzing. The remaining tools fill specific niches — learn them as needed for specific engagements.

---

*← [Previous: Audit Methodology](./AUDIT_METHODOLOGY.md) | [Next: Exploit Development →](./EXPLOIT_DEVELOPMENT.md)*
