# Implementation Plan: CRE Stablecoin Demo Workshop
## Step-by-Step Development Guide

**Version:** 1.0  
**Last Updated:** January 2025  
**Status:** Ready for Implementation  

---

## Overview

This document provides a detailed, step-by-step implementation plan to build the CRE Stablecoin Demo from scratch. Each phase is broken down into actionable tasks that should be completed sequentially.

---

## Phase 0: Project Setup & Environment Configuration

### Step 0.1: Repository Structure Setup
**Goal:** Create organized directory structure for contracts and workflows

**Tasks:**
- [ ] Create `contracts/` directory for Solidity smart contracts
- [ ] Create `contracts/src/` for contract source files
- [ ] Create `contracts/script/` for deployment scripts
- [ ] Create `contracts/test/` for Hardhat tests
- [ ] Create `workflow/` directory for CRE TypeScript workflow
- [ ] Create `docs/` directory for documentation

**Commands:**
```bash
cd /Users/woogieboogie/github/cre-stablecoin-demo
mkdir -p contracts/src contracts/script contracts/test
mkdir -p workflow/bank-mint-workflow
mkdir -p docs
```

**Verification:**
```bash
tree -L 2
```

---

### Step 0.2: Initialize Smart Contract Project
**Goal:** Set up Hardhat environment for Solidity development

**Tasks:**
- [ ] Initialize npm project in `contracts/`
- [ ] Install Hardhat and dependencies
- [ ] Install OpenZeppelin contracts
- [ ] Configure Hardhat for Sepolia deployment
- [ ] Set up .env.example template

**Commands:**
```bash
cd contracts
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npm install @openzeppelin/contracts
npx hardhat init  # Select "Create a TypeScript project"
```

**Files to Create:**
- `contracts/.env.example` - Template for environment variables
- `contracts/hardhat.config.ts` - Hardhat configuration

**Configuration:**
```typescript
// contracts/hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config();

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.19",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 11155111,
    },
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY || "",
  },
};

export default config;
```

**Verification:**
```bash
npx hardhat compile
```

---

### Step 0.3: Copy Base StablecoinERC20 Contract
**Goal:** Reuse existing audited stablecoin contract

**Tasks:**
- [ ] Copy `StablecoinERC20.sol` from stablecoin-workshop-evm
- [ ] Review contract interface (mint, grantMintRole)
- [ ] Verify OpenZeppelin dependencies match

**Commands:**
```bash
cp /Users/woogieboogie/github/stablecoin-workshop-evm/contracts/StablecoinERC20.sol \
   /Users/woogieboogie/github/cre-stablecoin-demo/contracts/src/
```

**Verification:**
```bash
cd contracts
npx hardhat compile
# Should compile without errors
```

---

## Phase 1: Smart Contract Development

### Step 1.1: Review Chainlink CRE Consumer Pattern
**Goal:** Understand IReceiverTemplate interface requirements

**Tasks:**
- [ ] Read CRE documentation on Consumer Contracts
- [ ] Study IReceiverTemplate interface
- [ ] Understand security validation flow
- [ ] Document required constructor parameters

**Reference Files to Review:**
- Chainlink CRE Docs: Consumer Contract Pattern
- Example: `CalculatorConsumer.sol` from Part 4 tutorial

**Key Concepts to Note:**
- `onReport(bytes metadata, bytes report)` - Entry point
- `_processReport(bytes report)` - Business logic override
- Workflow owner validation
- Workflow name validation
- Forwarder address validation

---

### Step 1.2: Create MintingConsumer Contract
**Goal:** Build the consumer contract that receives CRE reports

**Tasks:**
- [ ] Create `contracts/src/MintingConsumer.sol`
- [ ] Import IReceiverTemplate interface
- [ ] Define MintInstruction struct
- [ ] Implement constructor with validation
- [ ] Implement _processReport() override
- [ ] Add event definitions
- [ ] Add NatSpec documentation

**File to Create:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {IReceiverTemplate} from "./interfaces/IReceiverTemplate.sol";

/**
 * @title MintingConsumer
 * @notice Receives CRE workflow reports and triggers stablecoin minting
 * @dev Extends IReceiverTemplate for secure workflow authorization
 */
