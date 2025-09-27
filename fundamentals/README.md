# Cardano Fundamentals

Essential concepts and terminology for understanding the Cardano blockchain ecosystem.

## Table of Contents

- [Core Architecture](#core-architecture)
- [Key Concepts](#key-concepts)
- [Consensus Mechanism](#consensus-mechanism)
- [Native Tokens](#native-tokens)
- [Addresses](#addresses)
- [UTxO Model](#utxo-model)
- [Eras and Upgrades](#eras-and-upgrades)

## Core Architecture

### Layers
Cardano uses a layered architecture:

- **Cardano Settlement Layer (CSL)**: Handles ADA transactions and native tokens
- **Cardano Computation Layer (CCL)**: Planned layer for smart contracts and complex computations

### Node Components
- **Cardano Node**: Core blockchain node software
- **Cardano CLI**: Command-line interface for interacting with the blockchain
- **Cardano DB Sync**: Database synchronization service

## Key Concepts

### ADA
- **ADA**: Cardano's native cryptocurrency
- **Lovelace**: Smallest unit of ADA (1 ADA = 1,000,000 Lovelaces)
- **Symbol**: ₳ (ADA)

### Stake Pools
- **Stake Pool Operator (SPO)**: Entity that runs a stake pool
- **Stake Pool**: Infrastructure that validates transactions and produces blocks
- **Delegation**: Process of delegating ADA to a stake pool for rewards

### Epochs and Slots
- **Epoch**: Period of 5 days (432,000 slots)
- **Slot**: 1-second time unit for block production
- **Slot Leader**: Node selected to produce a block in a specific slot

## Consensus Mechanism

### Ouroboros Proof-of-Stake
- **Algorithm**: Ouroboros family of consensus protocols
- **Current Version**: Ouroboros Praos
- **Energy Efficiency**: Significantly more energy-efficient than Proof-of-Work
- **Security**: Mathematically provable security guarantees

### Randomness and Selection
- **VRF**: Verifiable Random Function for leader selection
- **Sigma Protocols**: Cryptographic proofs for stake verification

## Native Tokens

### Multi-Asset Support
- **Native Tokens**: Tokens that exist natively on Cardano without smart contracts
- **Policy ID**: Unique identifier for a token policy
- **Asset Name**: Human-readable name for a token
- **Minting Policy**: Rules governing token creation and destruction

### Token Types
- **Fungible Tokens**: Interchangeable tokens (like currencies)
- **Non-Fungible Tokens (NFTs)**: Unique, non-interchangeable tokens
- **Semi-Fungible Tokens**: Tokens that can be both fungible and non-fungible

## Addresses

### Address Types
- **Shelley Addresses**: Modern address format (starts with `addr1`)
- **Byron Addresses**: Legacy address format
- **Enterprise Addresses**: Addresses without staking rights
- **Reward Addresses**: Addresses for receiving staking rewards

### Address Components
- **Payment Part**: Controls spending of funds
- **Staking Part**: Controls delegation and rewards
- **Network Tag**: Identifies mainnet, testnet, or other networks

## UTxO Model

### Unspent Transaction Outputs
- **UTxO**: Unspent Transaction Output model
- **Extended UTxO (EUTxO)**: Cardano's extended version with smart contract support
- **Transaction Inputs**: UTxOs being spent
- **Transaction Outputs**: New UTxOs being created

### Benefits
- **Parallelism**: Transactions can be processed in parallel
- **Predictability**: Transaction fees and behavior are predictable
- **Security**: Each UTxO is independent, reducing attack vectors

## Eras and Upgrades

### Hard Fork Combinator
- **Seamless Upgrades**: Ability to upgrade without network interruption
- **Era Transitions**: Smooth transitions between protocol versions

### Historical Eras
- **Byron Era**: Foundation and basic functionality
- **Shelley Era**: Decentralization and staking
- **Allegra Era**: Token locking and metadata
- **Mary Era**: Native tokens and multi-assets
- **Alonzo Era**: Smart contracts and Plutus
- **Babbage Era**: Reference inputs and inline datums

### Current Era Features
- **Plutus Smart Contracts**: Functional programming for smart contracts
- **Reference Inputs**: Read-only transaction inputs
- **Inline Datums**: Datums stored directly in UTxOs
- **Reference Scripts**: Reusable scripts referenced in transactions

---

**Next Steps**: 
- Explore [Development Guides](../development/README.md) for practical implementation
- Learn about [Smart Contracts](../smart-contracts/README.md) for advanced functionality
- Check [Network Information](../network-info/README.md) for environment details