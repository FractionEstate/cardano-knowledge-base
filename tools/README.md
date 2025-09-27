# Cardano Development Tools

Comprehensive guide to essential tools for Cardano blockchain development.

## Table of Contents

- [Core Tools](#core-tools)
- [Development Frameworks](#development-frameworks)
- [Testing Tools](#testing-tools)
- [Deployment Tools](#deployment-tools)
- [Monitoring Tools](#monitoring-tools)
- [Analysis Tools](#analysis-tools)
- [Utilities](#utilities)

## Core Tools

### Cardano Node
**Official blockchain node implementation**

```bash
# Installation
curl -sSL https://get.cardano.org/install.sh | bash

# Configuration
mkdir -p $HOME/cardano/{config,db,logs}
cd $HOME/cardano/config

# Download configs for preview testnet
wget https://book.world.dev.cardano.org/environments/preview/config.json
wget https://book.world.dev.cardano.org/environments/preview/topology.json
wget https://book.world.dev.cardano.org/environments/preview/byron-genesis.json
wget https://book.world.dev.cardano.org/environments/preview/shelley-genesis.json
wget https://book.world.dev.cardano.org/environments/preview/alonzo-genesis.json
wget https://book.world.dev.cardano.org/environments/preview/conway-genesis.json

# Run node
cardano-node run \
  --config config.json \
  --topology topology.json \
  --database-path ../db \
  --socket-path ../db/socket \
  --port 3001
```

**Use Cases**: Full node operations, block production, transaction validation

### Cardano CLI
**Command-line interface for Cardano operations**

```bash
# Key generation
cardano-cli address key-gen \
  --verification-key-file payment.vkey \
  --signing-key-file payment.skey

# Address generation
cardano-cli address build \
  --payment-verification-key-file payment.vkey \
  --testnet-magic 2 \
  --out-file payment.addr

# Query balance
cardano-cli query utxo \
  --address $(cat payment.addr) \
  --testnet-magic 2

# Build transaction
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $RECIPIENT_ADDRESS+1000000 \
  --change-address $SENDER_ADDRESS \
  --testnet-magic 2 \
  --out-file payment.raw

# Sign transaction
cardano-cli transaction sign \
  --tx-body-file payment.raw \
  --signing-key-file payment.skey \
  --testnet-magic 2 \
  --out-file payment.signed

# Submit transaction
cardano-cli transaction submit \
  --tx-file payment.signed \
  --testnet-magic 2
```

**Use Cases**: Transaction building, key management, network queries

### Cardano Wallet
**Backend service for wallet operations**

```bash
# Installation
wget https://github.com/IntersectMBO/cardano-wallet/releases/latest/download/cardano-wallet-linux64.tar.gz
tar -xzf cardano-wallet-linux64.tar.gz

# Run wallet server
cardano-wallet serve \
  --node-socket $CARDANO_NODE_SOCKET_PATH \
  --testnet byron-genesis.json \
  --database ./wallet-db

# API usage
curl -X POST http://localhost:8090/v2/wallets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Wallet",
    "mnemonic_sentence": ["word1", "word2", ...],
    "passphrase": "secure-passphrase"
  }'
```

**Use Cases**: Wallet management, transaction handling, balance queries

## Development Frameworks

### Lucid
**Lightweight Cardano library for JavaScript/TypeScript**

```bash
npm install lucid-cardano
```

```javascript
import { Blockfrost, Lucid } from "lucid-cardano";

const lucid = await Lucid.new(
  new Blockfrost("https://cardano-preview.blockfrost.io/api/v0", "project_id"),
  "Preview"
);

// Transaction building
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .complete();

const signedTx = await tx.sign().complete();
const txHash = await signedTx.submit();
```

**Features**:
- ✅ Simple API
- ✅ TypeScript support
- ✅ Multiple backend support
- ✅ Smart contract integration

### Mesh
**Full-stack framework for Cardano applications**

```bash
npm install @meshsdk/core @meshsdk/react
```

```javascript
import { MeshWallet, BlockfrostProvider } from '@meshsdk/core';

const blockchainProvider = new BlockfrostProvider('project_id');
const wallet = new MeshWallet({
  networkId: 0,
  fetcher: blockchainProvider,
  submitter: blockchainProvider,
  key: {
    type: 'mnemonic',
    words: mnemonic.split(' '),
  },
});

// React components
import { CardanoWallet } from '@meshsdk/react';

function App() {
  return (
    <CardanoWallet />
  );
}
```

**Features**:
- ✅ React components
- ✅ Wallet connectors
- ✅ Transaction builders
- ✅ Smart contract utilities

### PyCardano
**Python library for Cardano development**

```bash
pip install pycardano
```

```python
from pycardano import *

# Chain context
context = BlockFrostChainContext(
    project_id="your_project_id",
    base_url="https://cardano-preview.blockfrost.io/api/v0"
)

# Transaction building
builder = TransactionBuilder(context)
builder.add_input_address(sender_address)
builder.add_output(
    TransactionOutput(
        address=recipient_address,
        amount=Value(coin=1000000)
    )
)

signed_tx = builder.build_and_sign([signing_key], change_address=sender_address)
context.submit_tx(signed_tx)
```

**Features**:
- ✅ Pythonic API
- ✅ Full transaction support
- ✅ Smart contract integration
- ✅ Serialization utilities

### Aiken
**Modern smart contract language and toolchain**

```bash
# Install Aiken
cargo install aiken

# Create new project
aiken new my-project
cd my-project

# Build contracts
aiken build

# Run tests
aiken test

# Generate documentation
aiken docs
```

```rust
use aiken/transaction.{ScriptContext, Spend}
use aiken/transaction/credential.{VerificationKey}

type Datum {
  owner: Hash<Blake2b_224, VerificationKey>,
}

type Redeemer {
  msg: ByteArray,
}

validator {
  fn spend(datum: Datum, redeemer: Redeemer, context: ScriptContext) -> Bool {
    when context.purpose is {
      Spend(_) -> {
        let must_be_signed = 
          list.has(context.transaction.extra_signatories, datum.owner)
        
        must_be_signed && (redeemer.msg == "unlock")
      }
      _ -> False
    }
  }
}
```

**Features**:
- ✅ Modern syntax
- ✅ Built-in testing
- ✅ LSP support
- ✅ Documentation generation

## Testing Tools

### Cardano Testnet
**Local testnet for development**

```bash
# cardano-testnet (coming soon)
cardano-testnet --num-pool-nodes 3 --slot-length 0.2

# Alternative: Use public testnets
export CARDANO_TESTNET=preview
export CARDANO_NODE_SOCKET_PATH=$HOME/cardano/db/socket
```

### Plutip
**Plutus integration testing framework**

```haskell
import Test.Plutip.Config
import Test.Plutip.Contract

testConfig :: PlutipConfig
testConfig = PlutipConfig
  { pcSlotConfig = def
  , pcRootDir = Nothing
  }

test :: IO ()
test = runPlutipContract testConfig $ do
  -- Test your contracts here
  user1 <- newUser $ ada 100
  user2 <- newUser $ ada 50
  
  -- Interact with contracts
  result <- submitTx user1 someTransaction
  -- Assert results
```

### Emulator Testing
**Plutus Application Backend (PAB) emulator**

```haskell
import Plutus.PAB.Simulator

runSimulation :: IO ()
runSimulation = runSimulationWith simulatorHandlers $ do
  cidInit <- Simulator.activateContract (Wallet 1) MyContract
  cidUser <- Simulator.activateContract (Wallet 2) MyContract
  
  -- Simulate user interactions
  callEndpointOnInstance cidUser "buy" 42
  
  -- Wait for transactions
  void $ Simulator.waitForLedgerSlot 10
```

### Property-Based Testing
**QuickCheck for Plutus**

```haskell
import Test.QuickCheck

prop_ValidatorAlwaysSucceeds :: Integer -> Bool
prop_ValidatorAlwaysSucceeds x = 
  let datum = MyDatum x
      redeemer = MyRedeemer
      ctx = mockScriptContext
  in myValidator datum redeemer ctx

main :: IO ()
main = quickCheck prop_ValidatorAlwaysSucceeds
```

## Deployment Tools

### Nix
**Reproducible build system**

```nix
# shell.nix
{ pkgs ? import <nixpkgs> {} }:

pkgs.mkShell {
  buildInputs = with pkgs; [
    cardano-node
    cardano-cli
    cardano-wallet
    ghc
    cabal-install
  ];
  
  shellHook = ''
    export CARDANO_NODE_SOCKET_PATH=$PWD/node.socket
  '';
}
```

```bash
# Enter development environment
nix-shell

# Build with Nix
nix-build -A cardano-node
```

### Docker
**Containerized Cardano services**

```dockerfile
# Dockerfile
FROM inputoutput/cardano-node:latest

COPY config/ /opt/cardano/config/
COPY scripts/ /opt/cardano/scripts/

EXPOSE 3001

CMD ["/opt/cardano/scripts/run-node.sh"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  cardano-node:
    build: .
    ports:
      - "3001:3001"
    volumes:
      - ./db:/opt/cardano/db
      - ./config:/opt/cardano/config
    environment:
      - NETWORK=preview

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: cardano
      POSTGRES_USER: cardano
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  cardano-db-sync:
    image: inputoutput/cardano-db-sync:latest
    depends_on:
      - postgres
      - cardano-node
```

### CI/CD Pipelines
**GitHub Actions example**

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run tests
        run: npm test
        
      - name: Build contracts
        run: aiken build
        
      - name: Deploy to testnet
        if: github.ref == 'refs/heads/main'
        run: npm run deploy:testnet
        env:
          BLOCKFROST_API_KEY: ${{ secrets.BLOCKFROST_API_KEY }}
```

## Monitoring Tools

### Cardano GraphQL
**GraphQL interface for Cardano**

```bash
# Using Docker
docker run -d \
  --name cardano-graphql \
  -p 3100:3100 \
  -e POSTGRES_HOST=postgres \
  -e POSTGRES_DB=cardano \
  inputoutput/cardano-graphql
```

```graphql
query {
  cardano {
    tip {
      number
      hash
      slotNo
    }
  }
  
  transactions(
    limit: 10
    where: { outputs: { address: { _eq: "addr1..." } } }
  ) {
    hash
    totalOutput
    fee
  }
}
```

### Prometheus & Grafana
**Metrics and monitoring**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cardano-node'
    static_configs:
      - targets: ['localhost:12798']
```

```bash
# Enable Cardano node metrics
cardano-node run \
  --config config.json \
  --topology topology.json \
  --database-path db \
  --socket-path db/socket \
  --host-addr 0.0.0.0 \
  --port 3001 \
  --prometheus-port 12798
```

### Cardano RTView
**Real-time monitoring dashboard**

```bash
# Install RTView
cabal install cardano-rt-view

# Configuration
cardano-rt-view --config rt-view.json

# Access dashboard
# http://localhost:8024
```

## Analysis Tools

### Cardano DB Sync
**PostgreSQL database synchronization**

```bash
# Setup PostgreSQL
createdb cardano

# Run DB Sync
cardano-db-sync \
  --config config.json \
  --socket-path ../cardano/db/socket \
  --state-dir db-sync \
  --schema-dir schema/

# Query database
psql cardano -c "
  SELECT 
    encode(hash, 'hex') as tx_hash,
    out_sum / 1000000 as ada_amount
  FROM tx 
  WHERE out_sum > 1000000000000
  ORDER BY out_sum DESC 
  LIMIT 10;
"
```

### Blockfrost
**API-based blockchain analysis**

```javascript
const BlockFrost = require('@blockfrost/blockfrost-js');

const API = new BlockFrost.BlockFrostAPI({
  projectId: 'your_project_id'
});

// Analyze address activity
const addressInfo = await API.addresses('addr1...');
const transactions = await API.addressesTransactions('addr1...');

// Token analytics
const asset = await API.assets('policy_id.asset_name');
const assetHistory = await API.assetsHistory('policy_id.asset_name');

// Pool analysis
const pools = await API.pools();
const poolInfo = await API.poolsById('pool_id');
```

### Koios
**Community analytics API**

```bash
# Pool performance analysis
curl -X POST "https://api.koios.rest/api/v1/pool_info" \
  -H "Content-Type: application/json" \
  -d '["pool1..."]'

# Address analysis
curl -X POST "https://api.koios.rest/api/v1/address_info" \
  -H "Content-Type: application/json" \
  -d '["addr1..."]'

# Asset analysis
curl -X POST "https://api.koios.rest/api/v1/asset_info" \
  -H "Content-Type: application/json" \
  -d '["policy_id.asset_name"]'
```

## Utilities

### Cardano Address
**Address utilities and validation**

```bash
# Install
npm install cardano-addresses

# Generate addresses
echo "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about" \
  | cardano-address recovery-phrase generate \
  | cardano-address key from-recovery-phrase Shelley \
  | cardano-address key child 1852H/1815H/0H/0/0 \
  | cardano-address address payment --network-tag mainnet
```

### CBOR Tools
**CBOR encoding/decoding utilities**

```bash
# Decode transaction CBOR
echo "84a400818258..." | xxd -r -p | cbor2 --decode

# Encode JSON to CBOR
echo '{"key": "value"}' | cbor2 --encode | xxd -p
```

```javascript
import cbor from 'cbor';

// Decode CBOR hex
const cborHex = "84a400818258...";
const cborBytes = Buffer.from(cborHex, 'hex');
const decoded = cbor.decode(cborBytes);

// Encode to CBOR
const data = { key: "value" };
const encoded = cbor.encode(data);
```

### Mnemonic Tools
**BIP39 mnemonic utilities**

```javascript
import * as bip39 from 'bip39';

// Generate mnemonic
const mnemonic = bip39.generateMnemonic(256); // 24 words

// Validate mnemonic
const isValid = bip39.validateMnemonic(mnemonic);

// Generate seed
const seed = bip39.mnemonicToSeedSync(mnemonic);
```

### Cardano Serialization Lib
**Core serialization library**

```javascript
import * as CardanoWasm from "@emurgo/cardano-serialization-lib-nodejs";

// Create address
const paymentKeyHash = CardanoWasm.Ed25519KeyHash.from_bytes(
  Buffer.from("...", "hex")
);
const stakeKeyHash = CardanoWasm.Ed25519KeyHash.from_bytes(
  Buffer.from("...", "hex")
);

const address = CardanoWasm.BaseAddress.new(
  CardanoWasm.NetworkInfo.mainnet().network_id(),
  CardanoWasm.StakeCredential.from_keyhash(paymentKeyHash),
  CardanoWasm.StakeCredential.from_keyhash(stakeKeyHash)
);

// Build transaction
const txBuilder = CardanoWasm.TransactionBuilder.new(
  CardanoWasm.TransactionBuilderConfigBuilder.new()
    .fee_algo(linearFee)
    .pool_deposit(CardanoWasm.BigNum.from_str("500000000"))
    .key_deposit(CardanoWasm.BigNum.from_str("2000000"))
    .build()
);
```

---

**Next Steps**:
- Review [Development](../development/README.md) for framework-specific guides
- Check [APIs](../apis/README.md) for service integration options
- See [Testing](../testing/README.md) for comprehensive testing strategies
- Explore [Best Practices](../best-practices/README.md) for tool selection guidance