contract MintingConsumer is IReceiverTemplate {
    // Struct matching the CRE workflow report encoding
    struct MintInstruction {
        address recipient;
        uint256 amount;
        string bankReference;
    }

    // Immutable reference to the stablecoin contract
    address public immutable stablecoin;

    // Events
    event MintInstructionReceived(
        address indexed recipient,
        uint256 amount,
        string bankReference,
        uint256 timestamp
    );

    /**
     * @notice Constructor
     * @param _stablecoin Address of StablecoinERC20 contract
     * @param expectedWorkflowOwner CRE workflow owner address
     * @param expectedWorkflowName Expected workflow name (bytes10)
     */
    constructor(
        address _stablecoin,
        address expectedWorkflowOwner,
        bytes10 expectedWorkflowName
    ) IReceiverTemplate(expectedWorkflowOwner, expectedWorkflowName) {
        require(_stablecoin != address(0), "Invalid stablecoin address");
        stablecoin = _stablecoin;
    }

    /**
     * @notice Process incoming CRE report
     * @param report ABI-encoded MintInstruction struct
     */
    function _processReport(bytes calldata report) internal override {
        // Decode the report
        MintInstruction memory instruction = abi.decode(
            report,
            (MintInstruction)
        );

        // Validate
        require(instruction.recipient != address(0), "Invalid recipient");
        require(instruction.amount > 0, "Invalid amount");

        // Call stablecoin.mint()
        (bool success, ) = stablecoin.call(
            abi.encodeWithSignature(
                "mint(address,uint256)",
                instruction.recipient,
                instruction.amount
            )
        );
        require(success, "Mint failed");

        // Emit event
        emit MintInstructionReceived(
            instruction.recipient,
            instruction.amount,
            instruction.bankReference,
            block.timestamp
        );
    }
}
```

**Verification:**
```bash
cd contracts
npx hardhat compile
# Should compile without errors
```

---

### Step 1.3: Obtain IReceiverTemplate Interface
**Goal:** Get the Chainlink CRE interface files

**Tasks:**
- [ ] Check if CRE provides npm package for interfaces
- [ ] If not, manually create interface from CRE docs
- [ ] Create `contracts/src/interfaces/` directory
- [ ] Add IReceiverTemplate.sol
- [ ] Add IERC165.sol if needed

**File to Create:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {IERC165} from "@openzeppelin/contracts/utils/introspection/IERC165.sol";

/**
 * @title IReceiverTemplate
 * @notice Base contract for CRE workflow consumers
 */
abstract contract IReceiverTemplate is IERC165 {
    address public immutable expectedWorkflowOwner;
    bytes10 public immutable expectedWorkflowName;

    error UnauthorizedWorkflowOwner(address actualOwner);
    error UnauthorizedWorkflowName(bytes10 actualWorkflowName);

    constructor(address _expectedWorkflowOwner, bytes10 _expectedWorkflowName) {
        expectedWorkflowOwner = _expectedWorkflowOwner;
        expectedWorkflowName = _expectedWorkflowName;
    }

    /**
     * @notice Called by KeystoneForwarder to deliver reports
     * @param metadata Contains workflow metadata for validation
     * @param report The actual report data
     */
    function onReport(
        bytes calldata metadata,
        bytes calldata report
    ) external virtual {
        // Decode metadata (simplified - actual implementation may vary)
        (address workflowOwner, bytes10 workflowName) = abi.decode(
            metadata,
            (address, bytes10)
        );

        // Validate workflow identity
        if (workflowOwner != expectedWorkflowOwner) {
            revert UnauthorizedWorkflowOwner(workflowOwner);
        }
        if (workflowName != expectedWorkflowName) {
            revert UnauthorizedWorkflowName(workflowName);
        }

        // Process the report
        _processReport(report);
    }

    /**
     * @notice Override this function with your business logic
     * @param report The decoded report data
     */
    function _processReport(bytes calldata report) internal virtual;

    /**
     * @notice ERC165 support
     */
    function supportsInterface(
        bytes4 interfaceId
    ) public view virtual override returns (bool) {
        return interfaceId == type(IERC165).interfaceId;
    }
}
```

**Note:** The actual IReceiverTemplate may differ. Check CRE documentation for the canonical version.

---

### Step 1.4: Write Contract Unit Tests
**Goal:** Ensure contracts work correctly before deployment

**Tasks:**
- [ ] Create `contracts/test/MintingConsumer.test.ts`
- [ ] Test: Deploy contracts successfully
- [ ] Test: Authorized workflow can mint
- [ ] Test: Unauthorized workflow is rejected
- [ ] Test: Invalid amounts are rejected
- [ ] Test: Event emission is correct

