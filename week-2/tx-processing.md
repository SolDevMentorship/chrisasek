# Session 9: Transaction Processing on Solana

## Transaction Lifecycle on Solana

Understanding how transactions work on Solana is crucial for developing efficient and secure programs. Let's break down the full lifecycle of a transaction.

### 1. Transaction Creation

A transaction on Solana consists of three key components:

```typescript
// Client-side transaction construction using @solana/web3.js v1.87.6
import { Connection, PublicKey, SystemProgram, Transaction } from '@solana/web3.js';

// Create a new transaction
const transaction = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey: wallet.publicKey,
    toPubkey: recipient,
    lamports: 1_000_000, // 0.001 SOL
  })
);

// Set the fee payer and recent blockhash
transaction.feePayer = wallet.publicKey;
transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
```

The three components are:
- **Instructions**: What to do (like "transfer 0.001 SOL")
- **Account metadata**: Which accounts to read/write
- **Signatures**: Proof of authorization

**Analogy**: Think of a transaction like a check. It has instructions ("pay $100"), account details (your account, recipient account), and a signature that authorizes it.

### 2. Client Processing

Before submission, your client code:
1. Gets a recent blockhash (transaction expiration mechanism)
2. Sets the fee payer
3. Adds required signatures
4. Serializes the transaction

```typescript
// Add signature
const signedTx = await wallet.signTransaction(transaction);

// Serialize for transmission
const serializedTx = signedTx.serialize();
```

**Analogy**: This is like filling out the check, signing it, and putting it in an envelope.

### 3. Network Submission

The client sends the transaction to a Solana node (validator):

```typescript
// Submit to network
const signature = await connection.sendRawTransaction(serializedTx);

// Or use the combined method
const signature = await connection.sendTransaction(transaction, [wallet]);

console.log(`Transaction submitted: https://explorer.solana.com/tx/${signature}`);
```

**Analogy**: This is like dropping your check in the mail or handing it to a bank teller.

### 4. Transaction Propagation

Once received, the validator:
1. Verifies the transaction format
2. Checks that the blockhash isn't too old (currently valid for ~2 minutes)
3. Validates signatures
4. Shares the transaction with other validators

**Analogy**: The bank teller checks your ID, verifies the date on the check, and routes it for processing.

### 5. Leader Selection and Execution

In Solana's consensus mechanism:
- One validator is the current "leader" (rotates every ~400ms)
- The leader batches transactions into a block
- The leader executes transactions and records state changes

During execution, each instruction in your transaction is processed:
1. The program is loaded into a SBF (Solana Berkeley Filter) VM
2. Account permissions are checked
3. Your program's code runs
4. State changes are computed

**Analogy**: This is like the bank's processing center that actually moves money between accounts and records the changes.

### 6. Confirmation and Finality

After execution:
1. State changes are written to accounts
2. The block is propagated to other validators
3. As more validators verify the block, it gains more confirmations
4. After enough confirmations (~31 blocks, ~0.4 seconds), the transaction reaches finality

```typescript
// Wait for confirmation with the latest commitment level
const confirmation = await connection.confirmTransaction({
  signature,
  blockhash: transaction.recentBlockhash,
  lastValidBlockHeight: (await connection.getLatestBlockhash()).lastValidBlockHeight
}, 'confirmed');

// Check if the transaction was successful
if (confirmation.value.err) {
  console.error('Transaction failed:', confirmation.value.err);
} else {
  console.log('Transaction confirmed!');
}
```

**Analogy**: This is like waiting for the check to clear, with other banks confirming the transaction is valid.

### Transaction Lifecycle Diagram

```
+----------------+    +----------------+    +----------------+
| Client Creates |    | Validator      |    | Leader         |
| Transaction    | => | Receives and   | => | Processes      |
|                |    | Verifies       |    | Transaction    |
+----------------+    +----------------+    +----------------+
                                                    ||
