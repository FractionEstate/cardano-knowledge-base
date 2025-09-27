# Cardano Development Troubleshooting

Common issues, solutions, and debugging strategies for Cardano development.

## Table of Contents

- [Node and CLI Issues](#node-and-cli-issues)
- [Transaction Problems](#transaction-problems)
- [Smart Contract Issues](#smart-contract-issues)
- [Wallet Integration Problems](#wallet-integration-problems)
- [API and Network Issues](#api-and-network-issues)
- [Build and Compilation Errors](#build-and-compilation-errors)
- [Debugging Strategies](#debugging-strategies)

## Node and CLI Issues

### Node Sync Problems

#### Issue: Node not syncing
```bash
# Check node status
cardano-cli query tip --testnet-magic 2

# If stuck, check logs
tail -f $CARDANO_NODE_LOGS/node.log

# Common solutions:
# 1. Restart the node
pkill cardano-node
cardano-node run --config config.json --topology topology.json ...

# 2. Check disk space
df -h

# 3. Update topology file
wget -O topology.json https://book.world.dev.cardano.org/environments/preview/topology.json
```

#### Issue: Node crashes with "MuxError"
```bash
# Solution: Update topology.json with working relays
# Download latest topology
curl -o topology.json https://book.world.dev.cardano.org/environments/preview/topology.json

# Or create minimal topology
cat > topology.json << EOF
{
  "Producers": [
    {
      "addr": "preview-node.world.dev.cardano.org",
      "port": 30002,
      "valency": 1
    }
  ]
}
EOF
```

#### Issue: Socket connection errors
```bash
# Error: cardano-cli: Network.Socket.connect: does not exist
export CARDANO_NODE_SOCKET_PATH="path/to/node.socket"

# Verify socket exists
ls -la $CARDANO_NODE_SOCKET_PATH

# Check node is running
ps aux | grep cardano-node
```

### CLI Command Issues

#### Issue: "Command failed" with no specific error
```bash
# Add --testnet-magic parameter for testnet commands
cardano-cli query tip --testnet-magic 2

# For mainnet, use --mainnet
cardano-cli query tip --mainnet

# Check cardano-cli version compatibility
cardano-cli --version
cardano-node --version
```

#### Issue: UTxO not found
```bash
# Wait for transaction confirmation
cardano-cli query tip --testnet-magic 2

# Check if UTxO exists at address
cardano-cli query utxo --address $(cat payment.addr) --testnet-magic 2

# If UTxO missing, check transaction was submitted
cardano-cli query tx-mempool info --testnet-magic 2
```

## Transaction Problems

### Transaction Building Errors

#### Issue: InsufficientFunds error
```javascript
// Check available UTxOs
const utxos = await lucid.wallet.getUtxos();
const totalBalance = utxos.reduce((sum, utxo) => 
  sum + BigInt(utxo.assets.lovelace), 0n
);

console.log(`Available balance: ${totalBalance} lovelace`);
console.log(`Required: ${paymentAmount + estimatedFee} lovelace`);

// Solution: Ensure sufficient funds including fees
const estimatedFee = 200000n; // Conservative estimate
const requiredAmount = paymentAmount + estimatedFee;

if (totalBalance < requiredAmount) {
  throw new Error(`Insufficient funds: ${totalBalance} < ${requiredAmount}`);
}
```

#### Issue: ValueNotConserved error
```bash
# In cardano-cli, ensure input value equals output value plus fee
# Calculate total input value
TOTAL_INPUT=$(cardano-cli query utxo --address $SENDER_ADDRESS --testnet-magic 2 | awk 'NR>2 {sum += $3} END {print sum}')

# Ensure: TOTAL_INPUT = PAYMENT_AMOUNT + CHANGE_AMOUNT + FEE
CHANGE_AMOUNT=$((TOTAL_INPUT - PAYMENT_AMOUNT - FEE))

cardano-cli transaction build \
  --tx-in $UTXO_IN \
  --tx-out $RECIPIENT_ADDRESS+$PAYMENT_AMOUNT \
  --tx-out $SENDER_ADDRESS+$CHANGE_AMOUNT \
  --fee $FEE \
  --testnet-magic 2 \
  --out-file tx.raw
```

#### Issue: Transaction too large
```javascript
// Reduce transaction size by:
// 1. Using fewer UTxOs
const selectedUtxos = utxos.slice(0, 5); // Limit UTxO count

// 2. Reducing metadata size
const metadata = {
  674: { msg: ["Short message"] } // Keep metadata minimal
};

// 3. Using reference scripts instead of inline scripts
const tx = await lucid
  .newTx()
  .collectFrom(utxos, redeemer)
  .readFrom([referenceScriptUtxo]) // Use reference instead of attach
  .complete();
```

### Transaction Submission Issues

#### Issue: Transaction rejected by network
```javascript
try {
  const txHash = await signedTx.submit();
} catch (error) {
  console.error('Submission error:', error.message);
  
  // Common causes and solutions:
  if (error.message.includes('BadInputs')) {
    console.log('UTxO already spent or doesn\'t exist');
    // Refresh UTxOs and rebuild transaction
  }
  
  if (error.message.includes('FeeTooSmall')) {
    console.log('Increase transaction fee');
    // Rebuild with higher fee
  }
  
  if (error.message.includes('ValueNotConserved')) {
    console.log('Input/output value mismatch');
    // Check calculation logic
  }
}
```

#### Issue: Transaction stuck in mempool
```bash
# Check if transaction is in mempool
cardano-cli query tx-mempool info --testnet-magic 2

# If stuck for >10 minutes, may need to rebuild with higher fee
# or wait for network congestion to clear

# Check network status
curl -s https://cardano-preview.blockfrost.io/api/v0/health
```

## Smart Contract Issues

### Plutus Validation Errors

#### Issue: Script validation failure
```haskell
-- Add tracing to debug validation logic
{-# INLINABLE myValidator #-}
myValidator :: MyDatum -> MyRedeemer -> ScriptContext -> Bool
myValidator dat red ctx = 
  traceIfFalse "Signature check failed" checkSignature &&
  traceIfFalse "Time constraint failed" checkTime &&
  traceIfFalse "Value preservation failed" checkValue
  where
    info = scriptContextTxInfo ctx
    
    checkSignature = case red of
      Unlock -> 
        traceIfFalse "Owner signature missing" $ 
        txSignedBy info (owner dat)
      _ -> True
    
    checkTime = case red of
      Unlock -> 
        traceIfFalse "Deadline passed" $
        deadline dat `after` txValidRange info
      _ -> True
```

#### Issue: Aiken compilation errors
```bash
# Common Aiken errors and solutions:

# Error: "function not found"
# Solution: Check imports and function names
use aiken/list
use aiken/transaction.{ScriptContext}

# Error: "type mismatch"
# Solution: Ensure types match exactly
fn check_value(value: Int) -> Bool {
  value > 0  // Returns Bool, not Int
}

# Error: "pattern match not exhaustive"
# Solution: Handle all cases
when my_option is {
  Some(value) -> value > 0
  None -> False  // Don't forget None case
}
```

#### Issue: Script execution budget exceeded
```rust
// Optimize Aiken code to reduce execution units:

// Instead of nested loops:
fn inefficient_search(items: List<Int>, target: Int) -> Bool {
  list.any(items, fn(item) {
    list.any(items, fn(other) { item + other == target })
  })
}

// Use more efficient algorithms:
fn efficient_search(items: List<Int>, target: Int) -> Bool {
  when items is {
    [] -> False
    [head, ..tail] -> 
      if head == target {
        True
      } else {
        efficient_search(tail, target)
      }
  }
}
```

### Contract Deployment Issues

#### Issue: Script address calculation mismatch
```javascript
// Ensure consistent script address calculation
import { SpendingValidator } from "lucid-cardano";

// Load compiled script correctly
const validator = {
  type: "PlutusV2" as const,
  script: compiledScriptCbor // Ensure this matches actual compiled output
};

const scriptAddress = lucid.utils.validatorToAddress(validator);
console.log(`Script address: ${scriptAddress}`);

// Verify with cardano-cli
// cardano-cli address build --payment-script-file validator.plutus --testnet-magic 2
```

#### Issue: Wrong datum hash/inline datum
```javascript
// For datum hash (legacy)
const datumHash = lucid.utils.datumToHash(datum);
const tx = await lucid
  .newTx()
  .payToContract(scriptAddress, { asHash: datum }, assets)
  .complete();

// For inline datum (recommended)
const tx = await lucid
  .newTx()
  .payToContract(scriptAddress, { inline: datum }, assets)
  .complete();
```

## Wallet Integration Problems

### CIP-30 Connection Issues

#### Issue: Wallet not detected
```javascript
// Debug wallet detection
function debugWalletDetection() {
  console.log('Window.cardano:', window.cardano);
  
  if (window.cardano) {
    Object.entries(window.cardano).forEach(([key, wallet]) => {
      console.log(`Wallet ${key}:`, {
        hasEnable: typeof wallet.enable === 'function',
        name: wallet.name,
        icon: wallet.icon,
        apiVersion: wallet.apiVersion
      });
    });
  } else {
    console.log('No cardano object found - no wallets installed');
  }
}

debugWalletDetection();
```

#### Issue: Wallet connection fails
```javascript
async function debugWalletConnection(walletName) {
  try {
    const wallet = window.cardano?.[walletName];
    if (!wallet) {
      throw new Error(`${walletName} not found`);
    }
    
    console.log(`Attempting to connect to ${walletName}...`);
    const api = await wallet.enable();
    console.log(`Connected! API version: ${api.apiVersion}`);
    
    // Test basic functions
    const networkId = await api.getNetworkId();
    console.log(`Network ID: ${networkId}`);
    
    return api;
  } catch (error) {
    console.error(`Connection failed:`, error);
    
    // Specific error handling
    if (error.code === -1) {
      console.log('User rejected connection');
    } else if (error.code === -2) {
      console.log('Wallet already connected elsewhere');
    }
    
    throw error;
  }
}
```

#### Issue: Wallet API methods failing
```javascript
// Test all wallet API methods systematically
async function testWalletAPI(api) {
  const tests = [
    { name: 'getNetworkId', fn: () => api.getNetworkId() },
    { name: 'getBalance', fn: () => api.getBalance() },
    { name: 'getUtxos', fn: () => api.getUtxos() },
    { name: 'getChangeAddress', fn: () => api.getChangeAddress() },
    { name: 'getRewardAddresses', fn: () => api.getRewardAddresses() }
  ];
  
  for (const test of tests) {
    try {
      const result = await test.fn();
      console.log(`✓ ${test.name}:`, result);
    } catch (error) {
      console.error(`✗ ${test.name}:`, error.message);
    }
  }
}
```

### Transaction Signing Issues

#### Issue: Transaction signing rejected
```javascript
// Add better user communication
async function signTransactionWithRetry(api, tx, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      console.log(`Signing attempt ${attempt}/${maxRetries}...`);
      
      // Show transaction details to user
      const outputs = tx.body.outputs.map(output => ({
        address: output.address,
        amount: `${BigInt(output.amount.coin) / 1000000n} ADA`
      }));
      
      console.log('Transaction outputs:', outputs);
      console.log('Fee:', `${BigInt(tx.body.fee) / 1000000n} ADA`);
      
      const witnessSet = await api.signTx(tx.toCBOR(), true);
      return witnessSet;
    } catch (error) {
      console.error(`Signing attempt ${attempt} failed:`, error.message);
      
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Wait before retry
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

## API and Network Issues

### Blockfrost API Issues

#### Issue: Rate limiting
```javascript
// Implement exponential backoff
class BlockfrostRetryClient {
  constructor(api) {
    this.api = api;
    this.maxRetries = 3;
    this.baseDelay = 1000;
  }
  
  async requestWithRetry(operation) {
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        if (error.status_code === 429 && attempt < this.maxRetries) {
          const delay = this.baseDelay * Math.pow(2, attempt - 1);
          console.log(`Rate limited, retrying in ${delay}ms...`);
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
        throw error;
      }
    }
  }
  
  async getUtxos(address) {
    return this.requestWithRetry(() => this.api.addressesUtxos(address));
  }
}
```

#### Issue: API key authentication
```javascript
// Debug API key issues
async function testBlockfrostConnection(projectId) {
  try {
    const api = new BlockFrost.BlockFrostAPI({
      projectId: projectId,
      network: 'preview' // or 'mainnet', 'preprod'
    });
    
    // Test connection
    const health = await api.health();
    console.log('API Health:', health);
    
    const tip = await api.blocksLatest();
    console.log('Latest block:', tip);
    
    return api;
  } catch (error) {
    if (error.status_code === 403) {
      console.error('Invalid API key or insufficient permissions');
    } else if (error.status_code === 418) {
      console.error('API key blocked due to abuse');
    } else {
      console.error('API connection failed:', error);
    }
    throw error;
  }
}
```

### Network Connectivity Issues

#### Issue: Node connection timeout
```bash
# Test network connectivity
ping preview-node.world.dev.cardano.org

# Test specific port
nc -zv preview-node.world.dev.cardano.org 30002

# Check firewall settings
sudo ufw status

# If behind corporate firewall, may need to configure proxy
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
```

#### Issue: Testnet vs Mainnet confusion
```javascript
// Always verify network configuration
function validateNetworkConfig(lucid, expectedNetwork) {
  const provider = lucid.provider;
  console.log('Provider network:', provider.network);
  
  if (provider.network !== expectedNetwork) {
    throw new Error(
      `Network mismatch: expected ${expectedNetwork}, got ${provider.network}`
    );
  }
}

// Usage
validateNetworkConfig(lucid, 'Preview');
```

## Build and Compilation Errors

### Plutus Build Issues

#### Issue: GHC version incompatibility
```bash
# Install specific GHC version
ghcup install ghc 8.10.7
ghcup set ghc 8.10.7

# Check version
ghc --version

# If cabal issues persist
cabal clean
cabal update
cabal build all
```

#### Issue: Cabal dependency conflicts
```bash
# Clear cabal cache
rm -rf ~/.cabal/store
rm -rf dist-newstyle

# Use specific resolver
cabal configure --constraint="plutus-core ==1.0.0.1"
cabal build

# Or use stack instead of cabal
stack build
```

### Aiken Build Issues

#### Issue: Aiken installation problems
```bash
# Ensure Rust is up to date
rustup update stable

# Clean install Aiken
cargo uninstall aiken
cargo install aiken --locked

# Verify installation
aiken --version
```

#### Issue: Aiken project build failures
```bash
# Clean build artifacts
aiken clean

# Check project structure
aiken check

# Rebuild from scratch
rm -rf build/
aiken build

# Enable verbose output for debugging
aiken build --verbose
```

### JavaScript Build Issues

#### Issue: Node.js version incompatibility
```bash
# Check Node.js version
node --version

# Use Node Version Manager for specific version
nvm install 18
nvm use 18

# Clear npm cache
npm cache clean --force

# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

## Debugging Strategies

### Transaction Debugging

#### Debug transaction construction
```javascript
function debugTransaction(tx) {
  console.log('=== Transaction Debug Info ===');
  console.log('Inputs:', tx.body.inputs.map(input => ({
    txHash: input.transaction_id,
    outputIndex: input.output_index
  })));
  
  console.log('Outputs:', tx.body.outputs.map(output => ({
    address: output.address,
    amount: output.amount.coin,
    assets: output.amount.multiasset || {}
  })));
  
  console.log('Fee:', tx.body.fee);
  console.log('TTL:', tx.body.ttl);
  console.log('Metadata:', tx.auxiliary_data?.metadata || 'None');
  console.log('Scripts:', tx.witness_set?.plutus_scripts || 'None');
  console.log('Datums:', tx.witness_set?.plutus_data || 'None');
  console.log('Redeemers:', tx.witness_set?.redeemers || 'None');
}
```

#### Debug UTxO state
```javascript
async function debugUTxOState(lucid, address) {
  const utxos = await lucid.utxosAt(address);
  
  console.log(`=== UTxOs at ${address} ===`);
  console.log(`Total UTxOs: ${utxos.length}`);
  
  let totalADA = 0n;
  const assets = new Map();
  
  utxos.forEach((utxo, index) => {
    console.log(`UTxO ${index}:`);
    console.log(`  TxHash: ${utxo.txHash}`);
    console.log(`  OutputIndex: ${utxo.outputIndex}`);
    console.log(`  Address: ${utxo.address}`);
    
    const lovelace = BigInt(utxo.assets.lovelace);
    totalADA += lovelace;
    console.log(`  ADA: ${lovelace / 1000000n}`);
    
    Object.entries(utxo.assets).forEach(([unit, quantity]) => {
      if (unit !== 'lovelace') {
        assets.set(unit, (assets.get(unit) || 0n) + BigInt(quantity));
        console.log(`  ${unit}: ${quantity}`);
      }
    });
    
    if (utxo.datum) {
      console.log(`  Datum: ${utxo.datum}`);
    }
    if (utxo.scriptRef) {
      console.log(`  Script Reference: Present`);
    }
  });
  
  console.log(`Total ADA: ${totalADA / 1000000n}`);
  console.log(`Native Assets:`, Object.fromEntries(assets));
}
```

### Smart Contract Debugging

#### Debug Plutus scripts
```haskell
-- Add comprehensive tracing
{-# INLINABLE debugValidator #-}
debugValidator :: MyDatum -> MyRedeemer -> ScriptContext -> Bool
debugValidator dat red ctx = 
  traceIfFalse "=== Validator Start ===" True &&
  traceIfFalse ("Datum: " ++ show dat) True &&
  traceIfFalse ("Redeemer: " ++ show red) True &&
  traceIfFalse ("TxInfo: " ++ show info) True &&
  traceIfFalse "=== Validation Logic ===" True &&
  actualValidation
  where
    info = scriptContextTxInfo ctx
    actualValidation = -- your validation logic here
```

#### Debug Aiken contracts
```rust
// Add debug traces in Aiken
validator {
  fn debug_spend(datum: Datum, redeemer: Redeemer, context: ScriptContext) -> Bool {
    trace @"=== Validation Start ==="
    trace @"Datum" datum
    trace @"Redeemer" redeemer
    trace @"Context" context
    
    let result = actual_validation_logic(datum, redeemer, context)
    
    if result {
      trace @"✓ Validation SUCCESS"
    } else {
      trace @"✗ Validation FAILED"
    }
    
    result
  }
}
```

### API Debugging

#### Debug API responses
```javascript
function createDebugAPI(api) {
  return new Proxy(api, {
    get(target, prop) {
      const originalMethod = target[prop];
      
      if (typeof originalMethod === 'function') {
        return async function(...args) {
          console.log(`API Call: ${prop}`, args);
          
          try {
            const result = await originalMethod.apply(target, args);
            console.log(`API Response: ${prop}`, result);
            return result;
          } catch (error) {
            console.error(`API Error: ${prop}`, error);
            throw error;
          }
        };
      }
      
      return originalMethod;
    }
  });
}

// Usage
const debugAPI = createDebugAPI(blockfrostAPI);
const utxos = await debugAPI.addressesUtxos(address);
```

### Performance Debugging

#### Monitor memory usage
```javascript
function monitorMemory(label) {
  if (typeof process !== 'undefined' && process.memoryUsage) {
    const usage = process.memoryUsage();
    console.log(`${label} - Memory:`, {
      rss: `${Math.round(usage.rss / 1024 / 1024)}MB`,
      heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
      heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
      external: `${Math.round(usage.external / 1024 / 1024)}MB`
    });
  }
}

// Usage
monitorMemory('Before transaction');
await buildTransaction();
monitorMemory('After transaction');
```

---

**Getting Help**: If you're still experiencing issues after trying these solutions, consider:
- Posting on [Cardano Stack Exchange](https://cardano.stackexchange.com/)
- Joining the [IOG Technical Discord](https://discord.gg/inputoutput)  
- Checking [GitHub Issues](https://github.com/input-output-hk/cardano-node/issues) for known problems
- Reviewing the [Developer Portal](https://developers.cardano.org/) for updated documentation