**File to Create:**
```typescript
// contracts/test/MintingConsumer.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";

describe("MintingConsumer", function () {
  let stablecoin: any;
  let consumer: any;
  let owner: any;
  let workflowOwner: any;
  let recipient: any;

  const WORKFLOW_NAME = ethers.utils.formatBytes32String("BankMint").slice(0, 22); // bytes10

  beforeEach(async function () {
    [owner, workflowOwner, recipient] = await ethers.getSigners();

    // Deploy StablecoinERC20
    const Stablecoin = await ethers.getContractFactory("StablecoinERC20");
    stablecoin = await Stablecoin.deploy(
      "USD Coin",
      "USDC",
      ethers.constants.MaxUint256
    );
    await stablecoin.deployed();

    // Deploy MintingConsumer
    const Consumer = await ethers.getContractFactory("MintingConsumer");
    consumer = await Consumer.deploy(
      stablecoin.address,
      workflowOwner.address,
      WORKFLOW_NAME
    );
    await consumer.deployed();

    // Grant minter role to consumer
    await stablecoin.grantMintRole(consumer.address);
  });

  describe("Deployment", function () {
    it("Should set correct stablecoin address", async function () {
      expect(await consumer.stablecoin()).to.equal(stablecoin.address);
    });

    it("Should set correct workflow owner", async function () {
      expect(await consumer.expectedWorkflowOwner()).to.equal(
        workflowOwner.address
      );
    });
  });

  describe("Processing Reports", function () {
    it("Should mint tokens when valid report received", async function () {
      const mintAmount = ethers.utils.parseEther("1000");
      const bankRef = "BNKUS33XXXX202501151030001";

      // Encode MintInstruction
      const report = ethers.utils.defaultAbiCoder.encode(
        ["address", "uint256", "string"],
        [recipient.address, mintAmount, bankRef]
      );

      // Encode metadata (simplified)
      const metadata = ethers.utils.defaultAbiCoder.encode(
        ["address", "bytes10"],
        [workflowOwner.address, WORKFLOW_NAME]
      );

      // Call onReport as if from Forwarder
      await expect(consumer.onReport(metadata, report))
        .to.emit(consumer, "MintInstructionReceived")
        .withArgs(recipient.address, mintAmount, bankRef, /* timestamp */);

      // Verify balance
      expect(await stablecoin.balanceOf(recipient.address)).to.equal(
        mintAmount
      );
    });

    it("Should reject unauthorized workflow owner", async function () {
      const mintAmount = ethers.utils.parseEther("1000");
      const report = ethers.utils.defaultAbiCoder.encode(
        ["address", "uint256", "string"],
        [recipient.address, mintAmount, "REF123"]
      );

      // Wrong owner in metadata
      const metadata = ethers.utils.defaultAbiCoder.encode(
        ["address", "bytes10"],
        [owner.address, WORKFLOW_NAME] // owner instead of workflowOwner
      );

      await expect(consumer.onReport(metadata, report)).to.be.revertedWith(
        "UnauthorizedWorkflowOwner"
      );
    });
  });
});
```

**Run Tests:**
```bash
cd contracts
npx hardhat test
```

---

### Step 1.5: Create Deployment Scripts
**Goal:** Automate contract deployment to Sepolia

**Tasks:**
- [ ] Create `contracts/script/deploy.ts`
- [ ] Handle environment variable loading
- [ ] Deploy StablecoinERC20
- [ ] Deploy MintingConsumer
- [ ] Grant minter role
- [ ] Save deployment addresses
- [ ] Verify contracts on Etherscan

