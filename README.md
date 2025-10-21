# CRE Stablecoin Demo
## Chainlink Runtime Environment - Traditional Finance to Web3 Bridge

> **⚠️ Educational Demonstration Only**  
> This project is designed for learning purposes and is not production-ready. Do not use in production without comprehensive security audits.

---

## 🎯 Project Overview

This demonstration showcases how **Chainlink Runtime Environment (CRE)** can serve as a secure bridge between traditional financial systems and blockchain infrastructure.

**Key Concept:** A bank's legacy system sends SWIFT-style payment instructions via HTTP → CRE workflow validates and processes → Stablecoin tokens are minted on-chain with DON verification.

---

## 📚 Documentation

- **[PRD.md](./PRD.md)** - Complete Product Requirements Document
- **INSTRUCTIONS.md** - Step-by-step setup guide (coming soon)
- **ARCHITECTURE.md** - System design details (coming soon)
- **API_REFERENCE.md** - HTTP API specification (coming soon)

---

## 🏗️ Architecture

```
Traditional Banking → HTTP POST → CRE Workflow → Signed Report → Consumer Contract → Mint Stablecoin
```

**Components:**
- **CRE Workflow** (TypeScript) - Receives HTTP triggers, generates DON-signed reports
- **MintingConsumer.sol** - Validates reports, triggers minting
- **StablecoinERC20.sol** - Role-based ERC20 token with minting controls
- **DataStreamsOracle.sol** - Reference implementation (Chainlink Data Streams)

---

## 🚀 Quick Start

> **Status:** Requirements phase - implementation coming soon

### Prerequisites
- Bun 1.2.21+
- CRE CLI installed
- Sepolia testnet ETH
- Basic Solidity & TypeScript knowledge

### Setup
```bash
# Clone repository
git clone https://github.com/YOUR_USERNAME/cre-stablecoin-demo
cd cre-stablecoin-demo

# Follow INSTRUCTIONS.md for detailed setup
```

---

## 🎓 Learning Objectives

By completing this demo, you will learn:

- ✅ How to integrate traditional finance systems with blockchain
- ✅ CRE workflow development (HTTP triggers, report generation)
- ✅ Secure consumer contract patterns (IReceiverTemplate)
- ✅ Role-based token minting with DON verification
- ✅ SWIFT-style message handling in Web3

---

## 📋 Project Status

- [x] Requirements definition (PRD.md)
- [ ] Smart contract development
- [ ] CRE workflow implementation
- [ ] Testing & documentation
- [ ] Workshop materials

---

## 🤝 Contributing

This is an educational project. Feedback and improvements are welcome!

---

## 📄 License

MIT License - See [LICENSE](./LICENSE) for details

---

## 🔗 Resources

- [Chainlink CRE Documentation](https://docs.chain.link/cre)
- [Chainlink Data Streams](https://docs.chain.link/data-streams)
- [Chainlink CCIP](https://docs.chain.link/ccip)

---

**Built with ❤️ using Chainlink CRE**

