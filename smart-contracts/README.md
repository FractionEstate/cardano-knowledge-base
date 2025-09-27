# Smart Contracts on Cardano

Comprehensive guide to developing smart contracts on Cardano using Plutus and Aiken.

## Table of Contents

- [Overview](#overview)
- [Plutus Development](#plutus-development)
- [Aiken Development](#aiken-development)
- [Contract Types](#contract-types)
- [Development Workflow](#development-workflow)
- [Testing Strategies](#testing-strategies)
- [Deployment Guide](#deployment-guide)
- [Common Patterns](#common-patterns)

## Overview

### Smart Contract Languages
- **Plutus**: Haskell-based, functional programming
- **Aiken**: Modern, Rust-inspired syntax
- **Native Scripts**: Simple validation logic

### Key Concepts
- **Validators**: Scripts that validate spending conditions
- **Datums**: Data attached to UTxOs
- **Redeemers**: Data provided when spending UTxOs
- **Script Context**: Transaction information available to validators

## Plutus Development

### Setup
```bash
# Install GHC and Cabal
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh

# Create new Plutus project
cabal init plutus-project
cd plutus-project

# Add Plutus dependencies to cabal.file
```

### Basic Validator Structure
```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeApplications #-}

import Plutus.V2.Ledger.Api
import Plutus.V2.Ledger.Contexts
import PlutusTx
import PlutusTx.Prelude

-- Custom datum type
data MyDatum = MyDatum
  { owner :: PubKeyHash
  , amount :: Integer
  } deriving Show

PlutusTx.unstableMakeIsData ''MyDatum

-- Custom redeemer type
data MyRedeemer = Unlock | Burn
  deriving Show

PlutusTx.unstableMakeIsData ''MyRedeemer

-- Validator function
{-# INLINABLE myValidator #-}
myValidator :: MyDatum -> MyRedeemer -> ScriptContext -> Bool
myValidator dat red ctx = case red of
  Unlock -> traceIfFalse "Wrong signature" checkSignature
  Burn   -> traceIfFalse "Not burning" checkBurn
  where
    info :: TxInfo
    info = scriptContextTxInfo ctx
    
    checkSignature :: Bool
    checkSignature = txSignedBy info (owner dat)
    
    checkBurn :: Bool
    checkBurn = True -- Add burn logic here

-- Compile validator
validator :: Validator
validator = mkValidatorScript $$(PlutusTx.compile [|| myValidator ||])

-- Script address
scriptAddress :: Ledger.Address
scriptAddress = scriptHashAddress (validatorHash validator)
```

### Plutus Libraries
- **plutus-ledger-api**: Core Plutus types and functions
- **plutus-tx**: Template Haskell and compilation
- **plutus-script-utils**: Utility functions for common patterns
- **cardano-api**: Integration with Cardano ecosystem

## Aiken Development

### Setup
```bash
# Install Aiken
cargo install aiken

# Create new project
aiken new my-project
cd my-project

# Build project
aiken build
```

### Basic Validator Structure
```rust
use aiken/hash.{Blake2b_224, Hash}
use aiken/list
use aiken/transaction.{ScriptContext, Spend, Transaction}
use aiken/transaction/credential.{VerificationKey}

type Datum {
  owner: Hash<Blake2b_224, VerificationKey>,
  amount: Int,
}

type Redeemer {
  Unlock
  Burn
}

validator {
  fn my_validator(datum: Datum, redeemer: Redeemer, context: ScriptContext) -> Bool {
    when context.purpose is {
      Spend(_) ->
        when redeemer is {
          Unlock -> check_signature(context.transaction, datum.owner)
          Burn -> check_burn(context.transaction)
        }
      _ -> False
    }
  }
}

fn check_signature(transaction: Transaction, owner: Hash<Blake2b_224, VerificationKey>) -> Bool {
  list.has(transaction.extra_signatories, owner)
}

fn check_burn(transaction: Transaction) -> Bool {
  // Add burn logic here
  True
}
```

### Aiken Features
- **Modern Syntax**: Rust-inspired, easier to read
- **Built-in Testing**: Integrated testing framework
- **Better Tooling**: LSP support, formatter, documentation generator
- **Type Safety**: Strong static typing with inference

## Contract Types

### Spending Validators
- Validate spending of UTxOs at script addresses
- Most common type of smart contract
- Examples: Escrow, vesting, atomic swaps

### Minting Policies
- Control creation and destruction of native tokens
- Define rules for token minting/burning
- Examples: NFT collections, utility tokens

### Staking Validators
- Validate staking-related operations
- Control delegation and reward withdrawal
- Examples: Liquid staking, staking pools

### Certificate Validators
- Validate certificate operations
- Handle governance and delegation certificates
- Examples: DAO governance, delegation management

## Development Workflow

### 1. Design Phase
- Define data structures (Datum, Redeemer)
- Specify validation logic
- Consider edge cases and security implications

### 2. Implementation Phase
- Write validator in Plutus or Aiken
- Implement necessary helper functions
- Add proper error handling and tracing

### 3. Compilation Phase
```bash
# Plutus compilation
cabal run plutus-project

# Aiken compilation
aiken build
```

### 4. Testing Phase
- Unit tests for individual functions
- Property-based testing
- Integration tests with transactions

### 5. Deployment Phase
- Generate script address
- Submit script to blockchain
- Test on testnet before mainnet

## Testing Strategies

### Unit Testing (Aiken)
```rust
test my_validator_unlock_success() {
  let datum = Datum { owner: #"abc123", amount: 1000 }
  let redeemer = Unlock
  let context = mock_spend_context(datum.owner)
  
  my_validator(datum, redeemer, context)
}

test my_validator_unlock_failure() {
  let datum = Datum { owner: #"abc123", amount: 1000 }
  let redeemer = Unlock
  let context = mock_spend_context(#"wrong_key")
  
  !my_validator(datum, redeemer, context)
}
```

### Integration Testing
- Use cardano-cli for end-to-end testing
- Test with actual transactions on testnet
- Verify script execution and costs

### Property-Based Testing
- Test with randomly generated inputs
- Verify invariants hold across all inputs
- Use QuickCheck (Haskell) or similar tools

## Deployment Guide

### Script Compilation
```bash
# Generate script files
aiken build
# or for Plutus
cabal run compile-scripts

# Script files generated:
# - validator.plutus (CBOR-encoded script)
# - validator.hash (script hash)
# - validator.addr (script address)
```

### Script Submission
```bash
# Calculate script address
cardano-cli address build \
  --payment-script-file validator.plutus \
  --testnet-magic 2

# Send ADA to script address (locking funds)
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $SCRIPT_ADDRESS+2000000 \
  --tx-out-datum-hash $DATUM_HASH \
  --change-address $CHANGE_ADDRESS \
  --testnet-magic 2 \
  --out-file tx.raw

# Unlock funds from script
cardano-cli transaction build \
  --tx-in $SCRIPT_UTXO \
  --tx-in-script-file validator.plutus \
  --tx-in-datum-file datum.json \
  --tx-in-redeemer-file redeemer.json \
  --tx-out $RECIPIENT_ADDRESS+1800000 \
  --change-address $CHANGE_ADDRESS \
  --testnet-magic 2 \
  --out-file unlock-tx.raw
```

## Common Patterns

### Access Control
```rust
fn check_owner_signature(tx: Transaction, owner: ByteArray) -> Bool {
  list.has(tx.extra_signatories, owner)
}
```

### Time-Based Conditions
```rust
fn check_deadline(tx: Transaction, deadline: Int) -> Bool {
  when tx.validity_range.upper_bound.bound_type is {
    Finite(upper) -> upper <= deadline
    _ -> False
  }
}
```

### Value Preservation
```rust
fn check_value_preserved(inputs: List<Value>, outputs: List<Value>) -> Bool {
  list.foldl(inputs, zero(), add) == list.foldl(outputs, zero(), add)
}
```

### Oracle Integration
```rust
fn get_oracle_price(tx: Transaction, oracle_nft: AssetName) -> Option<Int> {
  // Look for UTxO containing oracle NFT
  // Extract price from datum
}
```

---

**Next Steps**:
- Review [Transaction Building](../transaction-building/README.md) for off-chain integration
- Check [Testing](../testing/README.md) for comprehensive testing strategies
- See [Best Practices](../best-practices/README.md) for security guidelines
- Explore [Tools](../tools/README.md) for development utilities