**File to Create:**
```typescript
// contracts/script/deploy.ts
import { ethers } from "hardhat";
import * as fs from "fs";
import * as path from "path";

async function main() {
  console.log("🚀 Starting deployment to Sepolia...\n");

  const [deployer] = await ethers.getSigners();
  console.log("Deploying contracts with account:", deployer.address);
  console.log("Account balance:", ethers.utils.formatEther(await deployer.getBalance()), "ETH\n");

  // Configuration
  const TOKEN_NAME = "Bank Stablecoin";
  const TOKEN_SYMBOL = "BUSD";
  const MAX_SUPPLY = ethers.constants.MaxUint256;
  const WORKFLOW_OWNER = process.env.WORKFLOW_OWNER_ADDRESS || deployer.address;
  const WORKFLOW_NAME = ethers.utils.formatBytes32String("BankMint").slice(0, 22); // bytes10

  console.log("Configuration:");
  console.log("- Token Name:", TOKEN_NAME);
  console.log("- Token Symbol:", TOKEN_SYMBOL);
  console.log("- Workflow Owner:", WORKFLOW_OWNER);
  console.log("- Workflow Name:", ethers.utils.parseBytes32String(WORKFLOW_NAME + "0".repeat(44)));

  // Deploy StablecoinERC20
  console.log("\n📝 Deploying StablecoinERC20...");
  const Stablecoin = await ethers.getContractFactory("StablecoinERC20");
  const stablecoin = await Stablecoin.deploy(TOKEN_NAME, TOKEN_SYMBOL, MAX_SUPPLY);
  await stablecoin.deployed();
  console.log("✅ StablecoinERC20 deployed to:", stablecoin.address);

  // Deploy MintingConsumer
  console.log("\n📝 Deploying MintingConsumer...");
  const Consumer = await ethers.getContractFactory("MintingConsumer");
  const consumer = await Consumer.deploy(
    stablecoin.address,
    WORKFLOW_OWNER,
    WORKFLOW_NAME
  );
  await consumer.deployed();
  console.log("✅ MintingConsumer deployed to:", consumer.address);

  // Grant minter role
  console.log("\n🔑 Granting minter role to MintingConsumer...");
  const tx = await stablecoin.grantMintRole(consumer.address);
  await tx.wait();
  console.log("✅ Minter role granted");

  // Save deployment info
  const deploymentInfo = {
    network: "sepolia",
    deployer: deployer.address,
    timestamp: new Date().toISOString(),
    contracts: {
      StablecoinERC20: {
        address: stablecoin.address,
        name: TOKEN_NAME,
        symbol: TOKEN_SYMBOL,
      },
      MintingConsumer: {
        address: consumer.address,
        workflowOwner: WORKFLOW_OWNER,
        workflowName: ethers.utils.parseBytes32String(WORKFLOW_NAME + "0".repeat(44)),
      },
    },
  };

  const outputPath = path.join(__dirname, "../deployment-sepolia.json");
  fs.writeFileSync(outputPath, JSON.stringify(deploymentInfo, null, 2));
  console.log("\n💾 Deployment info saved to:", outputPath);

  // Print .env updates
  console.log("\n📋 Add these to your workflow .env file:");
  console.log(`STABLECOIN_ADDRESS=${stablecoin.address}`);
  console.log(`MINTING_CONSUMER_ADDRESS=${consumer.address}`);

  console.log("\n🎉 Deployment complete!");
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

**Run Deployment:**
```bash
cd contracts
npx hardhat run script/deploy.ts --network sepolia
```

---

## Phase 2: CRE Workflow Development

### Step 2.1: Initialize CRE Workflow Project
**Goal:** Set up CRE TypeScript workflow structure

**Tasks:**
- [ ] Navigate to workflow directory
- [ ] Run `cre init` for TypeScript workflow
- [ ] Select "Boilerplate: TypeScript HTTP Trigger"
- [ ] Install dependencies with `bun install`

**Commands:**
```bash
cd /Users/woogieboogie/github/cre-stablecoin-demo/workflow
cre init
# Project name: bank-mint-workflow
# Language: TypeScript
# Template: Boilerplate: Typescript Hello World example
# Workflow name: bank-mint-workflow

cd bank-mint-workflow
bun install
```

---

### Step 2.2: Configure Workflow Files
**Goal:** Set up configuration and secrets for the workflow

**Tasks:**
- [ ] Create `config.json` with chain and contract details
- [ ] Update `secrets.yaml` for private key
- [ ] Create `.env.example` template
- [ ] Update `project.yaml` with Sepolia RPC

**Files to Create/Update:**

**`workflow/bank-mint-workflow/config.json`:**
```json
{
  "workflowName": "BankMint",
  "chainName": "ethereum-testnet-sepolia",
  "mintingConsumerAddress": "0xYOUR_CONSUMER_ADDRESS_HERE",
  "gasLimit": "500000",
  "httpEndpoint": "/api/mint-instruction"
}
```

**`workflow/secrets.yaml`:**
```yaml
secretsNames:
  PRIVATE_KEY:
    - CRE_ETH_PRIVATE_KEY
```

**`workflow/.env.example`:**
```bash
# Ethereum private key (64 hex characters, no 0x prefix)
CRE_ETH_PRIVATE_KEY=your_private_key_here

# Target environment
CRE_TARGET=local-simulation
```

**`workflow/project.yaml`:**
```yaml
local-simulation:
  rpcs:
    - chain-name: ethereum-testnet-sepolia
      url: YOUR_SEPOLIA_RPC_URL
```

---

### Step 2.3: Define TypeScript Types
**Goal:** Create type-safe interfaces for SWIFT messages

**Tasks:**
- [ ] Create `workflow/bank-mint-workflow/types.ts`
- [ ] Define SwiftMT103Message interface
- [ ] Define Config interface
- [ ] Define error codes enum

**File to Create:**
```typescript
// workflow/bank-mint-workflow/types.ts

/**
 * SWIFT MT103-inspired message structure
 */
export interface SwiftMT103Message {
  messageType: "MT103";
  transactionReference: string;
  instructionType: "MINT";
  valueDate: string;
  currency: "USD";
  amount: string;
  orderingInstitution: {
    bicCode: string;
    name: string;
    country: string;
  };
  beneficiary: {
    name: string;
    accountNumber: string;
    blockchainAddress: string;
  };
  remittanceInformation?: string;
  senderToReceiverInfo?: string;
  regulatoryReporting?: string;
  timestamp: string;
  messageHash?: string;
}

/**
 * Workflow configuration
 */
export interface Config {
  workflowName: string;
  chainName: string;
  mintingConsumerAddress: string;
  gasLimit: string;
  httpEndpoint: string;
}