+----------------+    +----------------+    +----------------+
| Transaction    |    | Block is       |    | Other          |
| Reaches        | <= | Confirmed by   | <= | Validators     |
| Finality       |    | Network        |    | Verify Block   |
+----------------+    +----------------+    +----------------+
```

## Signatures and Multisig Transactions

### Basic Signature Mechanism

Every transaction requires signatures from accounts marked as signers:

```typescript
// Create a simple transaction
const tx = new Transaction().add(
    SystemProgram.transfer({
        fromPubkey: sender.publicKey,
        toPubkey: receiver.publicKey,
        lamports: 1_000_000, // 0.001 SOL
    })
);

// Set recent blockhash and fee payer
tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
tx.feePayer = sender.publicKey;

// Sign with the sender's keypair
tx.sign(sender);

// Or use the wallet adapter in a web app
const signedTx = await wallet.signTransaction(tx);
```

**Analogy**: A signature is like your handwritten signature on a check - it proves you authorize the transaction.

### How Signatures Work

1. Transactions contain a message (instructions + accounts)
2. Signers create signatures by signing this message with their private key
3. Anyone can verify the signature with the signer's public key

```rust
// Solana program checking a signature (Rust)
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let authority = next_account_info(accounts_iter)?;
    
    // Check if the authority account signed the transaction
    if !authority.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }
    
    // Continue with authorized operation...
    Ok(())
}
```

**Analogy**: It's like a wax seal on a letter - only you can make it (with your private key), but anyone can verify it's yours (with your public key).

### Multisig Transactions

Solana has built-in support for multisig accounts through the SPL Token program and other specialized programs. Here's how to use the Token Program's multisig:

```typescript
import { 
  Connection, 
  PublicKey, 
  Transaction, 
  sendAndConfirmTransaction 
} from '@solana/web3.js';
import { 
  Token, 
  TOKEN_PROGRAM_ID, 
  createMultisig 
} from '@solana/spl-token';

// Create a multisig with 2-of-3 signers required
const multisigAccount = await createMultisig(
  connection,
  payer, // pays for the transaction
  [signer1.publicKey, signer2.publicKey, signer3.publicKey], // signers
  2 // threshold: require 2 out of 3 signatures
);

console.log(`Multisig account created: ${multisigAccount.toBase58()}`);

// Later, to use the multisig for a token transfer:
const transaction = new Transaction();
transaction.add(
  Token.createTransferInstruction(
    TOKEN_PROGRAM_ID,
    sourceTokenAccount,
    destinationTokenAccount,
    multisigAccount, // owner
    [signer1.publicKey, signer2.publicKey], // signers (must meet threshold)
    amount
  )
);

// Sign with the required signers
transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
transaction.feePayer = payer.publicKey;
transaction.sign(payer, signer1, signer2);

// Send the transaction
const signature = await sendAndConfirmTransaction(
  connection,
  transaction,
  [payer, signer1, signer2]
);
```

**Analogy**: Think of a safe deposit box that requires two keys to open - one from you and one from the bank.

### Modern Multisig Implementation with Anchor

Here's a simplified multisig implementation using Anchor:

```rust
// Anchor program for a simple multisig (lib.rs)
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod simple_multisig {
    use super::*;

    // Initialize a new multisig account
    pub fn create_multisig(
        ctx: Context<CreateMultisig>,
        owners: Vec<Pubkey>,
        threshold: u8,
    ) -> Result<()> {
        let multisig = &mut ctx.accounts.multisig;
        
        // Validate threshold
        if threshold == 0 || threshold > owners.len() as u8 {
            return Err(error!(ErrorCode::InvalidThreshold));
        }
        
        multisig.owners = owners;
        multisig.threshold = threshold;
        multisig.owner_set_seqno = 0;
        
        Ok(())
    }

    // Execute a transaction if enough owners have signed
    pub fn execute_transaction(ctx: Context<ExecuteTransaction>) -> Result<()> {
        // Count valid signatures
        let mut sig_count = 0;
        for owner in ctx.accounts.multisig.owners.iter() {
            for signer in ctx.remaining_accounts.iter() {
                if signer.is_signer && signer.key() == *owner {
                    sig_count += 1;
                }
            }
        }
        
        // Check threshold
        if sig_count < ctx.accounts.multisig.threshold {
            return Err(error!(ErrorCode::NotEnoughSigners));
        }
        
        // Execute the transaction (CPI call would go here)
        // ...
        
        Ok(())
    }
}

