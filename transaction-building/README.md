# Transaction Building on Cardano

Comprehensive guide to building, signing, and submitting transactions on Cardano.

## Table of Contents

- [Overview](#overview)
- [Transaction Anatomy](#transaction-anatomy)
- [Building Transactions](#building-transactions)
- [Transaction Types](#transaction-types)
- [Fee Calculation](#fee-calculation)
- [Signing Process](#signing-process)
- [Submission](#submission)
- [Common Patterns](#common-patterns)

## Overview

### UTxO-Based Transactions
Cardano uses the Extended UTxO (EUTxO) model where:
- Transactions consume UTxOs as inputs
- Transactions create new UTxOs as outputs
- Each UTxO can only be spent once
- Transaction fees are calculated based on size and script execution

### Transaction Lifecycle
1. **Build** - Construct transaction with inputs/outputs
2. **Balance** - Ensure inputs ≥ outputs + fees
3. **Sign** - Add cryptographic signatures
4. **Submit** - Send to network for validation
5. **Confirm** - Wait for blockchain inclusion

## Transaction Anatomy

### Core Components
```javascript
{
  "inputs": [
    {
      "transaction_id": "abc123...",
      "output_index": 0
    }
  ],
  "outputs": [
    {
      "address": "addr1...",
      "amount": {
        "coin": "1000000",
        "multiasset": {
          "policy_id": {
            "asset_name": "quantity"
          }
        }
      },
      "datum_hash": "def456...", // Optional
      "script_ref": {...}        // Optional
    }
  ],
  "fee": "165000",
  "ttl": 85000000,               // Optional
  "certificates": [],            // Optional
  "withdrawals": {},             // Optional
  "validity_interval": {         // Optional
    "invalid_before": 84000000,
    "invalid_hereafter": 85000000
  },
  "mint": {},                    // Optional
  "script_data_hash": "...",     // Optional when using scripts
  "collateral": [],              // Required for script transactions
  "required_signers": [],        // Optional
  "network_id": 1
}
```

### Extended UTxO Features
- **Datum**: Data attached to UTxOs for smart contracts
- **Script Reference**: Scripts attached to UTxOs for reuse
- **Inline Datum**: Datum stored directly in UTxO
- **Reference Inputs**: Read-only transaction inputs

## Building Transactions

### Using cardano-cli
```bash
# Simple payment transaction
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $RECIPIENT_ADDRESS+1000000 \
  --change-address $SENDER_ADDRESS \
  --testnet-magic 2 \
  --out-file payment.raw

# Transaction with native tokens
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out "$RECIPIENT_ADDRESS+1000000+10 $POLICY_ID.$ASSET_NAME" \
  --change-address $SENDER_ADDRESS \
  --testnet-magic 2 \
  --out-file token-payment.raw

# Transaction with metadata
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $RECIPIENT_ADDRESS+1000000 \
  --change-address $SENDER_ADDRESS \
  --metadata-json-file metadata.json \
  --testnet-magic 2 \
  --out-file metadata-tx.raw
```

### Using Lucid (JavaScript)
```javascript
import { Blockfrost, Lucid } from "lucid-cardano";

const lucid = await Lucid.new(
  new Blockfrost("https://cardano-preview.blockfrost.io/api/v0", "project_id"),
  "Preview"
);

// Simple payment
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .complete();

// Payment with native tokens
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { 
    lovelace: 2000000n,
    [policyId + assetName]: 10n
  })
  .complete();

// Payment with metadata
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .attachMetadata(674, { msg: ["Hello, Cardano!"] })
  .complete();

// Payment with time constraints
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .validFrom(Date.now())
  .validTo(Date.now() + 300000) // 5 minutes
  .complete();
```

### Using PyCardano (Python)
```python
from pycardano import *

# Create chain context
context = BlockFrostChainContext(
    project_id="project_id",
    base_url="https://cardano-preview.blockfrost.io/api/v0"
)

# Build transaction
builder = TransactionBuilder(context)
builder.add_input_address(sender_address)
builder.add_output(
    TransactionOutput(
        address=recipient_address,
        amount=Value(coin=1000000)
    )
)

# Add native tokens
multi_asset = MultiAsset({
    ScriptHash(bytes.fromhex(policy_id)): Asset({
        AssetName(b"MyToken"): 10
    })
})

builder.add_output(
    TransactionOutput(
        address=recipient_address,
        amount=Value(coin=1000000, multi_asset=multi_asset)
    )
)

# Add metadata
metadata = AuxiliaryData(Metadata({674: {"msg": ["Hello, Cardano!"]}}))
builder.auxiliary_data = metadata

# Build and sign
signed_tx = builder.build_and_sign([signing_key], change_address=sender_address)
```

## Transaction Types

### Simple Payments
```javascript
// ADA only
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 5000000n })
  .complete();

// With native tokens
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { 
    lovelace: 2000000n,
    "policy123asset456": 100n
  })
  .complete();
```

### Delegation
```javascript
// Delegate to stake pool
const tx = await lucid
  .newTx()
  .delegateTo("pool1...", rewardAddress)
  .complete();

// Withdraw rewards
const tx = await lucid
  .newTx()
  .withdraw(rewardAddress, 5000000n)
  .complete();
```

### Minting/Burning
```javascript
// Mint native tokens
const mintingPolicy = lucid.utils.nativeScriptFromJson({
  type: "sig",
  keyHash: "your_key_hash"
});

const policyId = lucid.utils.mintingPolicyToId(mintingPolicy);

const tx = await lucid
  .newTx()
  .mintAssets({
    [policyId + "MyToken"]: 100n
  })
  .attachMintingPolicy(mintingPolicy)
  .complete();

// Burn tokens
const tx = await lucid
  .newTx()
  .mintAssets({
    [policyId + "MyToken"]: -50n
  })
  .attachMintingPolicy(mintingPolicy)
  .complete();
```

### Smart Contract Interaction
```javascript
// Lock funds in script
const scriptAddress = lucid.utils.validatorToAddress(validator);
const datum = Data.to(new Constr(0, [BigInt(42)]));

const tx = await lucid
  .newTx()
  .payToContract(scriptAddress, { inline: datum }, { lovelace: 10000000n })
  .complete();

// Unlock funds from script
const scriptUtxos = await lucid.utxosAt(scriptAddress);
const redeemer = Data.to(new Constr(0, []));

const tx = await lucid
  .newTx()
  .collectFrom(scriptUtxos, redeemer)
  .attachSpendingValidator(validator)
  .complete();
```

## Fee Calculation

### Fee Components
```
Total Fee = Base Fee + (Size Fee × Transaction Size) + Script Fee
```

### Linear Fee Formula
```
fee = a + b × size
```
Where:
- `a` = Fixed base fee (typically 155,381 lovelace)
- `b` = Fee per byte (typically 44 lovelace/byte)
- `size` = Transaction size in bytes

### Script Fees
- **Script Execution Units**: CPU and memory usage
- **Execution Price**: Cost per unit of CPU/memory
- **Reference Scripts**: Can reduce transaction size and fees

### Fee Estimation
```javascript
// Estimate with Lucid
const incompleteTx = lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n });

const estimatedFee = await incompleteTx.fee();
console.log(`Estimated fee: ${estimatedFee} lovelace`);

// Complete with estimated fee
const completeTx = await incompleteTx.complete();
const actualFee = completeTx.fee();
console.log(`Actual fee: ${actualFee} lovelace`);
```

### Fee Optimization
```javascript
// Use reference scripts to reduce fees
const tx = await lucid
  .newTx()
  .collectFrom(utxos, redeemer)
  .readFrom([referenceScriptUtxo]) // Reference script instead of attaching
  .complete();

// Batch operations to reduce per-transaction overhead
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 1000000n })
  .payToAddress("addr2...", { lovelace: 1000000n })
  .payToAddress("addr3...", { lovelace: 1000000n })
  .complete();
```

## Signing Process

### Single Signature
```javascript
// Sign with wallet
const signedTx = await tx.sign().complete();

// Sign with private key
const signedTx = await tx.signWithPrivateKey(privateKey).complete();
```

### Multi-Signature
```javascript
// Partial signing
const partiallySignedTx = await tx.partialSign();

// Additional signatures
const fullySignedTx = await partiallySignedTx
  .signWithPrivateKey(secondPrivateKey)
  .complete();

// Witness collection
const witness1 = await tx.partialSign();
const witness2 = await tx.signWithPrivateKey(secondKey).partialSign();

const signedTx = await tx
  .assemble([witness1, witness2])
  .complete();
```

### Hardware Wallet Signing
```javascript
// CIP-30 browser wallet
const api = await window.cardano.nami.enable();
const signedTx = await api.signTx(tx.toCBOR(), true);

// Ledger via cardano-hw-cli
const witnessFile = await executeCommand([
  'cardano-hw-cli', 'transaction', 'witness',
  '--tx-file', 'tx.raw',
  '--hw-signing-file', 'payment.hwsfile',
  '--out-file', 'witness.json'
]);
```

## Submission

### Network Submission
```javascript
// Submit with Lucid
const txHash = await signedTx.submit();
console.log(`Transaction submitted: ${txHash}`);

// Submit with cardano-cli
await executeCommand([
  'cardano-cli', 'transaction', 'submit',
  '--tx-file', 'tx.signed',
  '--testnet-magic', '2'
]);
```

### Confirmation Waiting
```javascript
// Wait for confirmation
async function waitForConfirmation(txHash, maxWait = 300000) {
  const startTime = Date.now();
  
  while (Date.now() - startTime < maxWait) {
    try {
      const tx = await lucid.provider.getTransaction(txHash);
      if (tx) {
        console.log(`Transaction confirmed in block: ${tx.blockHeight}`);
        return tx;
      }
    } catch (error) {
      // Transaction not yet confirmed
    }
    
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
  
  throw new Error('Transaction confirmation timeout');
}

const txHash = await signedTx.submit();
const confirmedTx = await waitForConfirmation(txHash);
```

### Error Handling
```javascript
try {
  const txHash = await signedTx.submit();
  console.log(`Success: ${txHash}`);
} catch (error) {
  if (error.message.includes('InsufficientFunds')) {
    console.error('Insufficient funds for transaction');
  } else if (error.message.includes('ValueNotConserved')) {
    console.error('Input and output values do not match');
  } else if (error.message.includes('ScriptFailure')) {
    console.error('Smart contract validation failed');
  } else {
    console.error('Transaction submission failed:', error.message);
  }
}
```

## Common Patterns

### UTxO Selection
```javascript
// Select UTxOs manually
const utxos = await lucid.wallet.getUtxos();
const selectedUtxos = utxos.slice(0, 2); // Select first 2 UTxOs

const tx = await lucid
  .newTx()
  .collectFrom(selectedUtxos)
  .payToAddress("addr1...", { lovelace: 2000000n })
  .complete();

// Automatic UTxO selection (default)
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .complete(); // Lucid automatically selects UTxOs
```

### Change Address Management
```javascript
// Explicit change address
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .addSigner(changeAddress)
  .complete();

// Automatic change handling (default)
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .complete(); // Change goes back to wallet
```

### Batch Processing
```javascript
// Process multiple payments efficiently
const recipients = [
  { address: "addr1...", amount: { lovelace: 1000000n } },
  { address: "addr2...", amount: { lovelace: 1500000n } },
  { address: "addr3...", amount: { lovelace: 2000000n } }
];

let tx = lucid.newTx();
for (const recipient of recipients) {
  tx = tx.payToAddress(recipient.address, recipient.amount);
}

const completeTx = await tx.complete();
const signedTx = await completeTx.sign().complete();
const txHash = await signedTx.submit();
```

### Time-Based Constraints
```javascript
// Transaction valid only after specific time
const validFrom = Date.now() + 60000; // 1 minute from now
const validTo = validFrom + 300000;   // 5 minutes window

const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .validFrom(validFrom)
  .validTo(validTo)
  .complete();
```

---

**Next Steps**:
- Review [Smart Contracts](../smart-contracts/README.md) for advanced transaction patterns
- Check [APIs](../apis/README.md) for transaction building services
- See [Testing](../testing/README.md) for transaction testing strategies
- Explore [Best Practices](../best-practices/README.md) for secure transaction handling