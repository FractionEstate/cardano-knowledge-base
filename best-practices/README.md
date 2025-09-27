# Cardano Development Best Practices

Essential security, performance, and architectural guidelines for Cardano development.

## Table of Contents

- [Security Best Practices](#security-best-practices)
- [Smart Contract Security](#smart-contract-security)
- [Transaction Safety](#transaction-safety)
- [Performance Optimization](#performance-optimization)
- [Error Handling](#error-handling)
- [Testing Strategies](#testing-strategies)
- [Code Quality](#code-quality)
- [Deployment Guidelines](#deployment-guidelines)

## Security Best Practices

### Private Key Management
```javascript
// ❌ Never do this
const privateKey = "ed25519_sk1abc123..."; // Hardcoded private key
localStorage.setItem('privateKey', privateKey); // Storing in browser storage

// ✅ Best practices
// Use environment variables for server-side applications
const privateKey = process.env.CARDANO_PRIVATE_KEY;

// Use secure storage for client applications
import { SecureStore } from 'expo-secure-store';
await SecureStore.setItemAsync('privateKey', privateKey);

// Use hardware wallets when possible
const ledgerWallet = new LedgerWallet();
```

### Environment Separation
```bash
# ✅ Use different configurations for different environments
# .env.development
CARDANO_NETWORK=preview
BLOCKFROST_PROJECT_ID=preview_abc123
CARDANO_TESTNET_MAGIC=2

# .env.production  
CARDANO_NETWORK=mainnet
BLOCKFROST_PROJECT_ID=mainnet_xyz789
CARDANO_TESTNET_MAGIC=764824073
```

### API Key Security
```javascript
// ❌ Exposed API keys
const API = new BlockFrost.BlockFrostAPI({
  projectId: 'mainnet_abc123def456' // Hardcoded API key
});

// ✅ Environment-based configuration
const API = new BlockFrost.BlockFrostAPI({
  projectId: process.env.BLOCKFROST_PROJECT_ID,
  network: process.env.CARDANO_NETWORK
});

// ✅ Server-side proxy for client applications
const response = await fetch('/api/cardano/utxos', {
  method: 'POST',
  body: JSON.stringify({ address }),
  headers: { 'Content-Type': 'application/json' }
});
```

### Input Validation
```javascript
// ✅ Validate all inputs
function validateAddress(address) {
  try {
    CardanoWasm.Address.from_bech32(address);
    return true;
  } catch (error) {
    return false;
  }
}

function validateADAAmount(amount) {
  const parsedAmount = BigInt(amount);
  return parsedAmount > 0n && parsedAmount <= 45000000000000000n; // Max ADA supply
}

// ✅ Sanitize user inputs
function sanitizeMetadata(metadata) {
  return Object.fromEntries(
    Object.entries(metadata).map(([key, value]) => [
      key.slice(0, 64), // Limit key length
      typeof value === 'string' ? value.slice(0, 64) : value // Limit string length
    ])
  );
}
```

## Smart Contract Security

### Access Control Patterns
```rust
// Aiken: Owner-only access
validator {
  fn spend(datum: Datum, redeemer: Redeemer, context: ScriptContext) -> Bool {
    when context.purpose is {
      Spend(_) -> {
        // ✅ Always verify signatures
        let must_be_signed_by_owner = 
          list.has(context.transaction.extra_signatories, datum.owner)
        
        // ✅ Check specific redeemer actions
        when redeemer is {
          Unlock -> must_be_signed_by_owner
          Update(new_data) -> must_be_signed_by_owner && validate_data(new_data)
          _ -> False
        }
      }
      _ -> False
    }
  }
}
```

### Time-Based Validation
```rust
// ✅ Proper time validation
fn check_deadline(tx: Transaction, deadline: Int) -> Bool {
  when tx.validity_range.upper_bound.bound_type is {
    Finite(upper) -> upper <= deadline
    _ -> False // Reject infinite validity ranges for time-sensitive operations
  }
}

fn check_minimum_time(tx: Transaction, minimum_time: Int) -> Bool {
  when tx.validity_range.lower_bound.bound_type is {
    Finite(lower) -> lower >= minimum_time
    _ -> False // Always require minimum time bounds
  }
}
```

### Value Preservation
```rust
// ✅ Ensure value conservation
fn validate_value_conservation(
  inputs: List<Value>,
  outputs: List<Value>,
  mint: Value
) -> Bool {
  let total_input = list.foldl(inputs, zero(), add)
  let total_output = list.foldl(outputs, zero(), add)
  
  // Input + Mint = Output + Fee (fee handled by ledger)
  add(total_input, mint) == total_output
}
```

### Oracle Integration Security
```rust
// ✅ Secure oracle usage
fn validate_oracle_data(tx: Transaction, oracle_nft: AssetName) -> Option<Int> {
  let oracle_inputs = 
    list.filter(tx.inputs, fn(input) { 
      value.quantity_of(input.output.value, oracle_policy_id, oracle_nft) == 1 
    })
  
  when oracle_inputs is {
    [oracle_input] -> {
      // ✅ Validate oracle signature
      let oracle_signed = list.has(tx.extra_signatories, oracle_pubkey)
      
      // ✅ Check data freshness
      let data_timestamp = get_timestamp_from_datum(oracle_input.output.datum)
      let current_time = get_current_time(tx)
      let is_fresh = current_time - data_timestamp < max_age
      
      if oracle_signed && is_fresh {
        Some(extract_price(oracle_input.output.datum))
      } else {
        None
      }
    }
    _ -> None // Must have exactly one oracle input
  }
}
```

## Transaction Safety

### UTxO Selection Strategy
```javascript
// ✅ Intelligent UTxO selection
async function selectUTxOs(utxos, targetAmount) {
  // Sort by ADA amount (largest first)
  const sortedUtxos = utxos.sort((a, b) => 
    BigInt(b.output.amount.coin) - BigInt(a.output.amount.coin)
  );
  
  let selectedUtxos = [];
  let totalAmount = 0n;
  
  for (const utxo of sortedUtxos) {
    selectedUtxos.push(utxo);
    totalAmount += BigInt(utxo.output.amount.coin);
    
    if (totalAmount >= targetAmount) {
      break;
    }
  }
  
  if (totalAmount < targetAmount) {
    throw new Error('Insufficient funds');
  }
  
  return selectedUtxos;
}
```

### Fee Estimation
```javascript
// ✅ Accurate fee estimation
async function buildTransactionWithFee(lucid, outputs) {
  // Build initial transaction for size estimation
  const initialTx = lucid
    .newTx()
    .payToAddress(outputs[0].address, outputs[0].amount);
  
  for (let i = 1; i < outputs.length; i++) {
    initialTx.payToAddress(outputs[i].address, outputs[i].amount);
  }
  
  // Estimate fee
  const estimatedFee = await initialTx.fee();
  
  // Add buffer for potential size changes
  const safetyBuffer = estimatedFee / 10n; // 10% buffer
  const finalFee = estimatedFee + safetyBuffer;
  
  // Build final transaction
  return await initialTx
    .payToAddress(outputs[0].address, outputs[0].amount)
    .complete();
}
```

### Collision Prevention
```javascript
// ✅ Prevent UTxO collisions in concurrent environments
class UTxOManager {
  constructor() {
    this.lockedUtxos = new Set();
  }
  
  async lockUtxos(utxos) {
    const utxoIds = utxos.map(utxo => `${utxo.input.txHash}#${utxo.input.outputIndex}`);
    
    // Check for conflicts
    for (const id of utxoIds) {
      if (this.lockedUtxos.has(id)) {
        throw new Error(`UTxO ${id} is already locked`);
      }
    }
    
    // Lock UTxOs
    utxoIds.forEach(id => this.lockedUtxos.add(id));
    
    return () => {
      // Release function
      utxoIds.forEach(id => this.lockedUtxos.delete(id));
    };
  }
}
```

## Performance Optimization

### Batch Operations
```javascript
// ✅ Batch multiple operations
async function batchPayments(lucid, payments) {
  if (payments.length === 0) return [];
  
  // Group payments to reduce transactions
  const batchSize = 10; // Adjust based on transaction size limits
  const batches = [];
  
  for (let i = 0; i < payments.length; i += batchSize) {
    batches.push(payments.slice(i, i + batchSize));
  }
  
  const results = [];
  for (const batch of batches) {
    let tx = lucid.newTx();
    
    for (const payment of batch) {
      tx = tx.payToAddress(payment.address, payment.amount);
    }
    
    const completeTx = await tx.complete();
    const signedTx = await completeTx.sign().complete();
    const txHash = await signedTx.submit();
    
    results.push(txHash);
    
    // Wait between batches to avoid overwhelming the network
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  return results;
}
```

### Caching Strategies
```javascript
// ✅ Implement intelligent caching
class CardanoCache {
  constructor(ttl = 60000) { // 1 minute default TTL
    this.cache = new Map();
    this.ttl = ttl;
  }
  
  set(key, value) {
    this.cache.set(key, {
      value,
      timestamp: Date.now()
    });
  }
  
  get(key) {
    const entry = this.cache.get(key);
    if (!entry) return null;
    
    if (Date.now() - entry.timestamp > this.ttl) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.value;
  }
  
  async getOrFetch(key, fetchFn) {
    let value = this.get(key);
    if (value === null) {
      value = await fetchFn();
      this.set(key, value);
    }
    return value;
  }
}

// Usage
const cache = new CardanoCache(300000); // 5 minutes

const utxos = await cache.getOrFetch(
  `utxos:${address}`,
  () => lucid.provider.getUtxos(address)
);
```

### Connection Pooling
```javascript
// ✅ Manage API connections efficiently
class APIConnectionManager {
  constructor(maxConnections = 10) {
    this.connections = [];
    this.maxConnections = maxConnections;
    this.waitQueue = [];
  }
  
  async getConnection() {
    if (this.connections.length < this.maxConnections) {
      const connection = new BlockFrost.BlockFrostAPI({
        projectId: process.env.BLOCKFROST_PROJECT_ID
      });
      this.connections.push(connection);
      return connection;
    }
    
    // Wait for available connection
    return new Promise((resolve) => {
      this.waitQueue.push(resolve);
    });
  }
  
  releaseConnection(connection) {
    if (this.waitQueue.length > 0) {
      const resolve = this.waitQueue.shift();
      resolve(connection);
    }
  }
}
```

## Error Handling

### Comprehensive Error Management
```javascript
// ✅ Structured error handling
class CardanoError extends Error {
  constructor(message, type, details = {}) {
    super(message);
    this.name = 'CardanoError';
    this.type = type;
    this.details = details;
  }
}

// Error types
const ErrorTypes = {
  INSUFFICIENT_FUNDS: 'INSUFFICIENT_FUNDS',
  INVALID_ADDRESS: 'INVALID_ADDRESS',
  NETWORK_ERROR: 'NETWORK_ERROR',
  SCRIPT_FAILURE: 'SCRIPT_FAILURE',
  TRANSACTION_TOO_LARGE: 'TRANSACTION_TOO_LARGE'
};

// Error handling wrapper
async function safeCardanoOperation(operation) {
  try {
    return await operation();
  } catch (error) {
    if (error.message.includes('InsufficientFunds')) {
      throw new CardanoError(
        'Insufficient funds for transaction',
        ErrorTypes.INSUFFICIENT_FUNDS,
        { originalError: error.message }
      );
    }
    
    if (error.message.includes('ScriptFailure')) {
      throw new CardanoError(
        'Smart contract validation failed',
        ErrorTypes.SCRIPT_FAILURE,
        { originalError: error.message }
      );
    }
    
    // Re-throw with additional context
    throw new CardanoError(
      `Cardano operation failed: ${error.message}`,
      ErrorTypes.NETWORK_ERROR,
      { originalError: error }
    );
  }
}
```

### Retry Logic
```javascript
// ✅ Intelligent retry mechanism
async function withRetry(
  operation,
  maxRetries = 3,
  baseDelay = 1000,
  backoffMultiplier = 2
) {
  let lastError;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      
      // Don't retry certain errors
      if (error.type === ErrorTypes.INVALID_ADDRESS ||
          error.type === ErrorTypes.INSUFFICIENT_FUNDS) {
        throw error;
      }
      
      if (attempt === maxRetries) {
        break;
      }
      
      // Exponential backoff
      const delay = baseDelay * Math.pow(backoffMultiplier, attempt - 1);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  
  throw lastError;
}
```

## Testing Strategies

### Unit Testing
```javascript
// ✅ Comprehensive unit tests
describe('Transaction Builder', () => {
  test('should build valid payment transaction', async () => {
    const mockLucid = createMockLucid();
    const builder = new TransactionBuilder(mockLucid);
    
    const tx = await builder
      .payToAddress('addr_test1...', { lovelace: 2000000n })
      .build();
    
    expect(tx.outputs).toHaveLength(2); // Payment + change
    expect(tx.outputs[0].amount.coin).toBe('2000000');
  });
  
  test('should handle insufficient funds gracefully', async () => {
    const mockLucid = createMockLucidWithLowFunds();
    const builder = new TransactionBuilder(mockLucid);
    
    await expect(
      builder.payToAddress('addr_test1...', { lovelace: 1000000000n }).build()
    ).rejects.toThrow(CardanoError);
  });
});
```

### Integration Testing
```javascript
// ✅ End-to-end testing on testnet
describe('Integration Tests', () => {
  beforeAll(async () => {
    // Setup testnet environment
    lucid = await Lucid.new(
      new Blockfrost('https://cardano-preview.blockfrost.io/api/v0', testnetProjectId),
      'Preview'
    );
  });
  
  test('should execute full payment flow', async () => {
    const senderWallet = generateTestWallet();
    const recipientAddress = 'addr_test1...';
    
    // Fund sender wallet from faucet
    await fundFromFaucet(senderWallet.address);
    
    // Execute payment
    const txHash = await sendPayment(
      senderWallet,
      recipientAddress,
      { lovelace: 2000000n }
    );
    
    // Verify transaction confirmation
    await waitForConfirmation(txHash);
    
    // Verify recipient balance
    const recipientUtxos = await lucid.utxosAt(recipientAddress);
    const totalReceived = recipientUtxos.reduce(
      (sum, utxo) => sum + BigInt(utxo.assets.lovelace),
      0n
    );
    
    expect(totalReceived).toBeGreaterThanOrEqual(2000000n);
  });
});
```

## Code Quality

### Type Safety
```typescript
// ✅ Use TypeScript for type safety
interface PaymentRequest {
  address: string;
  amount: Assets;
  metadata?: Metadata;
}

interface TransactionResult {
  txHash: string;
  fee: bigint;
  submittedAt: Date;
}

class CardanoService {
  async sendPayment(request: PaymentRequest): Promise<TransactionResult> {
    // Implementation with full type safety
  }
}
```

### Documentation
```javascript
/**
 * Builds and submits a payment transaction
 * @param {string} recipientAddress - Bech32 encoded Cardano address
 * @param {Assets} amount - Amount to send (lovelace and native tokens)
 * @param {Object} options - Additional options
 * @param {Metadata} options.metadata - Optional transaction metadata
 * @param {number} options.ttl - Time to live in seconds
 * @returns {Promise<string>} Transaction hash
 * @throws {CardanoError} When transaction fails
 * 
 * @example
 * const txHash = await sendPayment(
 *   'addr1...',
 *   { lovelace: 2000000n },
 *   { metadata: { msg: 'Hello Cardano' } }
 * );
 */
async function sendPayment(recipientAddress, amount, options = {}) {
  // Implementation
}
```

### Linting and Formatting
```json
// .eslintrc.json
{
  "extends": ["eslint:recommended", "@typescript-eslint/recommended"],
  "rules": {
    "no-unused-vars": "error",
    "prefer-const": "error",
    "no-var": "error",
    "@typescript-eslint/explicit-function-return-type": "warn"
  }
}

// prettier.config.js
module.exports = {
  semi: true,
  trailingComma: 'es5',
  singleQuote: true,
  printWidth: 100,
  tabWidth: 2
};
```

## Deployment Guidelines

### Environment Configuration
```yaml
# docker-compose.production.yml
version: '3.8'
services:
  cardano-app:
    image: my-cardano-app:latest
    environment:
      - NODE_ENV=production
      - CARDANO_NETWORK=mainnet
      - BLOCKFROST_PROJECT_ID=${BLOCKFROST_MAINNET_KEY}
      - LOG_LEVEL=info
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Monitoring and Alerting
```javascript
// ✅ Application monitoring
const prometheus = require('prom-client');

const transactionCounter = new prometheus.Counter({
  name: 'cardano_transactions_total',
  help: 'Total number of Cardano transactions processed',
  labelNames: ['status', 'network']
});

const transactionDuration = new prometheus.Histogram({
  name: 'cardano_transaction_duration_seconds',
  help: 'Duration of Cardano transaction processing',
  buckets: [0.1, 0.5, 1, 2, 5, 10]
});

async function monitoredSendPayment(address, amount) {
  const startTime = Date.now();
  
  try {
    const result = await sendPayment(address, amount);
    transactionCounter.labels('success', process.env.CARDANO_NETWORK).inc();
    return result;
  } catch (error) {
    transactionCounter.labels('error', process.env.CARDANO_NETWORK).inc();
    throw error;
  } finally {
    const duration = (Date.now() - startTime) / 1000;
    transactionDuration.observe(duration);
  }
}
```

### Security Checklist
- [ ] Private keys never stored in code or logs
- [ ] API keys properly secured and rotated
- [ ] All user inputs validated and sanitized
- [ ] Smart contracts audited by security professionals
- [ ] Rate limiting implemented for public APIs
- [ ] Monitoring and alerting configured
- [ ] Backup and disaster recovery procedures in place
- [ ] Regular security updates applied
- [ ] Access controls properly configured
- [ ] Network security measures implemented

---

**Next Steps**:
- Review [Testing](../testing/README.md) for detailed testing strategies
- Check [Smart Contracts](../smart-contracts/README.md) for contract-specific security
- See [Deployment](../deployment/README.md) for production deployment guides
- Explore [Troubleshooting](../troubleshooting/README.md) for common issues and solutions