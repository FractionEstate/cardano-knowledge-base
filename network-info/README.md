# Cardano Network Information

Essential information about Cardano networks for development and deployment.

## Table of Contents

- [Network Overview](#network-overview)
- [Mainnet](#mainnet)
- [Preview Testnet](#preview-testnet)
- [Pre-production Testnet](#pre-production-testnet)
- [SanchoNet](#sanchonet)
- [Local Testnets](#local-testnets)
- [Network Parameters](#network-parameters)
- [Faucets and Resources](#faucets-and-resources)

## Network Overview

### Network Types
- **Mainnet**: Production network with real ADA
- **Preview**: Latest features testing network
- **Pre-production**: Production-like testing environment
- **SanchoNet**: Governance features testing (Conway era)
- **Local**: Developer-controlled private networks

### Network Selection Guidelines
- **Development**: Preview testnet for latest features
- **Testing**: Pre-production for production-like testing
- **Governance**: SanchoNet for governance feature testing
- **Production**: Mainnet for live applications

## Mainnet

### Network Configuration
```json
{
  "NetworkId": 1,
  "NetworkMagic": 764824073,
  "ProtocolMagic": 764824073
}
```

### Key Information
- **Network ID**: `1`
- **Magic**: `764824073`
- **Byron Genesis Hash**: `5f20df933584822601f9e3f8c024eb5eb252fe8cefb24d1317dc3d432e940ebb`
- **Shelley Genesis Hash**: `1a3be38bcbb7911969283716ad7aa550250226b76a61fc51cc9a9a35d9276d81`

### Configuration URLs
```bash
# Byron Genesis
curl -O https://book.world.dev.cardano.org/environments/mainnet/byron-genesis.json

# Shelley Genesis
curl -O https://book.world.dev.cardano.org/environments/mainnet/shelley-genesis.json

# Alonzo Genesis
curl -O https://book.world.dev.cardano.org/environments/mainnet/alonzo-genesis.json

# Conway Genesis
curl -O https://book.world.dev.cardano.org/environments/mainnet/conway-genesis.json

# Node Configuration
curl -O https://book.world.dev.cardano.org/environments/mainnet/config.json

# Topology
curl -O https://book.world.dev.cardano.org/environments/mainnet/topology.json
```

### Explorers
- **CardanoScan**: https://cardanoscan.io
- **Cardano Explorer**: https://explorer.cardano.org
- **ADAStat**: https://adastat.net
- **Pool Tool**: https://pooltool.io

### RPC Endpoints
```javascript
// Blockfrost
const API = new BlockFrost.BlockFrostAPI({
  projectId: 'mainnet_your_project_id',
  network: 'mainnet'
});

// Koios
const KOIOS_MAINNET = 'https://api.koios.rest/api/v1';

// Maestro
const maestro = new MaestroClient({
  network: 'Mainnet',
  apiKey: 'your-api-key'
});
```

## Preview Testnet

### Network Configuration
```json
{
  "NetworkId": 0,
  "NetworkMagic": 2,
  "ProtocolMagic": 2
}
```

### Key Information
- **Network ID**: `0`
- **Magic**: `2`
- **Purpose**: Latest feature testing and development
- **Reset Schedule**: Irregular, when major updates needed

### Configuration URLs
```bash
# All configuration files
curl -O https://book.world.dev.cardano.org/environments/preview/byron-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preview/shelley-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preview/alonzo-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preview/conway-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preview/config.json
curl -O https://book.world.dev.cardano.org/environments/preview/topology.json
```

### CLI Usage
```bash
# All CLI commands use --testnet-magic 2
cardano-cli query tip --testnet-magic 2

cardano-cli query utxo \
  --address $(cat payment.addr) \
  --testnet-magic 2

cardano-cli transaction submit \
  --tx-file tx.signed \
  --testnet-magic 2
```

### SDK Configuration
```javascript
// Lucid
const lucid = await Lucid.new(
  new Blockfrost("https://cardano-preview.blockfrost.io/api/v0", "preview_project_id"),
  "Preview"
);

// PyCardano
context = BlockFrostChainContext(
    project_id="preview_project_id",
    base_url="https://cardano-preview.blockfrost.io/api/v0"
)

// Mesh
const blockchainProvider = new BlockfrostProvider('preview_project_id');
```

### Explorers
- **CardanoScan Preview**: https://preview.cardanoscan.io
- **Cardano Explorer Preview**: https://explorer.cardano-preview.testnets.cardano.org

## Pre-production Testnet

### Network Configuration
```json
{
  "NetworkId": 0,
  "NetworkMagic": 1,
  "ProtocolMagic": 1
}
```

### Key Information
- **Network ID**: `0`
- **Magic**: `1`
- **Purpose**: Production-like testing environment
- **Stability**: More stable than Preview, closer to Mainnet

### Configuration URLs
```bash
# All configuration files
curl -O https://book.world.dev.cardano.org/environments/preprod/byron-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preprod/shelley-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preprod/alonzo-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preprod/conway-genesis.json
curl -O https://book.world.dev.cardano.org/environments/preprod/config.json
curl -O https://book.world.dev.cardano.org/environments/preprod/topology.json
```

### CLI Usage
```bash
# All CLI commands use --testnet-magic 1
cardano-cli query tip --testnet-magic 1

cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $RECIPIENT_ADDRESS+1000000 \
  --change-address $SENDER_ADDRESS \
  --testnet-magic 1 \
  --out-file payment.raw
```

### SDK Configuration
```javascript
// Lucid
const lucid = await Lucid.new(
  new Blockfrost("https://cardano-preprod.blockfrost.io/api/v0", "preprod_project_id"),
  "Preprod"
);

// PyCardano
context = BlockFrostChainContext(
    project_id="preprod_project_id",
    base_url="https://cardano-preprod.blockfrost.io/api/v0"
)
```

### Explorers
- **CardanoScan Preprod**: https://preprod.cardanoscan.io
- **Cardano Explorer Preprod**: https://explorer.cardano-preprod.testnets.cardano.org

## SanchoNet

### Network Configuration
```json
{
  "NetworkId": 0,
  "NetworkMagic": 4,
  "ProtocolMagic": 4
}
```

### Key Information
- **Network ID**: `0`
- **Magic**: `4`
- **Purpose**: Governance (Conway era) feature testing
- **Special Features**: Voting, governance actions, constitutional committee

### Configuration URLs
```bash
# SanchoNet configurations
curl -O https://book.world.dev.cardano.org/environments/sanchonet/byron-genesis.json
curl -O https://book.world.dev.cardano.org/environments/sanchonet/shelley-genesis.json
curl -O https://book.world.dev.cardano.org/environments/sanchonet/alonzo-genesis.json  
curl -O https://book.world.dev.cardano.org/environments/sanchonet/conway-genesis.json
curl -O https://book.world.dev.cardano.org/environments/sanchonet/config.json
curl -O https://book.world.dev.cardano.org/environments/sanchonet/topology.json
```

### Governance Features
```bash
# Vote on governance actions
cardano-cli governance vote create \
  --yes \
  --governance-action-tx-id $GOV_ACTION_TX_ID \
  --governance-action-index 0 \
  --drep-verification-key-file drep.vkey \
  --out-file vote.json

# Submit governance action
cardano-cli governance action create-constitution \
  --testnet-magic 4 \
  --governance-action-deposit 50000000000 \
  --deposit-return-stake-verification-key-file stake.vkey \
  --constitution-url "https://example.com/constitution" \
  --constitution-hash "abc123..." \
  --out-file constitution-action.json
```

### Explorers
- **SanchoNet Explorer**: https://sancho.network
- **CardanoScan SanchoNet**: https://sanchonet.cardanoscan.io

## Local Testnets

### Cardano Testnet (Official Tool)
```bash
# Create local testnet
cardano-testnet cardano \
  --testnet-magic 42 \
  --num-pool-nodes 3 \
  --slot-length 0.1 \
  --security-param 10

# Directory structure created:
# testnet/
# ├── node-pool1/
# ├── node-pool2/  
# ├── node-pool3/
# └── configuration.yaml
```

### Manual Local Setup
```bash
# Create genesis files
cardano-cli genesis create-cardano \
  --genesis-dir genesis \
  --testnet-magic 42 \
  --supply 30000000000000000 \
  --gen-genesis-keys 3 \
  --gen-utxo-keys 3

# Start nodes
cardano-node run \
  --config genesis/configuration.yaml \
  --topology genesis/topology.json \
  --database-path node1/db \
  --socket-path node1/socket \
  --port 3001 \
  --shelley-kes-key genesis/node-keys/node1.kes.skey \
  --shelley-vrf-key genesis/node-keys/node1.vrf.skey \
  --shelley-operational-certificate genesis/node-keys/node1.opcert
```

### Docker Local Testnet
```yaml
# docker-compose.yml
version: '3.8'
services:
  cardano-node:
    image: inputoutput/cardano-node:latest
    volumes:
      - ./config:/opt/cardano/config
      - ./data:/opt/cardano/data
    ports:
      - "3001:3001"
    environment:
      - NETWORK=testnet
      - TESTNET_MAGIC=42
    command: >
      cardano-node run
      --config /opt/cardano/config/config.json
      --topology /opt/cardano/config/topology.json
      --database-path /opt/cardano/data/db
      --socket-path /opt/cardano/data/socket
      --port 3001
```

## Network Parameters

### Key Parameters Comparison

| Parameter | Mainnet | Preview | Preprod | SanchoNet |
|-----------|---------|---------|---------|-----------|
| Network ID | 1 | 0 | 0 | 0 |
| Magic | 764824073 | 2 | 1 | 4 |
| Slot Length | 1s | 1s | 1s | 1s |
| Epoch Length | 432,000 slots | 86,400 slots | 432,000 slots | 86,400 slots |
| Max Block Size | 90KB | 90KB | 90KB | 90KB |
| Max Tx Size | 16KB | 16KB | 16KB | 16KB |

### Protocol Parameters
```bash
# Query current protocol parameters
cardano-cli query protocol-parameters \
  --testnet-magic 2 \
  --out-file protocol.json

# Key parameters to monitor:
# - minFeeA: Fee per byte
# - minFeeB: Base fee
# - poolDeposit: Stake pool deposit
# - keyDeposit: Stake key deposit
# - maxTxSize: Maximum transaction size
# - maxBlockHeaderSize: Maximum block header size
```

### Genesis Parameters
```json
{
  "activeSlotsCoeff": 0.05,
  "protocolParams": {
    "protocolVersion": {
      "major": 8,
      "minor": 0
    },
    "decentralisationParam": 0,
    "eMax": 18,
    "extraEntropy": {
      "tag": "NeutralNonce"
    },
    "maxBlockBodySize": 90112,
    "maxBlockHeaderSize": 1100,
    "maxTxSize": 16384,
    "minFeeA": 44,
    "minFeeB": 155381,
    "minUTxOValue": 1000000,
    "poolDeposit": 500000000,
    "keyDeposit": 2000000,
    "rho": 0.003,
    "tau": 0.20,
    "a0": 0.3,
    "nOpt": 150,
    "costModels": {...},
    "maxCollateralInputs": 3
  }
}
```

## Faucets and Resources

### Preview Testnet Faucet
```bash
# Web Interface
# https://docs.cardano.org/cardano-testnet/tools/faucet

# API Usage
curl -X POST https://faucet.preview.world.dev.cardano.org/send-money \
  -H "Content-Type: application/json" \
  -d '{
    "address": "addr_test1...",
    "amount": 1000000000
  }'
```

### Pre-production Testnet Faucet
```bash
# Web Interface  
# https://docs.cardano.org/cardano-testnet/tools/faucet

# API Usage
curl -X POST https://faucet.preprod.world.dev.cardano.org/send-money \
  -H "Content-Type: application/json" \
  -d '{
    "address": "addr_test1...",
    "amount": 1000000000
  }'
```

### SanchoNet Faucet
```bash
# SanchoNet specific faucet
curl -X POST https://faucet.sanchonet.world.dev.cardano.org/send-money \
  -H "Content-Type: application/json" \
  -d '{
    "address": "addr_test1...",
    "amount": 1000000000
  }'
```

### Test Token Generation
```bash
# Create test native tokens on testnet
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --mint "1000000 $POLICY_ID.TestToken" \
  --mint-script-file policy.script \
  --tx-out "$RECIPIENT_ADDRESS+2000000+1000000 $POLICY_ID.TestToken" \
  --change-address $SENDER_ADDRESS \
  --testnet-magic 2 \
  --out-file mint.raw
```

### Network Status APIs
```javascript
// Check network health
const tip = await API.epoch();
const health = await API.health();

// Network statistics
const stats = await API.metrics();
const pools = await API.pools();

// Current epoch information
const currentEpoch = await API.epochsLatest();
const epochParams = await API.epochsParameters(currentEpoch.epoch);
```

### Useful Network Commands
```bash
# Check node sync status
cardano-cli query tip --testnet-magic 2

# Check network parameters
cardano-cli query protocol-parameters --testnet-magic 2

# Check stake distribution
cardano-cli query stake-distribution --testnet-magic 2

# Check pool information
cardano-cli query pool-params --stake-pool-id $POOL_ID --testnet-magic 2

# Check leadership schedule
cardano-cli query leadership-schedule \
  --genesis genesis.json \
  --stake-pool-id $POOL_ID \
  --vrf-signing-key-file vrf.skey \
  --testnet-magic 2
```

---

**Next Steps**:
- Review [Development](../development/README.md) for network-specific development setup
- Check [APIs](../apis/README.md) for network-specific API endpoints
- See [Tools](../tools/README.md) for network configuration utilities
- Explore [Testing](../testing/README.md) for network-specific testing strategies