/**
 * Mint instruction for CRE report encoding
 */
export interface MintInstruction {
  recipient: string;
  amount: bigint;
  bankReference: string;
}

/**
 * Success response
 */
export interface MintResponse {
  status: "SUCCESS";
  transactionReference: string;
  chainlinkTransactionHash: string;
  recipientAddress: string;
  mintedAmount: string;
  mintedAmountFormatted: string;
  network: string;
  blockNumber: number;
  timestamp: string;
  explorerUrl: string;
}

/**
 * Error response
 */
export interface ErrorResponse {
  status: "ERROR";
  errorCode: string;
  errorMessage: string;
  transactionReference: string;
  timestamp: string;
  details?: any;
}

/**
 * Error codes
 */
export enum ErrorCode {
  INVALID_ADDRESS = "E001",
  INVALID_AMOUNT = "E002",
  MISSING_FIELD = "E003",
  TRANSACTION_FAILED = "E004",
  INSUFFICIENT_GAS = "E005",
  NETWORK_ERROR = "E006",
  DUPLICATE_REFERENCE = "E007",
  WORKFLOW_AUTH_FAILED = "E008",
}
```

---

### Step 2.4: Implement Validation Logic
**Goal:** Create robust input validation

**Tasks:**
- [ ] Create `workflow/bank-mint-workflow/validation.ts`
- [ ] Implement address validation
- [ ] Implement amount validation
- [ ] Implement required fields check
- [ ] Implement timestamp validation

**File to Create:**
```typescript
// workflow/bank-mint-workflow/validation.ts
import { SwiftMT103Message } from "./types";
import { ErrorCode } from "./types";

/**
 * Validate EVM address format
 */
export function isValidEvmAddress(address: string): boolean {
  return /^0x[a-fA-F0-9]{40}$/.test(address);
}

/**
 * Validate amount format and value
 */
export function isValidAmount(amount: string): boolean {
  const regex = /^\d+(\.\d{1,2})?$/;
  if (!regex.test(amount)) return false;
  
  const numAmount = parseFloat(amount);
  return numAmount > 0;
}

/**
 * Validate timestamp is recent (within 5 minutes)
 */
export function isRecentTimestamp(timestamp: string): boolean {
  const msgTime = new Date(timestamp).getTime();
  const now = Date.now();
  const fiveMinutes = 5 * 60 * 1000;
  
  return (now - msgTime) < fiveMinutes && msgTime <= now;
}

/**
 * Validate complete SWIFT message
 */
export function validateSwiftMessage(
  msg: SwiftMT103Message
): { valid: boolean; errorCode?: string; errorMessage?: string; field?: string } {
  // Check message type
  if (msg.messageType !== "MT103") {
    return {
      valid: false,
      errorCode: ErrorCode.MISSING_FIELD,
      errorMessage: "Invalid message type",
      field: "messageType",
    };
  }

  // Check required fields
  const requiredFields = [
    "transactionReference",
    "instructionType",
    "valueDate",
    "currency",
    "amount",
    "timestamp",
  ];

  for (const field of requiredFields) {
    if (!(field in msg) || !msg[field as keyof SwiftMT103Message]) {
      return {
        valid: false,
        errorCode: ErrorCode.MISSING_FIELD,
        errorMessage: `Missing required field: ${field}`,
        field,
      };
    }
  }

  // Validate blockchain address
  if (!isValidEvmAddress(msg.beneficiary.blockchainAddress)) {
    return {
      valid: false,
      errorCode: ErrorCode.INVALID_ADDRESS,
      errorMessage: "Invalid blockchain address format",
      field: "beneficiary.blockchainAddress",
    };
  }

  // Validate amount
  if (!isValidAmount(msg.amount)) {
    return {
      valid: false,
      errorCode: ErrorCode.INVALID_AMOUNT,
      errorMessage: "Invalid amount format or value",
      field: "amount",
    };
  }

  // Validate currency
  if (msg.currency !== "USD") {
    return {
      valid: false,
      errorCode: ErrorCode.INVALID_AMOUNT,
      errorMessage: "Only USD currency supported",
      field: "currency",
    };
  }

  // Validate timestamp
  if (!isRecentTimestamp(msg.timestamp)) {
    return {
      valid: false,
      errorCode: ErrorCode.MISSING_FIELD,
      errorMessage: "Timestamp too old or in future",
      field: "timestamp",
    };
  }

  return { valid: true };
}

/**
 * Convert USD string to wei (18 decimals)
 */
