# Session 8: Writing Your First Solana Program with Anchor

## Basic Anchor Program Structure and Deployment

In this session, we'll create a simple counter program using Anchor. This program will let us:
1. Initialize a counter account
2. Increment the counter
3. Decrement the counter

### What is Anchor?

Anchor is a framework that simplifies Solana program development. It's like using Rails for Ruby or Django for Python - it handles a lot of the boilerplate code for us.

**Real-world analogy**: If building a raw Solana program is like building a house from scratch with raw materials, Anchor is like building with prefabricated parts. Same end result, but much faster and with fewer chances for structural mistakes.

### Project Setup

First, let's set up our Anchor project:

```bash
# Install Anchor CLI if you haven't already
cargo install --git https://github.com/coral-xyz/anchor avm --locked
avm install latest
avm use latest

# Create a new project
anchor init counter-program
cd counter-program
```

This creates a standard project structure:

```
counter-program/
├── Anchor.toml          # Configuration file
├── Cargo.toml           # Rust dependencies
├── programs/            # Contains your on-chain program
│   └── counter-program/ # Your Solana program code
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs   # Your program logic
└── tests/               # Test files
    └── counter-program.ts
```

### Program Implementation (lib.rs)

Here's our counter program using Anchor:

```rust
// programs/counter-program/src/lib.rs
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS"); // Program ID (will be different for you)

#[program]
pub mod counter_program {
    use super::*;

    // Initialize a new counter
    pub fn initialize(ctx: Context<Initialize>, initial_value: u64) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.authority = ctx.accounts.user.key();
        counter.count = initial_value;

        msg!("Counter initialized with value: {}", initial_value);
        Ok(())
    }

    // Increment counter by 1
    pub fn increment(ctx: Context<Update>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        
        // Verify authority
        require!(
            counter.authority == ctx.accounts.authority.key(),
            CounterError::UnauthorizedAccess
        );
        
        counter.count += 1;
        msg!("Counter incremented to: {}", counter.count);
        Ok(())
    }

    // Decrement counter by 1
    pub fn decrement(ctx: Context<Update>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        
        // Verify authority
        require!(
            counter.authority == ctx.accounts.authority.key(),
            CounterError::UnauthorizedAccess
        );
        
        // Prevent underflow
        require!(counter.count > 0, CounterError::UnderflowError);
        
        counter.count -= 1;
        msg!("Counter decremented to: {}", counter.count);
        Ok(())
    }
}

// Account validation for initialize instruction
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,                 // Create a new account
        payer = user,         // User pays for account creation
        space = 8 + 32 + 8,   // Discriminator (8) + authority pubkey (32) + count (8)
        seeds = [b"counter", user.key().as_ref()],
        bump
    )]
    pub counter: Account<'info, Counter>,
    
    #[account(mut)]
    pub user: Signer<'info>,
    
    pub system_program: Program<'info, System>,
}

// Account validation for increment/decrement instructions
#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
    
    pub authority: Signer<'info>,
}

// Counter account data structure
#[account]
pub struct Counter {
    pub authority: Pubkey,  // Authority allowed to update the counter
    pub count: u64,         // Current count value
}

// Custom error codes
#[error_code]
pub enum CounterError {
    #[msg("You are not authorized to perform this action")]
    UnauthorizedAccess,
    #[msg("Counter cannot be decremented below zero")]
    UnderflowError,
}
```

**Analogy**: If the program was a vending machine:
- `initialize` sets up the machine with initial inventory
- `increment` adds an item to inventory
- `decrement` removes an item from inventory
- The `Counter` struct is the inventory tracking system
- `CounterError` is the error display on the machine

### Understanding the Code

#### 1. Macro Magic

Anchor uses macros to simplify your code:

```rust
#[program]               // Marks the module containing instruction handlers
#[derive(Accounts)]      // Validates accounts for each instruction
#[account]               // Defines an account data structure
#[error_code]            // Creates custom error types
```

**Analogy**: Macros are like cookie cutters - they shape your code into standard patterns automatically.

#### 2. Account Constraints

Anchor makes account validation easy with constraints:

