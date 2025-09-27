# Cardano Metadata Standards and CIPs

Comprehensive guide to Cardano Improvement Proposals (CIPs) and metadata standards for interoperability.

## Table of Contents

- [CIP Overview](#cip-overview)
- [Transaction Metadata](#transaction-metadata)
- [Native Token Standards](#native-token-standards)
- [NFT Standards](#nft-standards)
- [Wallet Standards](#wallet-standards)
- [DApp Standards](#dapp-standards)
- [Implementation Examples](#implementation-examples)

## CIP Overview

### What are CIPs?
Cardano Improvement Proposals (CIPs) are standards that define:
- Technical specifications
- Processes and guidelines
- Metadata formats
- Interoperability standards

### CIP Categories
- **Standards Track**: Affect implementation compatibility
- **Process**: Describe Cardano processes
- **Informational**: General guidelines and information

### CIP Repository
- **Official Repository**: https://github.com/cardano-foundation/CIPs
- **Status Tracking**: Draft, Review, Active, Final, Stagnant, Withdrawn

## Transaction Metadata

### CIP-20: Transaction Message/Comment Metadata
**Purpose**: Standard for adding human-readable messages to transactions

```json
{
  "674": {
    "msg": ["Hello, Cardano!"]
  }
}
```

**Implementation**:
```javascript
// Using Lucid
const tx = await lucid
  .newTx()
  .payToAddress("addr1...", { lovelace: 2000000n })
  .attachMetadata(674, { msg: ["Payment for services"] })
  .complete();

// Using cardano-cli
echo '{"674":{"msg":["Hello, Cardano!"]}}' > metadata.json
cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $RECIPIENT_ADDRESS+2000000 \
  --change-address $SENDER_ADDRESS \
  --metadata-json-file metadata.json \
  --testnet-magic 2 \
  --out-file tx.raw
```

### CIP-8: Message Signing
**Purpose**: Standard for signing arbitrary messages with Cardano keys

```javascript
// Message signing structure
{
  "signature": "hex_encoded_signature",
  "key": "hex_encoded_public_key",
  "address": "bech32_address"
}

// Implementation with cardano-cli
cardano-cli key sign \
  --signing-key-file payment.skey \
  --file message.txt \
  --out-file signature.json
```

## Native Token Standards

### CIP-25: NFT Metadata Standard
**Purpose**: Standard metadata format for NFTs on Cardano

```json
{
  "721": {
    "policy_id": {
      "asset_name": {
        "name": "My Awesome NFT",
        "description": "This is my awesome NFT description",
        "image": "ipfs://QmHash...",
        "mediaType": "image/png",
        "attributes": {
          "trait_type": "Background",
          "value": "Blue"
        },
        "files": [{
          "name": "My Awesome NFT",
          "mediaType": "image/png",
          "src": "ipfs://QmHash..."
        }]
      }
    }
  }
}
```

**Implementation**:
```javascript
// Minting NFT with CIP-25 metadata
const policyId = "abc123...";
const assetName = "MyNFT001";

const metadata = {
  721: {
    [policyId]: {
      [assetName]: {
        name: "Awesome Dragon #001",
        description: "A rare digital dragon",
        image: "ipfs://QmYourImageHash",
        mediaType: "image/png",
        attributes: {
          "Rarity": "Legendary",
          "Element": "Fire",
          "Power": "9000"
        },
        files: [{
          name: "Awesome Dragon #001",
          mediaType: "image/png",
          src: "ipfs://QmYourImageHash"
        }]
      }
    }
  }
};

const tx = await lucid
  .newTx()
  .mintAssets({ [policyId + assetName]: 1n })
  .attachMintingPolicy(mintingPolicy)
  .attachMetadata(721, metadata[721])
  .complete();
```

### CIP-26: Cardano Off-Chain Metadata
**Purpose**: Standard for off-chain metadata storage and verification

```json
{
  "subject": "policy_id + asset_name (hex)",
  "policy": "policy_id",
  "name": {
    "value": "Token Name",
    "sequenceNumber": 0
  },
  "description": {
    "value": "Token description",
    "sequenceNumber": 0
  },
  "ticker": {
    "value": "TICK",
    "sequenceNumber": 0
  },
  "url": {
    "value": "https://token-website.com",
    "sequenceNumber": 0
  },
  "logo": {
    "value": "base64_encoded_logo",
    "sequenceNumber": 0
  },
  "decimals": {
    "value": 6,
    "sequenceNumber": 0
  }
}
```

### CIP-68: Datum Metadata Standard
**Purpose**: On-chain metadata using datums for enhanced functionality

```rust
// Aiken implementation
type CIP68Datum {
  metadata: Map<ByteArray, Data>,
  version: Int,
  extra: Data
}

// Standard reference token (100) and user token (222) prefixes
let reference_prefix = #"000643b0" // (100)
let user_token_prefix = #"000de140" // (222)
```

```javascript
// JavaScript implementation
const cip68Metadata = {
  name: "My Token",
  description: "Token description",
  image: "ipfs://...",
  // Custom metadata fields
  attributes: {
    color: "blue",
    rarity: "common"
  }
};

// Create datum with metadata
const datum = Data.to(new Constr(0, [
  new Map([
    ["name", cip68Metadata.name],
    ["description", cip68Metadata.description],
    ["image", cip68Metadata.image]
  ]),
  1, // version
  Data.void() // extra
]));
```

## NFT Standards

### CIP-25: NFT Metadata Standard (Extended)
**Media Types**:
- `image/png`, `image/jpeg`, `image/gif`
- `video/mp4`, `video/webm`
- `audio/mpeg`, `audio/wav`
- `model/gltf+json` (3D models)
- `text/html` (HTML content)

**Attributes Structure**:
```json
{
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Blue"
    },
    {
      "trait_type": "Rarity",
      "value": "Common",
      "display_type": "string"
    },
    {
      "trait_type": "Power Level",
      "value": 85,
      "display_type": "number"
    },
    {
      "trait_type": "Speed",
      "value": 92.5,
      "display_type": "boost_percentage"
    }
  ]
}
```

### CIP-27: CNFT Community Royalties Standard
**Purpose**: Standard for NFT creator royalties

```json
{
  "777": {
    "rate": "0.025", // 2.5% royalty
    "addr": "addr1royalty_address_here"
  }
}
```

**Implementation**:
```javascript
const royaltyMetadata = {
  777: {
    rate: "0.05", // 5% royalty
    addr: "addr1qx2fxv2umyhttkxyxp8x0dlpdt3k6cwng5pxj3jhsydzer3"
  }
};

// Include in minting transaction
const tx = await lucid
  .newTx()
  .mintAssets({ [policyId + assetName]: 1n })
  .attachMintingPolicy(mintingPolicy)
  .attachMetadata(721, nftMetadata)
  .attachMetadata(777, royaltyMetadata[777])
  .complete();
```

### CIP-60: Music Token Metadata
**Purpose**: Specialized metadata for music NFTs

```json
{
  "721": {
    "policy_id": {
      "asset_name": {
        "name": "Epic Song #001",
        "artist": "Digital Musician",
        "genre": ["Electronic", "Ambient"],
        "duration": 240, // seconds
        "release_date": "2024-01-15",
        "album": "Digital Dreams",
        "track_number": 1,
        "files": [{
          "name": "Epic Song #001",
          "mediaType": "audio/mpeg",
          "src": "ipfs://QmMusicHash"
        }],
        "lyrics": "ipfs://QmLyricsHash",
        "bpm": 128,
        "key": "C major"
      }
    }
  }
}
```

## Wallet Standards

### CIP-30: Cardano dApp-Wallet Web Bridge
**Purpose**: Standard API for dApp-wallet communication

```javascript
// Check if wallet is available
if (window.cardano && window.cardano.nami) {
  // Request wallet access
  const api = await window.cardano.nami.enable();
  
  // Get network ID
  const networkId = await api.getNetworkId();
  
  // Get wallet balance
  const balance = await api.getBalance();
  
  // Get UTxOs
  const utxos = await api.getUtxos();
  
  // Sign transaction
  const signedTx = await api.signTx(txCbor, partialSign);
  
  // Submit transaction
  const txHash = await api.submitTx(signedTxCbor);
  
  // Sign data
  const signature = await api.signData(address, payload);
}

// Multi-wallet support
const supportedWallets = ['nami', 'eternl', 'flint', 'typhon', 'cardwallet'];
const availableWallets = supportedWallets.filter(wallet => 
  window.cardano && window.cardano[wallet]
);
```

### CIP-95: Web-Wallet Bridge - Governance
**Purpose**: Extension of CIP-30 for governance actions

```javascript
if (window.cardano && window.cardano.nami) {
  const api = await window.cardano.nami.enable();
  
  // Get governance-related UTxOs
  const govUtxos = await api.experimental.getGovernanceUtxos();
  
  // Get DRep key
  const drepKey = await api.experimental.getDRepKey();
  
  // Sign governance action
  const govSignature = await api.experimental.signGovernanceData(govAction);
}
```

## DApp Standards

### CIP-30: dApp Connector (Extended Usage)
**Connection Management**:
```javascript
class WalletManager {
  constructor() {
    this.connectedWallet = null;
    this.api = null;
  }
  
  async connect(walletName) {
    if (!window.cardano || !window.cardano[walletName]) {
      throw new Error(`${walletName} wallet not found`);
    }
    
    const wallet = window.cardano[walletName];
    
    // Check if wallet is enabled
    const isEnabled = await wallet.isEnabled();
    if (!isEnabled) {
      // Request permission
      this.api = await wallet.enable();
    } else {
      this.api = await wallet.enable();
    }
    
    this.connectedWallet = walletName;
    return this.api;
  }
  
  async getWalletInfo() {
    if (!this.api) throw new Error('Wallet not connected');
    
    const [networkId, balance, utxos, changeAddress, rewardAddresses] = await Promise.all([
      this.api.getNetworkId(),
      this.api.getBalance(),
      this.api.getUtxos(),
      this.api.getChangeAddress(),
      this.api.getRewardAddresses()
    ]);
    
    return {
      networkId,
      balance,
      utxoCount: utxos.length,
      changeAddress,
      rewardAddresses
    };
  }
}
```

### CIP-45: Decentralized WebRTC dApp-Wallet Communication
**Purpose**: Alternative communication method for mobile wallets

```javascript
// WebRTC-based wallet connection
class WebRTCWalletConnector {
  constructor() {
    this.connection = null;
    this.dataChannel = null;
  }
  
  async connect(walletId) {
    this.connection = new RTCPeerConnection({
      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
    });
    
    this.dataChannel = this.connection.createDataChannel('wallet-comm');
    
    this.dataChannel.onmessage = (event) => {
      const response = JSON.parse(event.data);
      this.handleWalletResponse(response);
    };
    
    // Exchange offers/answers with wallet
    const offer = await this.connection.createOffer();
    await this.connection.setLocalDescription(offer);
    
    // Send offer to wallet via QR code or deep link
    this.displayConnectionOffer(offer);
  }
  
  async sendCommand(command, params) {
    const message = {
      id: crypto.randomUUID(),
      command,
      params,
      timestamp: Date.now()
    };
    
    this.dataChannel.send(JSON.stringify(message));
  }
}
```

## Implementation Examples

### Complete NFT Minting with Multiple Standards
```javascript
async function mintCompliantNFT(lucid, mintingData) {
  const {
    policyId,
    assetName,
    name,
    description,
    image,
    attributes,
    royaltyRate,
    royaltyAddress,
    creatorAddress
  } = mintingData;
  
  // CIP-25 NFT Metadata
  const nftMetadata = {
    721: {
      [policyId]: {
        [assetName]: {
          name,
          description,
          image,
          mediaType: "image/png",
          attributes,
          files: [{
            name,
            mediaType: "image/png",
            src: image
          }]
        }
      }
    }
  };
  
  // CIP-27 Royalty Metadata
  const royaltyMetadata = {
    777: {
      rate: royaltyRate.toString(),
      addr: royaltyAddress
    }
  };
  
  // Build transaction
  let tx = lucid
    .newTx()
    .mintAssets({ [policyId + assetName]: 1n })
    .attachMintingPolicy(mintingPolicy)
    .attachMetadata(721, nftMetadata[721])
    .attachMetadata(777, royaltyMetadata[777]);
  
  // Add creator attribution (custom metadata)
  if (creatorAddress) {
    tx = tx.attachMetadata(1337, {
      creator: creatorAddress,
      minted_at: new Date().toISOString()
    });
  }
  
  const completeTx = await tx.complete();
  const signedTx = await completeTx.sign().complete();
  return await signedTx.submit();
}

// Usage
const txHash = await mintCompliantNFT(lucid, {
  policyId: "abc123...",
  assetName: "Dragon001",
  name: "Fire Dragon #001",
  description: "A legendary fire dragon NFT",
  image: "ipfs://QmDragonImage",
  attributes: {
    "Element": "Fire",
    "Rarity": "Legendary",
    "Power": "9000"
  },
  royaltyRate: 0.05, // 5%
  royaltyAddress: "addr1...",
  creatorAddress: "addr1..."
});
```

### Multi-Standard Token Registry
```javascript
class TokenRegistry {
  constructor() {
    this.tokens = new Map();
  }
  
  // Register token with multiple metadata standards
  async registerToken(tokenInfo) {
    const {
      policyId,
      assetName,
      metadata,
      standards = ['CIP-25', 'CIP-26', 'CIP-68']
    } = tokenInfo;
    
    const tokenKey = policyId + assetName;
    const registryEntry = {
      policyId,
      assetName,
      registeredAt: new Date().toISOString(),
      standards: {},
      metadata: {}
    };
    
    // Process each standard
    for (const standard of standards) {
      switch (standard) {
        case 'CIP-25':
          registryEntry.standards['CIP-25'] = this.formatCIP25(metadata);
          break;
        case 'CIP-26':
          registryEntry.standards['CIP-26'] = await this.formatCIP26(metadata);
          break;
        case 'CIP-68':
          registryEntry.standards['CIP-68'] = this.formatCIP68(metadata);
          break;
      }
    }
    
    this.tokens.set(tokenKey, registryEntry);
    return registryEntry;
  }
  
  formatCIP25(metadata) {
    return {
      721: {
        [metadata.policyId]: {
          [metadata.assetName]: {
            name: metadata.name,
            description: metadata.description,
            image: metadata.image,
            mediaType: metadata.mediaType || "image/png",
            attributes: metadata.attributes || {}
          }
        }
      }
    };
  }
  
  async formatCIP26(metadata) {
    const subject = metadata.policyId + Buffer.from(metadata.assetName).toString('hex');
    
    return {
      subject,
      policy: metadata.policyId,
      name: {
        value: metadata.name,
        sequenceNumber: 0
      },
      description: {
        value: metadata.description,
        sequenceNumber: 0
      },
      ticker: {
        value: metadata.ticker || metadata.name.substring(0, 5).toUpperCase(),
        sequenceNumber: 0
      },
      decimals: {
        value: metadata.decimals || 0,
        sequenceNumber: 0
      }
    };
  }
  
  formatCIP68(metadata) {
    return {
      metadata: new Map([
        ["name", metadata.name],
        ["description", metadata.description],
        ["image", metadata.image],
        ["attributes", metadata.attributes || {}]
      ]),
      version: 1,
      extra: null
    };
  }
  
  // Validate token against standards
  validateToken(tokenKey, standards = []) {
    const token = this.tokens.get(tokenKey);
    if (!token) return { valid: false, errors: ["Token not found"] };
    
    const errors = [];
    
    for (const standard of standards) {
      if (!token.standards[standard]) {
        errors.push(`Missing ${standard} compliance`);
        continue;
      }
      
      // Validate each standard
      const validation = this.validateStandard(standard, token.standards[standard]);
      if (!validation.valid) {
        errors.push(...validation.errors);
      }
    }
    
    return {
      valid: errors.length === 0,
      errors,
      token
    };
  }
  
  validateStandard(standard, data) {
    switch (standard) {
      case 'CIP-25':
        return this.validateCIP25(data);
      case 'CIP-26':
        return this.validateCIP26(data);
      case 'CIP-68':
        return this.validateCIP68(data);
      default:
        return { valid: false, errors: [`Unknown standard: ${standard}`] };
    }
  }
  
  validateCIP25(data) {
    const errors = [];
    
    if (!data[721]) errors.push("Missing 721 metadata label");
    
    // Additional CIP-25 validations...
    
    return { valid: errors.length === 0, errors };
  }
}

// Usage
const registry = new TokenRegistry();

await registry.registerToken({
  policyId: "abc123...",
  assetName: "MyToken",
  metadata: {
    name: "My Awesome Token",
    description: "An awesome token",
    image: "ipfs://QmHash",
    attributes: { rarity: "common" }
  },
  standards: ['CIP-25', 'CIP-26']
});

const validation = registry.validateToken("abc123...MyToken", ['CIP-25']);
console.log(validation); // { valid: true, errors: [], token: {...} }
```

### Governance Metadata (CIP-1694 Compatible)
```json
{
  "1694": {
    "governance_action_type": "parameter_change",
    "proposal_id": "gov_action_123",
    "title": "Increase Block Size Limit",
    "abstract": "Proposal to increase the maximum block size",
    "motivation": "Network throughput improvement needed",
    "rationale": "Analysis shows 20% increase is safe",
    "references": [
      {
        "type": "research_paper", 
        "url": "https://research.example.com/block-size-analysis"
      }
    ],
    "voting_procedures": {
      "constitutional_committee": "yes",
      "dreps": "yes", 
      "spo": "no"
    }
  }
}
```

---

**Next Steps**:
- Check [Smart Contracts](../smart-contracts/README.md) for implementing metadata standards
- Review [Transaction Building](../transaction-building/README.md) for metadata attachment
- See [Wallet Integration](../wallet-integration/README.md) for CIP-30 implementation
- Explore [Resources](../resources/README.md) for official CIP documentation