export function usdToWei(usdAmount: string): bigint {
  // Parse USD amount (e.g., "1000.00")
  const [dollars, cents = "00"] = usdAmount.split(".");
  const paddedCents = cents.padEnd(2, "0").slice(0, 2);
  
  // Convert to smallest unit (cents)
  const totalCents = BigInt(dollars) * 100n + BigInt(paddedCents);
  
  // Convert to wei (multiply by 10^16 to get 18 decimals from 2)
  const wei = totalCents * (10n ** 16n);
  
  return wei;
}
```

---

### Step 2.5: Implement Main Workflow Logic
**Goal:** Build the core CRE workflow handler

**Tasks:**
- [ ] Update `workflow/bank-mint-workflow/main.ts`
- [ ] Implement HTTP trigger handler
- [ ] Implement message parsing and validation
- [ ] Implement report generation
- [ ] Implement blockchain submission
- [ ] Implement error handling
- [ ] Add comprehensive logging

**File to Update:**
```typescript
// workflow/bank-mint-workflow/main.ts
import {
  cre,
  Runner,
  type Runtime,
  getNetwork,
  encodeCallMsg,
  bytesToHex,
  hexToBase64,
} from "@chainlink/cre-sdk";

import { encodeAbiParameters, parseAbiParameters } from "viem";

import type {
  Config,
  SwiftMT103Message,
  MintInstruction,
  MintResponse,
  ErrorResponse,
} from "./types";
import { ErrorCode } from "./types";
import { validateSwiftMessage, usdToWei } from "./validation";

/**
 * Initialize workflow with HTTP trigger
 */
const initWorkflow = (config: Config) => {
  const httpTrigger = new cre.capabilities.HTTPCapability();
  
  return [
    cre.handler(
      httpTrigger.trigger({ path: config.httpEndpoint }),
      onHttpTrigger
    ),
  ];
};

/**
 * HTTP trigger handler - main entry point
 */
const onHttpTrigger = (
  runtime: Runtime<Config>
): MintResponse | ErrorResponse => {
  runtime.log("🏦 Bank Minting Workflow triggered via HTTP");
  
  try {
    // Step 1: Parse HTTP payload
    const httpPayload = runtime.trigger.payload;
    const swiftMessage: SwiftMT103Message = JSON.parse(
      new TextDecoder().decode(httpPayload)
    );
    
    runtime.log(`📨 Received message: ${swiftMessage.transactionReference}`);
    
    // Step 2: Validate message
    const validation = validateSwiftMessage(swiftMessage);
    if (!validation.valid) {
      runtime.log(`❌ Validation failed: ${validation.errorMessage}`);
      return createErrorResponse(
        swiftMessage.transactionReference,
        validation.errorCode!,
        validation.errorMessage!,
        { field: validation.field }
      );
    }
    
    runtime.log("✅ Message validation passed");
    
    // Step 3: Convert amount
    const amountWei = usdToWei(swiftMessage.amount);
    runtime.log(`💰 Amount: ${swiftMessage.amount} USD = ${amountWei} wei`);
    
    // Step 4: Prepare mint instruction
    const mintInstruction: MintInstruction = {
      recipient: swiftMessage.beneficiary.blockchainAddress as `0x${string}`,
      amount: amountWei,
      bankReference: swiftMessage.transactionReference,
    };
    
    // Step 5: Execute onchain mint
    const txHash = executeMint(runtime, mintInstruction);
    
    runtime.log(`🎉 Minting successful! TxHash: ${txHash}`);
    
    // Step 6: Return success response
    return createSuccessResponse(
      swiftMessage,
      mintInstruction,
      txHash
    );
    
  } catch (error: any) {
    runtime.log(`💥 Error: ${error.message}`);
    return createErrorResponse(
      "UNKNOWN",
      ErrorCode.TRANSACTION_FAILED,
      error.message
    );
  }
};

/**
 * Execute the onchain minting operation
 */
function executeMint(
  runtime: Runtime<Config>,
  instruction: MintInstruction
): string {
  runtime.log("📝 Generating DON-signed report...");
  
  // Get network configuration
  const network = getNetwork({
    chainFamily: "evm",
    chainSelectorName: runtime.config.chainName,
    isTestnet: true,
  });
  
  if (!network) {
    throw new Error(`Unknown chain: ${runtime.config.chainName}`);
  }
  
  const evmClient = new cre.capabilities.EVMClient(
    network.chainSelector.selector
  );
  
  // Encode the MintInstruction struct
  const reportData = encodeAbiParameters(
    parseAbiParameters("address recipient, uint256 amount, string bankReference"),
    [instruction.recipient, instruction.amount, instruction.bankReference]
  );
  
  runtime.log("🔐 Requesting DON signatures...");
  
  // Generate signed report
  const reportResponse = runtime
    .report({
      encodedPayload: hexToBase64(reportData),
      encoderName: "evm",
      signingAlgo: "ecdsa",
      hashingAlgo: "keccak256",
    })
    .result();
  
  runtime.log("⛓️  Submitting to blockchain...");
  
  // Submit to MintingConsumer contract
  const writeReportResult = evmClient
    .writeReport(runtime, {
      receiver: runtime.config.mintingConsumerAddress,
      report: reportResponse,
      gasConfig: {
        gasLimit: runtime.config.gasLimit,
      },
    })
    .result();
  
  const txHash = bytesToHex(writeReportResult.txHash || new Uint8Array(32));
  
  return txHash;
}