#[derive(Accounts)]
pub struct CreateMultisig<'info> {
    #[account(init, payer = payer, space = 8 + 32 + 1 + 4 + 32 * 10)]
    pub multisig: Account<'info, Multisig>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ExecuteTransaction<'info> {
    pub multisig: Account<'info, Multisig>,
    // The remaining_accounts will contain the signers
}

#[account]
pub struct Multisig {
    pub owners: Vec<Pubkey>,
    pub threshold: u8,
    pub owner_set_seqno: u8,
}

#[error_code]
pub enum ErrorCode {
    #[msg("Threshold must be greater than 0 and less than or equal to the number of owners")]
    InvalidThreshold,
    #[msg("Not enough signers to meet the threshold")]
    NotEnoughSigners,
}
```

**Analogy**: The program acts like a security guard, counting how many authorized people have shown their ID before allowing access.

## Program Testing with Solana CLI

### Setting Up a Test Environment

First, ensure you're on a test network (localnet or devnet):

```bash
# Configure for devnet
solana config set --url devnet

# Check your configuration
solana config get

# Create a new wallet for testing (if needed)
solana-keygen new --outfile ~/.config/solana/devnet.json
solana config set --keypair ~/.config/solana/devnet.json

# Fund your wallet
solana airdrop 2
```

**Analogy**: This is like setting up a test kitchen where you can experiment without affecting real customers.

### Deploying Your Program for Testing

```bash
# Build your program (for Rust/Anchor programs)
cargo build-sbf

# Or for Anchor programs
anchor build

# Deploy to devnet
solana program deploy ./target/deploy/your_program.so

# For Anchor programs
anchor deploy
```

**Analogy**: This is like installing your equipment in the test kitchen.

### Creating Test Accounts

```bash
# Create a keypair for testing
solana-keygen new -o test-wallet.json

# Airdrop some SOL to the test wallet
solana airdrop 2 $(solana-keygen pubkey test-wallet.json)

# Create a token account for testing SPL tokens
spl-token create-account TokenAddressHere --owner $(solana-keygen pubkey test-wallet.json)
```

**Analogy**: This is like creating test customers with sample money to try your service.

### Interacting with Your Program

For simple testing, you can use the CLI to call your program:

```bash
# Create a test account with your program
solana program call \
    --program-id your_program_id \
    --keypair test-wallet.json \
    -- \
    initialize 42 \
    --account new-account:keypair:account.json
```

For Anchor programs, use the Anchor CLI:

```bash
# Run a test script
anchor test

# Or deploy and run a specific test
anchor deploy
anchor run test-name
```

**Analogy**: This is like manually sending test orders to your kitchen to see how they're processed.

### Script-Based Testing

For more complex tests, create TypeScript tests with Anchor:

```typescript
// tests/your-program.ts
import * as anchor from '@coral-xyz/anchor';
import { Program } from '@coral-xyz/anchor';
import { YourProgram } from '../target/types/your_program';
import { expect } from 'chai';