```rust
#[account(
    init,                 // Create a new account
    payer = user,         // User pays for account creation
    space = 8 + 32 + 8,   // Discriminator (8) + authority pubkey (32) + count (8)
    seeds = [b"counter", user.key().as_ref()],
    bump
)]
```

**Analogy**: Account constraints are like a security system that checks IDs and permissions before letting someone in.

#### 3. Program Derived Addresses (PDAs)

In Anchor, PDAs are handled with `seeds` and `bump` constraints:

```rust
seeds = [b"counter", user.key().as_ref()],
bump
```

This automatically creates an account at a PDA derived from:
- The string "counter"
- The user's public key
- The program ID

**Analogy**: A PDA is like a deterministic locker number. If you know the locker room (program) and the person's name (seed), you can always find the right locker (account).

## Deploying Your Anchor Program

Deployment is simple with Anchor:

```bash
# Build the program
anchor build

# Deploy to devnet (make sure your Solana CLI is configured for devnet)
anchor deploy
```

This will update your `Anchor.toml` and `declare_id!()` with your program's new ID.

**Analogy**: Deployment with Anchor is like having a moving company that packs and ships your furniture to a new house - much easier than doing it yourself.

## Testing Your Anchor Program

Anchor makes testing straightforward with TypeScript:

```typescript
// tests/counter-program.ts
import * as anchor from '@coral-xyz/anchor';
import { Program } from '@coral-xyz/anchor';
import { CounterProgram } from '../target/types/counter_program';
import { expect } from 'chai';

describe('counter-program', () => {
  // Configure the client
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
  const program = anchor.workspace.CounterProgram as Program<CounterProgram>;
  const user = provider.wallet;

  it('Initializes counter with initial value', async () => {
    // Get PDA for the counter
    const [counterPDA] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from('counter'), user.publicKey.toBuffer()],
      program.programId
    );

    // Initialize counter with value 5
    await program.methods
      .initialize(new anchor.BN(5))
      .accounts({
        counter: counterPDA,
        user: user.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    // Fetch the counter account
    const counterAccount = await program.account.counter.fetch(counterPDA);
    expect(counterAccount.count.toNumber()).to.equal(5);
    expect(counterAccount.authority.toString()).to.equal(user.publicKey.toString());
  });

  it('Increments counter', async () => {
    // Get PDA
    const [counterPDA] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from('counter'), user.publicKey.toBuffer()],
      program.programId
    );

    // First fetch the current value
    const counterBefore = await program.account.counter.fetch(counterPDA);
    const valueBefore = counterBefore.count.toNumber();

    // Increment counter
    await program.methods
      .increment()
      .accounts({
        counter: counterPDA,
        authority: user.publicKey,
      })
      .rpc();

    // Verify it was incremented
    const counterAfter = await program.account.counter.fetch(counterPDA);
    expect(counterAfter.count.toNumber()).to.equal(valueBefore + 1);
  });

  it('Decrements counter', async () => {
    // Get PDA
    const [counterPDA] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from('counter'), user.publicKey.toBuffer()],
      program.programId
    );

    // First fetch the current value
    const counterBefore = await program.account.counter.fetch(counterPDA);
    const valueBefore = counterBefore.count.toNumber();

    // Decrement counter
    await program.methods
      .decrement()
      .accounts({
        counter: counterPDA,
        authority: user.publicKey,
      })
      .rpc();

    // Verify it was decremented
    const counterAfter = await program.account.counter.fetch(counterPDA);
    expect(counterAfter.count.toNumber()).to.equal(valueBefore - 1);
  });

  it('Prevents unauthorized access', async () => {
    // Get PDA
    const [counterPDA] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from('counter'), user.publicKey.toBuffer()],
      program.programId
    );
    
    // Create another user (keypair)
    const unauthorizedUser = anchor.web3.Keypair.generate();
    
    // Airdrop some SOL to this user
    const airdropSignature = await provider.connection.requestAirdrop(
      unauthorizedUser.publicKey,
      1 * anchor.web3.LAMPORTS_PER_SOL
    );
    await provider.connection.confirmTransaction(airdropSignature);
    
    try {
      // Try to increment with unauthorized user
      await program.methods
        .increment()
        .accounts({
          counter: counterPDA,
          authority: unauthorizedUser.publicKey,
        })
        .signers([unauthorizedUser]) // Use the unauthorized user as signer
        .rpc();
      
      // Should not reach here
      expect.fail('Expected error but transaction succeeded');
    } catch (error) {
      // Expected error
      expect(error.toString()).to.include('UnauthorizedAccess');
    }
  });
});
```

