# Day 5 - Introduction to Anchor (Solana Framework)

Today we were introduced to **Anchor**, the framework that simplifies Solana smart contract development.

## ✅ What We Covered

### 📦 Installation
We set up Anchor using Cargo (Rust’s package manager) and Solana CLI.

```bash
cargo install --git https://github.com/coral-xyz/anchor --tag v0.29.0 anchor-cli --locked
```

After installation, we verified it using:
```bash
solana --version
anchor --version
```

### 🗂️ Project Structure

Anchor organizes projects with the following structure:

- `programs/` — Rust-based on-chain programs
- `tests/` — Off-chain logic, written in JavaScript/TypeScript
- `migrations/` — Deployment scripts
- `Anchor.toml` — Anchor configuration file
- `Cargo.toml` — Rust dependencies

### 🔁 Anchor Application Flow

1. Write the smart contract using Rust and Anchor macros:
   - `#[program]`
   - `#[derive(Accounts)]`
   - `#[account]`

2. Build the program:
```bash
anchor build
```

3. Deploy it:
```bash
anchor deploy
```

4. Interact with it using scripts in the `tests/` folder

### 💡 Why Anchor?

- Eliminates boilerplate code
- Provides macros for account validation
- Improves safety and productivity

We’ll be writing our first Anchor program next!

---