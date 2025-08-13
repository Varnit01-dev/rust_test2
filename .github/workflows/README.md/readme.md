# 🔄 Token Swap Program (AMM)

[![Solana](https://img.shields.io/badge/Solana-Native-9945FF?style=flat&logo=solana)](https://solana.com)
[![Rust](https://img.shields.io/badge/Language-Rust-orange?style=flat&logo=rust)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Devnet](https://img.shields.io/badge/Network-Devnet-green)](https://explorer.solana.com/?cluster=devnet)

A production-ready **Automated Market Maker (AMM)** implementation for Solana. Powers decentralized token swaps using the same architecture as Raydium, Orca, and Jupiter.

## ✨ Features

🔄 **Token Swapping** - Trade any SPL token pairs with constant product formula  
💧 **Liquidity Pools** - Provide liquidity and earn trading fees  
🛡️ **Slippage Protection** - Guaranteed minimum outputs  
🔐 **Secure PDAs** - Program-derived authorities for pool management  
⚡ **Gas Optimized** - Efficient Rust implementation  
📊 **Fee Collection** - Configurable trading fees for LPs  

## 🚀 Quick Start

### Prerequisites
- Rust 1.70+
- Solana CLI 1.18+
- 2+ SOL on devnet

### 1. Setup Environment
```bash

sh -c "$(curl -sSfL https://release.solana.com/stable/install)"


solana config set --url devnet
solana-keygen new --outfile ~/.config/solana/id.json
solana airdrop 2
```

### 2. Clone & Build
```bash
git clone <your-repo-url>
cd token-swap-program
cargo build-bpf
```

### 3. Deploy to Devnet
```bash
solana program deploy target/deploy/token_swap_program.so
```

> 💡 **Save the Program ID!** You'll need it for all interactions.

## 🔧 Program Instructions

### `Initialize` - Create New Pool
```rust
Initialize {
    nonce: u8,          
    fee_numerator: 25,   
    fee_denominator: 10000,
}
```

### `Swap` - Trade Tokens
```rust
Swap {
    amount_in: 1000000,      
    minimum_amount_out: 950000,
}
```

### `AddLiquidity` - Provide Liquidity  
```rust
AddLiquidity {
    pool_token_amount: 1000000,    
    maximum_token_a_amount: 500000, 
    maximum_token_b_amount: 500000, 
}
```

### `RemoveLiquidity` - Withdraw Assets
```rust
RemoveLiquidity {
    pool_token_amount: 500000,      
    minimum_token_a_amount: 240000, 
    minimum_token_b_amount: 240000, 
}
```

## 🧮 AMM Mathematics

### Constant Product Formula
```
x * y = k
```
- `x` = Token A reserves
- `y` = Token B reserves
- `k` = Constant invariant

### Swap Calculation
```
output = (reserves_out * input) / (reserves_in + input)
```

### Fee Application
```rust
fee = (amount_in * fee_numerator) / fee_denominator
swap_amount = amount_in - fee
```

## 📂 Account Structure

### SwapPool Account (324 bytes)
```rust
pub struct SwapPool {
    pub version: u8,                    
    pub nonce: u8,                     
    pub token_a: Pubkey,
    pub token_b: Pubkey,                
    pub pool_mint: Pubkey,
    pub token_a_account: Pubkey,       
    pub token_b_account: Pubkey,     
    pub pool_fee_account: Pubkey,      
    pub fee_numerator: u64,            
    pub fee_denominator: u64,          
    
}
```

## 🔒 Security Considerations

✅ **Program Derived Addresses** - Secure pool authority  
✅ **Arithmetic Overflow Protection** - Safe math operations  
✅ **Account Ownership Validation** - Prevents unauthorized access  
✅ **Slippage Protection** - User-defined minimum outputs  
✅ **Signer Verification** - Ensures transaction authorization  

## 🌍 Production Examples

| DEX | TVL | Daily Volume | Architecture |
|-----|-----|--------------|-------------|
| **Raydium** | $2B+ | $100M+ | This AMM model |
| **Orca** | $500M+ | $50M+ | Enhanced version |
| **Jupiter** | Aggregator | $200M+ | Routes through AMMs |

## 🧪 Testing Guide

```bash

cargo test


cargo test test_swap


RUST_LOG=debug cargo test -- --nocapture


cargo test --test integration
```

## 🚨 Deployment Checklist

- [ ] **Code audited** for production use
- [ ] **Tests passing** on all functions
- [ ] **Fee rates configured** appropriately (0.25-0.3%)
- [ ] **Slippage tolerances** tested
- [ ] **PDA derivation** verified
- [ ] **Token accounts** properly initialized
- [ ] **Upgrade authority** considered for production

## 💡 Integration Examples

### JavaScript/TypeScript Client
```javascript
import { Connection, PublicKey } from '@solana/web3.js';




```

### Common Use Cases
- **DEX Frontend** - Trading interface
- **Arbitrage Bots** - Automated trading
- **Liquidity Mining** - Yield farming protocols  
- **Portfolio Management** - Automated rebalancing

## 📈 Roadmap

- [ ] **Phase 1**: Core AMM functionality ✅
- [ ] **Phase 2**: Advanced fee structures
- [ ] **Phase 3**: Concentrated liquidity
- [ ] **Phase 4**: Multi-hop routing
- [ ] **Phase 5**: Governance integration

## 🤝 Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- **Solana Labs** - For the incredible blockchain infrastructure
- **SPL Token Program** - Token standard implementation
- **Uniswap** - AMM formula inspiration
- **Serum DEX** - Solana DeFi pioneer

---

<div align="center">

**🚀 Built for the Solana ecosystem | ⚡ Powered by Rust | 🌊 Ready for DeFi**



</div>