To run the tests:

```bash
anchor test
```

**Analogy**: Testing is like having a home inspector who checks that everything in your program works as expected before the new owners move in.

## Using Your Program in a Frontend

Here's how to interact with your program from a simple frontend:

```typescript
// Connect to the Solana network and your deployed program
import * as anchor from '@coral-xyz/anchor';
import { Program } from '@coral-xyz/anchor';
import { CounterProgram } from './types/counter_program'; // Generated by Anchor
import { Connection, PublicKey } from '@solana/web3.js';

// Set up connection and wallet
const connection = new Connection('https://api.devnet.solana.com');
const wallet = window.solana; // Phantom wallet or other injected wallet

// Connect to your program
const provider = new anchor.AnchorProvider(connection, wallet, {});
const programId = new PublicKey('YOUR_PROGRAM_ID');
const program = new Program<CounterProgram>(IDL, programId, provider);

// Get PDA for counter
const [counterPDA] = PublicKey.findProgramAddressSync(
  [Buffer.from('counter'), wallet.publicKey.toBuffer()],
  programId
);

// Initialize a counter
async function initializeCounter(initialValue) {
  await program.methods
    .initialize(new anchor.BN(initialValue))
    .accounts({
      counter: counterPDA,
      user: wallet.publicKey,
      systemProgram: anchor.web3.SystemProgram.programId,
    })
    .rpc();
}

// Increment counter
async function incrementCounter() {
  await program.methods
    .increment()
    .accounts({
      counter: counterPDA,
      authority: wallet.publicKey,
    })
    .rpc();
}

// Get current count
async function getCount() {
  const counterAccount = await program.account.counter.fetch(counterPDA);
  return counterAccount.count.toNumber();
}
```

**Analogy**: The frontend is like the user interface panel of a smart home. It lets users control the smart devices (the program) without knowing how the wiring works.

## Advanced PDA Usage in Anchor

Anchor makes PDAs much easier to work with:

### 1. Creating PDAs with seeds

```rust
#[account(
    init,
    payer = user,
    space = 8 + 32 + 8,
    seeds = [b"counter", user.key().as_ref()],
    bump
)]
pub counter: Account<'info, Counter>,
```

### 2. Reading from a PDA

```rust
#[account(
    seeds = [b"counter", authority.key().as_ref()],
    bump,
    has_one = authority
)]
pub counter: Account<'info, Counter>,
```

### 3. Using PDAs for Cross-Program Invocation (CPI)

```rust
// Define the seeds for the PDA
let seeds = &[
    b"counter", 
    ctx.accounts.user.key.as_ref(),
    &[bump]
];

// CPI with signer PDA
token_program::transfer(
    CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        token_program::Transfer {
            from: ctx.accounts.vault.to_account_info(),
            to: ctx.accounts.user_token_account.to_account_info(),
            authority: ctx.accounts.vault_authority.to_account_info(),
        },
        &[seeds]
    ),
    amount
)?;
```

**Analogy**: In traditional Solana, signing with PDAs is like having to manually stamp documents with the company seal. With Anchor, it's like having an automatic stamping machine that handles all the details for you.

## Summary

In this session, we've:

1. Set up an Anchor project structure
2. Implemented a counter program with:
   - Initialize functionality
   - Increment/decrement functionality
   - PDA-based storage
   - Error handling
3. Learned how to deploy and test the program
4. Explored how to integrate with a frontend

Anchor makes Solana development much simpler by:
- Reducing boilerplate code
- Automating account validation
- Simplifying PDAs
- Providing a robust testing framework

**Key differences from raw Solana development:**
- Declares instructions with normal functions
- Handles serialization/deserialization automatically
- Simplifies account initialization and verification
- Generates TypeScript bindings for frontend integration

Next steps:
1. Add more features to your counter (e.g., reset, set to specific value)
2. Create a web frontend to interact with your program
3. Learn about more advanced Anchor features like events