/**
 * Create success response
 */
function createSuccessResponse(
  swiftMsg: SwiftMT103Message,
  instruction: MintInstruction,
  txHash: string
): MintResponse {
  return {
    status: "SUCCESS",
    transactionReference: swiftMsg.transactionReference,
    chainlinkTransactionHash: txHash,
    recipientAddress: instruction.recipient,
    mintedAmount: instruction.amount.toString(),
    mintedAmountFormatted: swiftMsg.amount,
    network: "sepolia",
    blockNumber: 0, // Would need to fetch from receipt
    timestamp: new Date().toISOString(),
    explorerUrl: `https://sepolia.etherscan.io/tx/${txHash}`,
  };
}

/**
 * Create error response
 */
function createErrorResponse(
  transactionRef: string,
  errorCode: string,
  errorMessage: string,
  details?: any
): ErrorResponse {
  return {
    status: "ERROR",
    errorCode,
    errorMessage,
    transactionReference: transactionRef,
    timestamp: new Date().toISOString(),
    details,
  };
}

/**
 * Main entry point
 */
export async function main() {
  const runner = await Runner.newRunner<Config>();
  await runner.run(initWorkflow);
}

main();
```

---

### Step 2.6: Create ABI Files
**Goal:** Define contract ABIs for MintingConsumer

**Tasks:**
- [ ] Create `workflow/contracts/abi/` directory
- [ ] Create `MintingConsumer.ts` with ABI
- [ ] Create `index.ts` for exports

**Files to Create:**

**`workflow/contracts/abi/MintingConsumer.ts`:**
```typescript
export const MintingConsumer = [
  {
    inputs: [
      { internalType: "address", name: "_stablecoin", type: "address" },
      { internalType: "address", name: "expectedWorkflowOwner", type: "address" },
      { internalType: "bytes10", name: "expectedWorkflowName", type: "bytes10" },
    ],
    stateMutability: "nonpayable",
    type: "constructor",
  },
  {
    anonymous: false,
    inputs: [
      { indexed: true, internalType: "address", name: "recipient", type: "address" },
      { indexed: false, internalType: "uint256", name: "amount", type: "uint256" },
      { indexed: false, internalType: "string", name: "bankReference", type: "string" },
      { indexed: false, internalType: "uint256", name: "timestamp", type: "uint256" },
    ],
    name: "MintInstructionReceived",
    type: "event",
  },
  {
    inputs: [
      { internalType: "bytes", name: "metadata", type: "bytes" },
      { internalType: "bytes", name: "report", type: "bytes" },
    ],
    name: "onReport",
    outputs: [],
    stateMutability: "nonpayable",
    type: "function",
  },
] as const;
```

---

## Phase 3: Testing & Integration

### Step 3.1: Local Workflow Simulation
**Goal:** Test workflow locally without HTTP trigger

**Tasks:**
- [ ] Create test payload JSON file
- [ ] Run `cre workflow simulate` with test data
- [ ] Verify compilation succeeds
- [ ] Verify validation logic works
- [ ] Verify encoding is correct

**Commands:**
```bash
cd workflow
cre workflow simulate bank-mint-workflow --target local-simulation
```

---

### Step 3.2: HTTP Trigger Testing with Webhook.site
**Goal:** Test end-to-end HTTP → Blockchain flow

**Tasks:**
- [ ] Go to webhook.site and get unique URL
- [ ] Configure workflow to log responses
- [ ] Send test HTTP POST with valid payload
- [ ] Verify workflow receives request
- [ ] Verify blockchain transaction
- [ ] Check Sepolia Etherscan for mint

**Test Payload:**
```json
{
  "messageType": "MT103",
  "transactionReference": "BNKUS33XXXX202501220001",
  "instructionType": "MINT",
  "valueDate": "2025-01-22",
  "currency": "USD",
  "amount": "1000.00",
  "orderingInstitution": {
    "bicCode": "BNKUS33XXXX",
    "name": "Test Bank N.A.",
    "country": "US"
  },
  "beneficiary": {
    "name": "Alice Testing",
    "accountNumber": "TEST001",
    "blockchainAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
  },
  "remittanceInformation": "Test minting operation",
  "timestamp": "2025-01-22T10:00:00.000Z"
}
```

---

### Step 3.3: Error Scenario Testing
**Goal:** Verify error handling works correctly

**Test Cases:**
- [ ] Invalid address format
- [ ] Negative amount
- [ ] Missing required field
- [ ] Wrong currency
- [ ] Old timestamp
- [ ] Duplicate transaction reference

**For Each Test:**
1. Send invalid payload
2. Verify error response matches expected format
3. Verify no onchain transaction occurred
4. Document results

---

## Phase 4: Documentation

### Step 4.1: Create Workshop Instructions
**Goal:** Step-by-step guide for participants

**File to Create:** `docs/WORKSHOP_INSTRUCTIONS.md`

**Sections:**
- Prerequisites checklist
- Environment setup
- Contract deployment walkthrough
- Workflow setup walkthrough
- Testing guide
- Troubleshooting common issues

---

### Step 4.2: Create Architecture Documentation
**Goal:** Explain system design and security

**File to Create:** `docs/ARCHITECTURE.md`

**Sections:**
- Component diagram
- Data flow sequence diagrams
- Security model explanation
- DON signature verification flow
- Consumer contract authorization

---

### Step 4.3: Create API Reference
**Goal:** Document HTTP API for banking systems

**File to Create:** `docs/API_REFERENCE.md`

**Sections:**
- Endpoint specification
- Request format with examples
- Response format with examples
- Error codes reference
- Rate limiting (if applicable)

---

## Phase 5: Workshop Preparation

### Step 5.1: Create Participant Checklist
**Goal:** Pre-workshop setup instructions

**Tasks:**
- [ ] List required software installations
- [ ] Provide Sepolia faucet links
- [ ] Create RPC URL signup instructions
- [ ] Provide test wallet setup guide

---

### Step 5.2: Prepare Demo Scripts
**Goal:** Instructor demonstration materials

**Tasks:**
- [ ] Create demo deployment script
- [ ] Create demo HTTP requests (Postman collection)
- [ ] Prepare Etherscan links for live demo
- [ ] Create backup deployment in case of issues

---

### Step 5.3: Test Complete Workshop Flow
**Goal:** Ensure workshop can be completed end-to-end

**Tasks:**
- [ ] Fresh environment test (new machine if possible)
- [ ] Time each phase
- [ ] Document any blockers
- [ ] Prepare solutions for common errors
- [ ] Create FAQ document

---

## Phase 6: Optional Enhancements

### Step 6.1: Add Idempotency Check
**Goal:** Prevent duplicate transaction references

**Implementation:**
- [ ] Add mapping in MintingConsumer
- [ ] Check transactionReference before minting
- [ ] Emit duplicate detection event
- [ ] Update tests

---

### Step 6.2: Add Pause Functionality
**Goal:** Emergency stop mechanism

**Implementation:**
- [ ] Add Pausable from OpenZeppelin
- [ ] Add pause/unpause functions
- [ ] Add whenNotPaused modifier to _processReport
- [ ] Update tests

---

### Step 6.3: Create Monitoring Dashboard
**Goal:** Real-time visibility into minting operations

**Implementation:**
- [ ] Set up event indexing (The Graph or similar)
- [ ] Create simple frontend dashboard
- [ ] Display recent mints
- [ ] Show success/failure rates

---

## Success Criteria Checklist

### Functional Requirements
- [ ] Smart contracts compile without errors
- [ ] All contract tests pass
- [ ] Workflow compiles to WASM successfully
- [ ] HTTP trigger receives and parses messages
- [ ] Validation rejects invalid inputs
- [ ] DON-signed reports are generated
- [ ] Onchain minting succeeds
- [ ] Events are emitted correctly
- [ ] Unauthorized workflows are rejected

### Documentation Requirements
- [ ] README is complete and clear
- [ ] Workshop instructions are step-by-step
- [ ] Architecture is documented
- [ ] API reference is complete
- [ ] Troubleshooting guide exists

### Testing Requirements
- [ ] Unit tests for contracts pass
- [ ] Integration test passes end-to-end
- [ ] Error scenarios are tested
- [ ] Security tests pass

---

## Timeline Estimate

| Phase | Estimated Time | Complexity |
|-------|---------------|------------|
| Phase 0: Project Setup | 2 hours | Low |
| Phase 1: Smart Contracts | 8 hours | Medium |
| Phase 2: CRE Workflow | 12 hours | High |
| Phase 3: Testing | 6 hours | Medium |
| Phase 4: Documentation | 8 hours | Medium |
| Phase 5: Workshop Prep | 4 hours | Low |
| **Total** | **40 hours** | - |

---

## Next Immediate Steps

**Start Here:**
1. ✅ Review and approve this implementation plan
2. 🎯 Begin Phase 0: Project Setup (Steps 0.1-0.3)
3. 📋 Create tracking issues for each phase
4. 🚀 Start development!

---

**Ready to begin implementation?** Let me know if you'd like me to start with Phase 0, or if you have any questions or adjustments to this plan!