describe('your-program', () => {
  // Configure the client
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
  const program = anchor.workspace.YourProgram as Program<YourProgram>;
  
  const wallet = provider.wallet;
  
  it('Initializes a counter', async () => {
    // Generate a new account keypair
    const counterKeypair = anchor.web3.Keypair.generate();
    
    // Call the initialize instruction
    await program.methods
      .initialize(new anchor.BN(5))
      .accounts({
        counter: counterKeypair.publicKey,
        user: wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([counterKeypair])
      .rpc();
    
    // Fetch the account and check its data
    const counterAccount = await program.account.counter.fetch(counterKeypair.publicKey);
    expect(counterAccount.count.toNumber()).to.equal(5);
  });
  
  it('Increments a counter', async () => {
    // Use the same account from the previous test
    const counterKeypair = anchor.web3.Keypair.generate();
    
    // Initialize with 5
    await program.methods
      .initialize(new anchor.BN(5))
      .accounts({
        counter: counterKeypair.publicKey,
        user: wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([counterKeypair])
      .rpc();
    
    // Increment
    await program.methods
      .increment()
      .accounts({
        counter: counterKeypair.publicKey,
      })
      .rpc();
    
    // Check the new value
    const counterAccount = await program.account.counter.fetch(counterKeypair.publicKey);
    expect(counterAccount.count.toNumber()).to.equal(6);
  });
});
```

**Analogy**: This is like creating a standardized test procedure for your kitchen that automatically tests multiple aspects of your service.

### Reading Program Output

After running operations, check the transaction logs:

```bash
# Get transaction details with verbose output
solana confirm -v TX_SIGNATURE

# View account data
solana account ACCOUNT_ADDRESS --output json
```

For Anchor accounts, use the Anchor CLI:

```bash
# Fetch and parse an Anchor account
anchor account ACCOUNT_ADDRESS
```

**Analogy**: This is like checking the results of your test to see if the orders were fulfilled correctly.

### Advanced Testing with Programs

For more sophisticated tests, use the Solana Test Validator with your client:

```typescript
// TypeScript test with local validator
import {
  Connection,
  Keypair,
  LAMPORTS_PER_SOL,
  PublicKey,
} from '@solana/web3.js';
import { execSync } from 'child_process';

async function main() {
  // Start a local validator (you can also do this in a separate terminal)
  console.log('Starting local validator...');
  const validatorProcess = execSync('solana-test-validator --reset', { stdio: 'inherit' });
  
  // Wait for validator to start
  await new Promise(resolve => setTimeout(resolve, 2000));
  
  // Connect to the local network
  const connection = new Connection('http://localhost:8899', 'confirmed');
  
  // Create a test wallet
  const wallet = Keypair.generate();
  
  // Fund the wallet
  const airdropSignature = await connection.requestAirdrop(
    wallet.publicKey,
    2 * LAMPORTS_PER_SOL
  );
  await connection.confirmTransaction(airdropSignature);
  
  // Deploy your program (if needed)
  // ...
  
  // Run your tests
  // ...
  
  console.log('Test completed successfully!');
}

main().catch(console.error);
```

**Analogy**: This is like having an automated testing suite that runs complex scenarios through your system.

## Transaction Best Practices

### 1. Minimize Transaction Size

Solana has transaction size limits (1232 bytes). To stay under the limit:
- Batch related operations into single instructions
- Use compact data encodings
- Split very large operations into multiple transactions

```typescript
// Better: Batch related operations
const tx = new Transaction();

// Add multiple instructions to a single transaction
tx.add(
    program.instruction.initializeUser(params, {
        accounts: { /* accounts */ }
    }),
    program.instruction.fundUser(amount, {
        accounts: { /* accounts */ }
    })
);

// If you need to split into multiple transactions due to size:
const txBatch1 = new Transaction();
const txBatch2 = new Transaction();

// Add instructions to each batch
// ...

// Send them sequentially
const sig1 = await sendAndConfirmTransaction(connection, txBatch1, [wallet]);
const sig2 = await sendAndConfirmTransaction(connection, txBatch2, [wallet]);
```

**Analogy**: This is like packing efficiently for a trip with a strict luggage weight limit.

### 2. Optimize for Cost

Transactions incur costs based on:
- Signatures (most expensive)
- Account access (read/write)
- Compute units used

Optimize by:
- Reducing signature count
- Minimizing account access
- Being computation-efficient

```typescript
// Set compute unit limit for a transaction (if you need more than default)
const modifyComputeUnits = ComputeBudgetProgram.setComputeUnitLimit({ 
  units: 200_000 // Default is 200k, max is 1.4M
});

// Set compute unit price (useful during network congestion)
const priorityFee = ComputeBudgetProgram.setComputeUnitPrice({ 
  microLamports: 1_000 // Priority fee of 0.000001 SOL per compute unit
});

// Add these instructions to your transaction
transaction.add(modifyComputeUnits, priorityFee);
```

**Analogy**: This is like planning a road trip to minimize gas and toll costs.

### 3. Handle Transaction Failures Gracefully

Transactions might fail for many reasons:
- Network congestion
- Insufficient funds
- Account constraints

Add retry logic with exponential backoff:

```typescript
async function sendWithRetry(
  connection: Connection,
  transaction: Transaction,
  signers: Keypair[],
  retries = 3
) {
  let lastError;
  
  for (let i = 0; i < retries; i++) {
    try {
      // Get a new blockhash for each attempt
      const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
      transaction.recentBlockhash = blockhash;
      
      const signature = await sendAndConfirmTransaction(
        connection, 
        transaction, 
        signers,
        {
          skipPreflight: false, // Run preflight checks
          commitment: 'confirmed',
          maxRetries: 5 // Built-in retries for network issues
        }
      );
      
      console.log(`Transaction confirmed: https://explorer.solana.com/tx/${signature}`);
      return signature;
    } catch (error) {
      console.log(`Attempt ${i+1} failed, retrying...`, error);
      lastError = error;
      
      // Check if we should retry based on error type
      if (error.toString().includes('blockhash not found') || 
          error.toString().includes('Block height exceeded')) {
        // Definitely retry these errors
      } else if (error.toString().includes('insufficient funds')) {
        // Don't retry these errors
        throw error;
      }
      
      // Wait longer between each retry
      await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, i)));
    }
  }
  
  throw lastError;
}
```

**Analogy**: This is like having a backup plan for when your primary route has traffic jams.

### 4. Versioned Transactions

Solana now supports versioned transactions (v0), which offer advantages like address lookup tables for working with many accounts:

```typescript
import { 
  TransactionMessage, 
  VersionedTransaction, 
  AddressLookupTableProgram 
} from '@solana/web3.js';

// Create a lookup table (one-time setup)
const [lookupTableInstr, lookupTableAddress] = AddressLookupTableProgram.createLookupTable({
  authority: wallet.publicKey,
  payer: wallet.publicKey,
  recentSlot: await connection.getSlot(),
});

// Add addresses to the lookup table
const addAddressesInstr = AddressLookupTableProgram.extendLookupTable({
  payer: wallet.publicKey,
  authority: wallet.publicKey,
  lookupTable: lookupTableAddress,
  addresses: [address1, address2, address3, /* ... many more addresses */],
});

// Later, use the lookup table in a versioned transaction
const lookupTableAccount = await connection.getAddressLookupTable(lookupTableAddress).then(res => res.value);

// Create a v0 transaction
const messageV0 = new TransactionMessage({
  payerKey: wallet.publicKey,
  recentBlockhash: (await connection.getLatestBlockhash()).blockhash,
  instructions: [/* your instructions */],
}).compileToV0Message([lookupTableAccount]);

const transactionV0 = new VersionedTransaction(messageV0);

// Sign and send
transactionV0.sign([wallet]);
const signature = await connection.sendTransaction(transactionV0);
```

**Analogy**: This is like using a contact list shortcut instead of typing out full addresses each time.

## Summary

In this session, we've covered:

1. The complete transaction lifecycle on Solana:
   - Creation
   - Signing
   - Submission
   - Execution
   - Confirmation

2. Signatures and multisig transactions:
   - How signatures work
   - Creating multisig accounts
   - Implementing multisig logic with Anchor

3. Program testing with Solana CLI and Anchor:
   - Setting up test environments
   - Creating test accounts
   - Interacting with programs
   - Reading program output
   - Advanced testing techniques

4. Transaction best practices:
   - Minimizing transaction size
   - Optimizing for cost
   - Handling failures gracefully
   - Using versioned transactions

Understanding these concepts will help you build more robust Solana programs and effectively debug issues that arise during development.

Next steps:
1. Try implementing a multisig wallet on Solana
2. Create a comprehensive test suite for your programs
3. Experiment with complex transaction patterns (e.g., atomic swaps)
4. Learn about versioned transactions and address lookup tables for scaling