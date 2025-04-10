# Day 2 – Development Environment Setup (Solana)

**Date:** 9th April, 2025

## 🛠️ What I Did

- Installed the Rust toolchain
- Installed Solana CLI tools
- Configured development environment
- Connected to Solana clusters (localnet, devnet, testnet)
- Generated a local wallet and requested test SOL via airdrop
- Explored common Solana CLI commands via a cheat sheet

## 📦 Commands Used

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Install Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"

# Configure devnet
solana config set --url https://api.devnet.solana.com

# Generate wallet
solana-keygen new

# View address and balance
solana address
solana balance

# Request test SOL
solana airdrop 2
```

## 🧾 Resources

- 🔗 [Solana CLI Cheat Sheet by @dvrvsimi](https://gist.github.com/dvrvsimi/500f02ee94957b14aa9755c67f669436)

## 🗣️ Build In Public

Shared the setup process in a thread  
(https://x.com/Chrisasek/status/1910315907005432222)