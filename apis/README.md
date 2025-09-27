# Cardano APIs and Services

Comprehensive guide to Cardano APIs, SDKs, and third-party services for blockchain integration.

## Table of Contents

- [Official APIs](#official-apis)
- [Third-Party Services](#third-party-services)
- [SDK Libraries](#sdk-libraries)
- [WebSocket APIs](#websocket-apis)
- [Authentication](#authentication)
- [Rate Limits](#rate-limits)
- [Error Handling](#error-handling)

## Official APIs

### Cardano Node API
**Direct node interaction via local socket**

```bash
# Query tip
cardano-cli query tip --testnet-magic 2

# Query UTxOs
cardano-cli query utxo --address $ADDRESS --testnet-magic 2

# Submit transaction
cardano-cli transaction submit --tx-file tx.signed --testnet-magic 2
```

**Use Cases**: Direct blockchain queries, transaction submission, local development

### Cardano Wallet API
**RESTful API for wallet operations**

```bash
# Base URL
https://localhost:8090/v2/

# Create wallet
curl -X POST \
  https://localhost:8090/v2/wallets \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "My Wallet",
    "mnemonic_sentence": ["word1", "word2", ...],
    "passphrase": "my-secret-passphrase"
  }'

# Get wallet balance
curl https://localhost:8090/v2/wallets/{walletId}

# Send transaction
curl -X POST \
  https://localhost:8090/v2/wallets/{walletId}/transactions \
  -H 'Content-Type: application/json' \
  -d '{
    "payments": [{
      "address": "addr1...",
      "amount": {
        "quantity": 1000000,
        "unit": "lovelace"
      }
    }],
    "passphrase": "my-secret-passphrase"
  }'
```

**Endpoints**:
- `GET /v2/wallets` - List wallets
- `POST /v2/wallets` - Create wallet
- `GET /v2/wallets/{walletId}` - Get wallet details
- `POST /v2/wallets/{walletId}/transactions` - Send transaction
- `GET /v2/wallets/{walletId}/transactions` - List transactions

## Third-Party Services

### Blockfrost
**Instant Cardano API access**

```javascript
// Setup
const BlockFrost = require('@blockfrost/blockfrost-js');
const API = new BlockFrost.BlockFrostAPI({
  projectId: 'your-project-id',
  network: 'testnet'
});

// Get latest block
const latestBlock = await API.blocksLatest();

// Get address info
const addressInfo = await API.addresses('addr1...');

// Get UTxOs
const utxos = await API.addressesUtxos('addr1...');

// Get transaction details
const tx = await API.txs('tx-hash');
```

**Key Features**:
- ✅ No node required
- ✅ High availability
- ✅ Comprehensive endpoints
- ✅ WebSocket support
- ✅ IPFS integration

**Endpoints**:
- `/blocks` - Block information
- `/addresses` - Address details and UTxOs
- `/transactions` - Transaction data
- `/assets` - Native token information
- `/pools` - Stake pool data
- `/epochs` - Epoch information

### Koios
**Community-driven Cardano API**

```bash
# Base URLs
# Mainnet: https://api.koios.rest/api/v1
# Preview: https://preview.koios.rest/api/v1
# Preprod: https://preprod.koios.rest/api/v1

# Get chain tip
curl https://api.koios.rest/api/v1/tip

# Get address info
curl https://api.koios.rest/api/v1/address_info \
  -H "Content-Type: application/json" \
  -d '["addr1..."]'

# Get UTxOs
curl https://api.koios.rest/api/v1/address_utxos \
  -H "Content-Type: application/json" \
  -d '["addr1..."]'
```

**Key Features**:
- ✅ Free to use
- ✅ Open source
- ✅ Multiple networks
- ✅ Batch requests
- ✅ No rate limits

### Maestro
**Enterprise-grade Cardano API**

```javascript
// Setup
const { MaestroClient } = require('@maestro-org/typescript-sdk');

const maestro = new MaestroClient({
  apiKey: 'your-api-key',
  network: 'Preprod'
});

// Get UTxOs
const utxos = await maestro.addresses.addressUtxos('addr1...');

// Get transaction details
const tx = await maestro.transactions.transactionDetails('tx-hash');

// Submit transaction
const result = await maestro.transactions.submitTransaction(signedTxCbor);
```

**Key Features**:
- ✅ Enterprise SLA
- ✅ High performance
- ✅ Advanced features
- ✅ Dedicated support

## SDK Libraries

### JavaScript/TypeScript

#### Lucid
```javascript
import { Blockfrost, Lucid } from "lucid-cardano";

const lucid = await Lucid.new(
  new Blockfrost("https://cardano-preview.blockfrost.io/api/v0", "project_id"),
  "Preview"
);

// Select wallet
lucid.selectWalletFromSeed("your mnemonic phrase here");

// Build transaction
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 5000000n })
  .complete();

const signedTx = await tx.sign().complete();
const txHash = await signedTx.submit();
```

#### Mesh
```javascript
import { MeshWallet, BlockfrostProvider } from '@meshsdk/core';

const blockchainProvider = new BlockfrostProvider('project_id');
const wallet = new MeshWallet({
  networkId: 0,
  fetcher: blockchainProvider,
  submitter: blockchainProvider,
  key: {
    type: 'mnemonic',
    words: 'your mnemonic phrase here'.split(' '),
  },
});

// Send transaction
const tx = await wallet.createTx()
  .txOut('addr1...', [{ unit: 'lovelace', quantity: '1000000' }])
  .complete();

const signedTx = await wallet.signTx(tx);
const txHash = await wallet.submitTx(signedTx);
```

#### cardano-serialization-lib
```javascript
import * as CardanoWasm from "@emurgo/cardano-serialization-lib-nodejs";

// Build transaction output
const shelleyOutputAddress = CardanoWasm.Address.from_bech32('addr1...');
const shelleyChangeAddress = CardanoWasm.Address.from_bech32('addr1...');

const output = CardanoWasm.TransactionOutput.new(
  shelleyOutputAddress,
  CardanoWasm.Value.new(CardanoWasm.BigNum.from_str('1000000'))
);

// Build transaction
const txBuilder = CardanoWasm.TransactionBuilder.new(
  CardanoWasm.LinearFee.new(
    CardanoWasm.BigNum.from_str('44'),
    CardanoWasm.BigNum.from_str('155381')
  ),
  CardanoWasm.BigNum.from_str('1000000'),
  CardanoWasm.BigNum.from_str('500000000'),
  CardanoWasm.BigNum.from_str('2000000')
);
```

### Python

#### PyCardano
```python
from pycardano import *

# Create blockchain context
context = BlockFrostChainContext(
    project_id="your_project_id",
    base_url="https://cardano-preview.blockfrost.io/api/v0"
)

# Create payment signing key
payment_skey = PaymentSigningKey.generate()
payment_vkey = PaymentVerificationKey.from_signing_key(payment_skey)

# Derive address
address = Address(payment_vkey.hash())

# Build transaction
builder = TransactionBuilder(context)
builder.add_input_address(address)
builder.add_output(TransactionOutput(recipient_address, Value(1000000)))

# Sign and submit
signed_tx = builder.build_and_sign([payment_skey], change_address=address)
context.submit_tx(signed_tx)
```

### Rust

#### cardano-serialization-lib (Rust)
```rust
use cardano_serialization_lib as csl;

// Create transaction builder
let linear_fee = csl::LinearFee::new(
    &csl::BigNum::from_str("44").unwrap(),
    &csl::BigNum::from_str("155381").unwrap(),
);

let cfg = csl::TransactionBuilderConfigBuilder::new()
    .fee_algo(&linear_fee)
    .pool_deposit(&csl::BigNum::from_str("500000000").unwrap())
    .key_deposit(&csl::BigNum::from_str("2000000").unwrap())
    .max_value_size(5000)
    .max_tx_size(16384)
    .build();

let mut tx_builder = csl::TransactionBuilder::new(&cfg);

// Add output
let output_address = csl::Address::from_bech32("addr1...").unwrap();
let output_value = csl::Value::new(&csl::BigNum::from_str("1000000").unwrap());
let output = csl::TransactionOutput::new(&output_address, &output_value);

tx_builder.add_output(&output).unwrap();
```

## WebSocket APIs

### Ogmios
**WebSocket interface to Cardano node**

```javascript
import { createInteractionContext, findIntersection } from '@cardano-ogmios/client';

const context = await createInteractionContext(
  (error) => console.error(error),
  () => console.log('Connection closed'),
  { connection: { host: 'localhost', port: 1337 } }
);

// Find intersection
const intersection = await findIntersection(context, points);

// Local state query
const utxos = await queryUtxos(context, addresses);
```

**Key Features**:
- ✅ Real-time blockchain data
- ✅ State queries
- ✅ Chain synchronization
- ✅ Low latency

### Blockfrost WebSocket
```javascript
const WebSocket = require('ws');

const ws = new WebSocket('wss://cardano-mainnet.blockfrost.io/api/ws', {
  headers: {
    'project_id': 'your-project-id'
  }
});

ws.on('message', (data) => {
  const message = JSON.parse(data);
  console.log('Received:', message);
});

// Subscribe to address transactions
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'address',
  address: 'addr1...'
}));
```

## Authentication

### API Keys
Most services require API keys:

```javascript
// Blockfrost
const API = new BlockFrost.BlockFrostAPI({
  projectId: 'mainnet_your_project_id_here'
});

// Maestro
const maestro = new MaestroClient({
  apiKey: 'your-api-key',
  network: 'Mainnet'
});
```

### IP Whitelisting
Some enterprise services offer IP whitelisting for enhanced security.

## Rate Limits

### Blockfrost
- **Free**: 10 requests/second, 50,000 requests/day
- **Paid Plans**: Higher limits based on subscription

### Koios
- **No rate limits**: Community-driven, best effort

### Maestro
- **Custom limits**: Based on subscription tier

### Best Practices
```javascript
// Implement retry logic
async function apiCall(fn, retries = 3) {
  try {
    return await fn();
  } catch (error) {
    if (error.status === 429 && retries > 0) {
      await new Promise(resolve => setTimeout(resolve, 1000));
      return apiCall(fn, retries - 1);
    }
    throw error;
  }
}

// Use caching
const cache = new Map();
async function getCachedData(key, fetchFn, ttl = 60000) {
  if (cache.has(key)) {
    const { data, timestamp } = cache.get(key);
    if (Date.now() - timestamp < ttl) {
      return data;
    }
  }
  
  const data = await fetchFn();
  cache.set(key, { data, timestamp: Date.now() });
  return data;
}
```

## Error Handling

### Common Error Patterns
```javascript
try {
  const result = await API.addresses('addr1...');
} catch (error) {
  switch (error.status_code) {
    case 400:
      console.error('Bad Request:', error.message);
      break;
    case 404:
      console.error('Not Found:', error.message);
      break;
    case 429:
      console.error('Rate Limited:', error.message);
      // Implement backoff
      break;
    case 500:
      console.error('Server Error:', error.message);
      // Retry logic
      break;
    default:
      console.error('Unknown Error:', error);
  }
}
```

### Robust Error Handling
```javascript
class CardanoAPIClient {
  constructor(provider) {
    this.provider = provider;
    this.maxRetries = 3;
    this.baseDelay = 1000;
  }

  async withRetry(operation) {
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        if (attempt === this.maxRetries || !this.isRetryable(error)) {
          throw error;
        }
        
        const delay = this.baseDelay * Math.pow(2, attempt - 1);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  isRetryable(error) {
    return error.status >= 500 || error.status === 429;
  }
}
```

---

**Next Steps**:
- Explore [Transaction Building](../transaction-building/README.md) for practical implementation
- Check [Tools](../tools/README.md) for additional development utilities
- Review [Best Practices](../best-practices/README.md) for production-ready integrations
- See [Wallet Integration](../wallet-integration/README.md) for user-facing applications