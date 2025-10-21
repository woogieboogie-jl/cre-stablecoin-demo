# Product Requirements Document: CRE Stablecoin Demo
## Chainlink Runtime Environment (CRE) - TradFi to Web3 Bridge

**Version:** 1.0  
**Last Updated:** January 2025  
**Status:** Requirements Definition  

---

## 1. Executive Summary

### 1.1 Purpose
This document defines the requirements for a demonstration system that showcases Chainlink Runtime Environment (CRE) as a bridge between traditional financial infrastructure and the multi-chain Web3 ecosystem.

### 1.2 Project Goal
Build an educational demonstration that illustrates how a financial institution (e.g., a bank) can use CRE workflows to issue and manage an on-chain stablecoin, triggered by traditional banking systems and secured by Chainlink's Decentralized Oracle Network (DON).

### 1.3 Target Audience
- Financial institutions exploring blockchain integration
- Web3 developers learning CRE
- Workshop participants
- Technical decision-makers evaluating oracle solutions

---

## 2. System Overview

### 2.1 Narrative Context
The demo operates from the perspective of a **commercial bank** that wants to:
1. Issue a blockchain-based stablecoin backed by their reserves
2. Enable minting through existing legacy systems
3. Maintain security and compliance through decentralized verification
4. Operate across multiple blockchain networks

**The CRE workflow acts as the secure, verifiable bridge between these worlds.**

### 2.2 High-Level Architecture

```
┌────────────────────────────────────────────────────────────┐
│              TRADITIONAL BANKING SYSTEM                     │
│  - Core Banking Platform                                    │
│  - Payment Processing System                                │
│  - SWIFT Integration Layer                                  │
└─────────────────────┬──────────────────────────────────────┘
                      │ HTTP POST (SWIFT MT103-style message)
                      ↓
┌────────────────────────────────────────────────────────────┐
│           CHAINLINK RUNTIME ENVIRONMENT (CRE)              │
│  Single Point of Contact - Secure Workflow Execution       │
│                                                             │
│  • Receives banking instructions                           │
│  • Validates and processes requests                        │
│  • Generates DON-signed reports                            │
│  • Executes multi-chain operations                         │
└─────────────────────┬──────────────────────────────────────┘
                      │ Signed Reports / Transactions
                      ↓
┌────────────────────────────────────────────────────────────┐
│               BLOCKCHAIN LAYER (Sepolia Testnet)           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  MintingConsumer.sol                                 │ │
│  │  - Receives & validates CRE reports                  │ │
│  │  - Enforces workflow authorization                   │ │
│  └────────────────────┬─────────────────────────────────┘ │
│                       │                                     │
│  ┌────────────────────▼─────────────────────────────────┐ │
│  │  StablecoinERC20.sol                                 │ │
│  │  - Role-based minting (onlyMinter)                   │ │
│  │  - ERC20 standard token                              │ │
│  │  - CCIP-compatible (future)                          │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

---

## 3. Core Requirements

### 3.1 Functional Requirements

#### FR-1: Legacy System Integration
**Priority:** P0 (Must Have)

**Description:** The system must accept minting instructions from simulated traditional banking systems.

**Requirements:**
- FR-1.1: Accept HTTP POST requests as trigger mechanism
- FR-1.2: Parse SWIFT MT103-style message format (JSON representation)
- FR-1.3: Support standardized banking instruction fields
- FR-1.4: Validate message structure and required fields
- FR-1.5: Return acknowledgment with transaction reference

**Acceptance Criteria:**
- System successfully receives HTTP POST request
- Message is parsed without errors
- Invalid messages are rejected with clear error codes
- Valid messages proceed to minting workflow

---

#### FR-2: Secure Minting Workflow
**Priority:** P0 (Must Have)

**Description:** Convert banking instructions into verified on-chain token mints.

**Requirements:**
- FR-2.1: Extract recipient address from banking instruction
- FR-2.2: Convert USD amount to wei (18 decimals)
- FR-2.3: Validate recipient address format (EVM address)
- FR-2.4: Generate DON-signed report with minting data
- FR-2.5: Submit report to on-chain consumer contract
- FR-2.6: Verify transaction success
- FR-2.7: Log minting operation with bank reference

**Acceptance Criteria:**
- Recipient receives exact USD amount as stablecoin tokens
- Transaction is verified by Chainlink DON
- Minting is recorded on-chain with event emission
- Bank reference is preserved in transaction logs

---

#### FR-3: Security & Authorization
**Priority:** P0 (Must Have)

**Description:** Ensure only authorized workflows can trigger token minting.

**Requirements:**
- FR-3.1: Consumer contract validates DON signatures
- FR-3.2: Consumer contract validates workflow owner address
- FR-3.3: Consumer contract validates workflow name
- FR-3.4: Only consumer contract can call stablecoin.mint()
- FR-3.5: Workflow identity is cryptographically proven
- FR-3.6: Prevent replay attacks

**Acceptance Criteria:**
- Unauthorized minting attempts are rejected
- Only CRE workflow can trigger mints
- Signature verification occurs on-chain
- Workflow metadata is validated before execution

---

### 3.2 Non-Functional Requirements

#### NFR-1: Performance
- NFR-1.1: End-to-end minting latency < 30 seconds (Sepolia confirmation time)
- NFR-1.2: Support concurrent HTTP requests (DON handles multiple triggers)
- NFR-1.3: Gas-optimized smart contract operations

#### NFR-2: Reliability
- NFR-2.1: HTTP trigger must handle network failures gracefully
- NFR-2.2: Transaction failures must be logged with clear error messages
- NFR-2.3: System must be resilient to temporary RPC failures

#### NFR-3: Security
- NFR-3.1: Private keys stored securely (via 1Password CLI or equivalent)
- NFR-3.2: No hardcoded credentials in source code
- NFR-3.3: All on-chain operations verified by Chainlink DON
- NFR-3.4: Consumer contract follows IReceiverTemplate security pattern

#### NFR-4: Maintainability
- NFR-4.1: Code must be well-documented
- NFR-4.2: Configuration externalized (no hardcoded addresses)
- NFR-4.3: Clear separation of concerns (workflow / contracts)

---

## 4. User Stories

### 4.1 Primary Use Case: Bank Initiates Stablecoin Minting

**As a:** Bank Operations System  
**I want to:** Send a payment instruction to mint on-chain stablecoins  
**So that:** Customers can receive tokenized USD on the blockchain  

**Given:** A customer has deposited $1,000 USD into their bank account  
**When:** The bank's system sends a SWIFT-style minting instruction via HTTP  
**Then:** 
- The CRE workflow receives and validates the instruction
- A DON-signed report is generated
- The on-chain consumer contract verifies the report
- 1,000 stablecoin tokens (18 decimals) are minted to the customer's wallet
- The transaction is confirmed on Sepolia
- The bank reference is preserved in the event logs

**Acceptance Criteria:**
- ✅ Customer's wallet balance increases by 1,000 tokens
- ✅ Transaction visible on Etherscan with bank reference
- ✅ Minting completes within 30 seconds
- ✅ System returns transaction hash to bank

---

### 4.2 Error Handling Use Case

**As a:** Bank Operations System  
**I want to:** Receive clear error messages when minting fails  
**So that:** I can retry or escalate the issue  

**Scenarios:**
1. **Invalid recipient address:** Return error code `E001` with message
2. **Insufficient gas:** Return error code `E002` with details
3. **Contract paused:** Return error code `E003` with reason
4. **Network failure:** Return error code `E004` with retry guidance

---

## 5. Message Format Specifications

### 5.1 Inbound HTTP Message (SWIFT MT103-Inspired)

**Endpoint:** `POST /api/mint-instruction`

**Content-Type:** `application/json`

**Message Structure:**
```json
{
  "messageType": "MT103",
  "transactionReference": "BNKUS33XXXX202501151030001",
  "instructionType": "MINT",
  "valueDate": "2025-01-15",
  "currency": "USD",
  "amount": "1000.00",
  "orderingInstitution": {
    "bicCode": "BNKUS33XXXX",
    "name": "Example Bank N.A.",
    "country": "US"
  },
  "beneficiary": {
    "name": "Acme Corporation",
    "accountNumber": "ACME001234",
    "blockchainAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
  },
  "remittanceInformation": "Customer deposit - Wire transfer from external bank",
  "senderToReceiverInfo": "//INV/INVOICE-2025-001",
  "regulatoryReporting": "CTPY/US/123456789",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "messageHash": "sha256:a1b2c3d4..."
}
```

**Field Specifications:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageType` | String | Yes | Fixed value: "MT103" |
| `transactionReference` | String (35 chars) | Yes | Unique transaction identifier (bank's internal ref) |
| `instructionType` | String | Yes | Fixed value: "MINT" for this demo |
| `valueDate` | ISO Date | Yes | Effective date of transaction |
| `currency` | String (3 chars) | Yes | ISO 4217 currency code (USD) |
| `amount` | Decimal String | Yes | Amount in USD (e.g., "1000.00") |
| `orderingInstitution.bicCode` | String (11 chars) | Yes | SWIFT BIC code |
| `orderingInstitution.name` | String | Yes | Bank name |
| `orderingInstitution.country` | String (2 chars) | Yes | ISO 3166 country code |
| `beneficiary.name` | String | Yes | Customer/company name |
| `beneficiary.accountNumber` | String | Yes | Bank account reference |
| `beneficiary.blockchainAddress` | Address (42 chars) | Yes | EVM address starting with 0x |
| `remittanceInformation` | String | No | Payment details / memo |
| `senderToReceiverInfo` | String | No | Additional reference (invoice, etc.) |
| `regulatoryReporting` | String | No | Compliance / KYC reference |
| `timestamp` | ISO 8601 DateTime | Yes | Message creation time |
| `messageHash` | String | No | SHA-256 hash for integrity |

**Validation Rules:**
- `blockchainAddress` must be valid EVM address (checksum optional)
- `amount` must be positive, max 2 decimal places
- `transactionReference` must be unique (idempotency)
- `currency` must be "USD" for this demo
- `timestamp` must be within last 5 minutes (prevent replay)

---

### 5.2 Outbound Response Message

**Success Response (HTTP 200):**
```json
{
  "status": "SUCCESS",
  "transactionReference": "BNKUS33XXXX202501151030001",
  "chainlinkTransactionHash": "0x5905b48e72521f65fdc997a3911809432139fb9403a8d2091b700fd650e72a23",
  "recipientAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "mintedAmount": "1000000000000000000000",
  "mintedAmountFormatted": "1000.00",
  "network": "sepolia",
  "blockNumber": 12345678,
  "timestamp": "2025-01-15T10:30:45.123Z",
  "explorerUrl": "https://sepolia.etherscan.io/tx/0x5905b48..."
}
```

**Error Response (HTTP 400/500):**
```json
{
  "status": "ERROR",
  "errorCode": "E001",
  "errorMessage": "Invalid recipient address format",
  "transactionReference": "BNKUS33XXXX202501151030001",
  "timestamp": "2025-01-15T10:30:45.123Z",
  "details": {
    "field": "beneficiary.blockchainAddress",
    "receivedValue": "0xinvalid",
    "expectedFormat": "0x followed by 40 hexadecimal characters"
  }
}
```

**Error Codes:**
- `E001`: Invalid address format
- `E002`: Invalid amount (negative, zero, or too many decimals)
- `E003`: Missing required field
- `E004`: Transaction failed on-chain
- `E005`: Insufficient gas
- `E006`: Network/RPC error
- `E007`: Duplicate transaction reference (idempotency check)
- `E008`: Workflow authentication failed

---

## 6. Smart Contract Specifications

### 6.1 MintingConsumer.sol (NEW CONTRACT)

**Purpose:** Receive CRE reports and trigger stablecoin minting.

**Inheritance:**
- `IReceiverTemplate` (Chainlink CRE security pattern)

**Key Functions:**

#### 6.1.1 Constructor
```solidity
constructor(
    address _stablecoin,
    address expectedWorkflowOwner,
    bytes10 expectedWorkflowName
)
```
**Parameters:**
- `_stablecoin`: Address of StablecoinERC20 contract
- `expectedWorkflowOwner`: CRE workflow owner's wallet address
- `expectedWorkflowName`: Expected workflow name (e.g., "BankMint")

**Requirements:**
- Must store immutable stablecoin reference
- Must call parent IReceiverTemplate constructor

---

#### 6.1.2 _processReport (Internal)
```solidity
function _processReport(bytes calldata report) internal override
```
**Parameters:**
- `report`: ABI-encoded MintInstruction struct

**Requirements:**
- Decode report to extract: recipient, amount, memo
- Call `stablecoin.mint(recipient, amount)`
- Emit `MintInstructionReceived` event

**Events:**
```solidity
event MintInstructionReceived(
    address indexed recipient,
    uint256 amount,
    string bankReference,
    uint256 timestamp
);
```

---

### 6.2 StablecoinERC20.sol (REUSED WITH MINIMAL MODIFICATIONS)

**Source:** `stablecoin-workshop-evm/contracts/StablecoinERC20.sol`

**Required Modifications:** 
- ✅ **NONE** - Contract already has `mint()` with `onlyMinter` modifier
- ✅ **NONE** - Contract already has `grantMintRole()` for authorization

**Key Functions (Existing):**

#### 6.2.1 mint (External, Role-Protected)
```solidity
function mint(address account, uint256 amount) external onlyMinter
```
**Requirements:**
- Only addresses with minter role can call
- Must check `maxSupply` if configured
- Must emit standard ERC20 `Transfer` event

#### 6.2.2 grantMintRole (External, Owner Only)
```solidity
function grantMintRole(address minter) external onlyOwner
```
**Usage:**
- Called by owner to grant minter role to MintingConsumer

---

## 7. CRE Workflow Specifications

### 7.1 Workflow 1: Bank Minting Instruction Handler

**Workflow Name:** `bank-mint-workflow`

**Trigger:** HTTP Trigger

**Configuration Requirements:**

#### config.json
```json
{
  "workflowName": "BankMint",
  "chainName": "ethereum-testnet-sepolia",
  "mintingConsumerAddress": "0x...",
  "gasLimit": "500000",
  "httpEndpoint": "/api/mint-instruction"
}
```

#### secrets.yaml
```yaml
secretsNames:
  PRIVATE_KEY:
    - CRE_ETH_PRIVATE_KEY
  MINTING_CONSUMER_ADDRESS:
    - MINTING_CONSUMER_ADDRESS_ENV
```

---

### 7.2 Workflow Logic Flow

**Step 1: Receive HTTP Trigger**
- Input: HTTP POST with SWIFT-style JSON payload
- Validation: Check required fields
- Error: Return 400 if validation fails

**Step 2: Parse Banking Instruction**
- Extract: `beneficiary.blockchainAddress`
- Extract: `amount`
- Extract: `transactionReference`
- Validation: Address format, amount > 0

**Step 3: Convert Amount**
- Convert USD string to BigInt with 18 decimals
- Example: "1000.00" → `1000000000000000000000n`

**Step 4: Generate Report**
- Encode: `(address recipient, uint256 amount, string memo)`
- Generate DON-signed report via `runtime.report()`
- Signing: ECDSA signatures from DON nodes

**Step 5: Submit to Blockchain**
- Target: MintingConsumer contract
- Method: `evmClient.writeReport()`
- Gas: Configured gas limit
- Verification: Check `TxStatus.SUCCESS`

**Step 6: Return Response**
- Success: Return transaction hash and details
- Failure: Return error code and message

---

## 8. Deployment Requirements

### 8.1 Smart Contract Deployment (Sepolia)

**Deployment Order:**
1. Deploy `StablecoinERC20.sol`
2. Deploy `MintingConsumer.sol` (with workflow owner address)
3. Grant minter role: `stablecoin.grantMintRole(mintingConsumer.address)`

**Configuration Artifacts:**
- Contract addresses stored in `.env`
- Deployment receipts saved
- Etherscan verification completed

---

### 8.2 CRE Workflow Deployment

**Prerequisites:**
- Bun 1.2.21+ installed
- CRE CLI installed
- Funded Sepolia wallet (for gas)
- Workflow project initialized

**Setup Steps:**
1. Initialize CRE project: `cre init`
2. Install dependencies: `bun install`
3. Configure secrets (1Password CLI recommended)
4. Test locally: `cre workflow simulate`
5. Verify with webhook.site

---

## 9. Testing Requirements

### 9.1 Unit Testing

**Smart Contracts:**
- ✅ MintingConsumer validates workflow identity
- ✅ MintingConsumer decodes report correctly
- ✅ MintingConsumer calls stablecoin.mint()
- ✅ StablecoinERC20 enforces minter role
- ✅ Unauthorized mint attempts fail

**CRE Workflow:**
- ✅ Parse valid SWIFT-style message
- ✅ Reject invalid address format
- ✅ Reject invalid amount format
- ✅ Reject missing required fields
- ✅ Convert USD to wei correctly
- ✅ Generate valid ABI-encoded report

---

### 9.2 Integration Testing

**End-to-End Scenarios:**

**Test Case 1: Successful Minting**
- Send valid HTTP POST
- Verify workflow receives request
- Verify report generation
- Verify on-chain transaction
- Verify token balance increase
- Verify event emission

**Test Case 2: Invalid Address**
- Send message with invalid address
- Verify error response (E001)
- Verify no on-chain transaction

**Test Case 3: Duplicate Transaction**
- Send same `transactionReference` twice
- Verify second request rejected (E007)

**Test Case 4: Network Failure**
- Simulate RPC failure
- Verify retry logic (if implemented)
- Verify error response (E006)

---

### 9.3 Security Testing

**Attack Scenarios:**

**Test 1: Unauthorized Workflow**
- Deploy rogue workflow with different owner
- Attempt to mint via rogue workflow
- Verify consumer contract rejects

**Test 2: Replay Attack**
- Capture valid report
- Replay same report
- Verify rejection (signature/nonce check)

**Test 3: Signature Tampering**
- Modify report payload
- Attempt submission
- Verify Forwarder rejects invalid signature

---

## 10. Success Criteria

### 10.1 Functional Success

✅ **Criterion 1:** Bank sends HTTP POST → Tokens minted successfully
- Metric: End-to-end completion within 30 seconds
- Verification: Etherscan shows transaction with correct amount

✅ **Criterion 2:** Only authorized workflow can mint
- Metric: 100% rejection of unauthorized attempts
- Verification: Contract events show only valid mints

✅ **Criterion 3:** Error handling works correctly
- Metric: All error codes return appropriate messages
- Verification: Test suite passes for all error scenarios

---

### 10.2 Educational Success

✅ **Criterion 1:** Participants understand TradFi → Web3 bridge
- Metric: Can explain workflow end-to-end
- Verification: Post-workshop survey

✅ **Criterion 2:** Participants understand CRE security model
- Metric: Can identify security benefits vs direct wallet pattern
- Verification: Quiz/discussion

✅ **Criterion 3:** Participants can replicate demo
- Metric: Follow documentation without assistance
- Verification: Successfully complete workshop independently

---

## 11. Out of Scope (Future Phases)

### 11.1 Not Included in MVP

❌ **Cross-Chain CCIP Integration**
- Reason: Focus on core minting functionality first
- Future: Phase 2 addition

❌ **DEX Swap Integration**
- Reason: Secondary feature, not core to demo
- Future: Phase 3 optional extension

❌ **ETH Collateral Backing**
- Reason: Simplified for demo (pure minting)
- Future: Could add `depositAndMint()` flow

❌ **Multiple Blockchain Support**
- Reason: Sepolia only for MVP
- Future: Add Avalanche Fuji in Phase 2

❌ **Production Security Features**
- Reason: Educational demo, not production-ready
- Examples: Rate limiting, pause functionality, admin controls

---

## 12. Constraints & Assumptions

### 12.1 Technical Constraints

- **Network:** Sepolia testnet only
- **Language:** TypeScript for CRE workflow
- **Token Standard:** ERC20 (18 decimals)
- **Message Format:** JSON (not actual SWIFT binary)
- **Security:** Testnet-grade (not production-audited)

### 12.2 Assumptions

- Participants have Sepolia ETH for testing
- Participants have basic Solidity knowledge
- Participants have basic TypeScript knowledge
- HTTP trigger testing uses webhook.site
- Workshop conducted with live instructor support

---

**End of Product Requirements Document**
