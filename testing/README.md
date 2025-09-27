# Testing Strategies for Cardano Development

Comprehensive guide to testing Cardano applications, smart contracts, and integrations.

## Table of Contents

- [Testing Overview](#testing-overview)
- [Unit Testing](#unit-testing)
- [Integration Testing](#integration-testing)
- [Smart Contract Testing](#smart-contract-testing)
- [End-to-End Testing](#end-to-end-testing)
- [Performance Testing](#performance-testing)
- [Security Testing](#security-testing)
- [Testing Tools](#testing-tools)

## Testing Overview

### Testing Pyramid for Cardano
```
    /\
   /  \     E2E Tests (Testnet/Mainnet)
  /____\    
 /      \   Integration Tests (Local/Testnet)
/________\  Unit Tests (Isolated Components)
```

### Testing Environments
- **Local**: Isolated component testing
- **Testnet**: Preview/Preprod for integration testing
- **Mainnet**: Production validation (read-only)

### Testing Types
- **Unit**: Individual functions and components
- **Integration**: API integrations and services
- **Contract**: Smart contract validation
- **End-to-End**: Full user workflows
- **Performance**: Load and stress testing
- **Security**: Vulnerability and penetration testing

## Unit Testing

### JavaScript/TypeScript Testing
```javascript
// Using Jest for unit testing
import { validateAddress, buildTransaction } from '../src/cardano-utils';
import { CardanoError, ErrorTypes } from '../src/errors';

describe('Cardano Utilities', () => {
  describe('validateAddress', () => {
    test('should validate mainnet address', () => {
      const validAddress = 'addr1qx2fxv2umyhttkxyxp8x0dlpdt3k6cwng5pxj3jhsydzer3jcu5d8ps7zex2k2xt3uqxgjqnnj83ws8lhrn648jjxtwq2ytjqp';
      expect(validateAddress(validAddress)).toBe(true);
    });

    test('should reject invalid address', () => {
      const invalidAddress = 'invalid_address';
      expect(validateAddress(invalidAddress)).toBe(false);
    });

    test('should validate testnet address', () => {
      const testnetAddress = 'addr_test1qz2fxv2umyhttkxyxp8x0dlpdt3k6cwng5pxj3jhsydzer3jcu5d8ps7zex2k2xt3uqxgjqnnj83ws8lhrn648jjxtwqcygnqq';
      expect(validateAddress(testnetAddress)).toBe(true);
    });
  });

  describe('buildTransaction', () => {
    test('should build valid payment transaction', async () => {
      const mockLucid = {
        newTx: () => ({
          payToAddress: jest.fn().mockReturnThis(),
          complete: jest.fn().mockResolvedValue({
            fee: () => 165381n,
            outputs: [
              { address: 'addr1...', amount: { coin: '2000000' } },
              { address: 'addr2...', amount: { coin: '7834619' } } // change
            ]
          })
        })
      };

      const tx = await buildTransaction(mockLucid, 'addr1...', { lovelace: 2000000n });
      
      expect(tx.outputs).toHaveLength(2);
      expect(tx.fee()).toBe(165381n);
    });

    test('should throw error for insufficient funds', async () => {
      const mockLucid = {
        newTx: () => ({
          payToAddress: jest.fn().mockReturnThis(),
          complete: jest.fn().mockRejectedValue(new Error('InsufficientFunds'))
        })
      };

      await expect(
        buildTransaction(mockLucid, 'addr1...', { lovelace: 1000000000n })
      ).rejects.toThrow(CardanoError);
    });
  });
});
```

### Python Testing with PyTest
```python
import pytest
from unittest.mock import Mock, patch
from pycardano import *
from src.cardano_service import CardanoService, CardanoServiceError

class TestCardanoService:
    @pytest.fixture
    def mock_context(self):
        context = Mock(spec=ChainContext)
        context.submit_tx.return_value = "tx_hash_123"
        return context
    
    @pytest.fixture
    def cardano_service(self, mock_context):
        return CardanoService(mock_context)
    
    def test_build_payment_transaction(self, cardano_service):
        sender_address = Address.from_bech32("addr_test1...")
        recipient_address = Address.from_bech32("addr_test1...")
        amount = Value(coin=2000000)
        
        tx = cardano_service.build_payment(sender_address, recipient_address, amount)
        
        assert len(tx.transaction_body.outputs) >= 1
        assert tx.transaction_body.outputs[0].amount.coin == 2000000
    
    def test_build_payment_invalid_address(self, cardano_service):
        with pytest.raises(CardanoServiceError, match="Invalid address"):
            cardano_service.build_payment("invalid", "addr_test1...", Value(coin=1000000))
    
    @patch('src.cardano_service.time.time')
    def test_transaction_timeout(self, mock_time, cardano_service):
        mock_time.return_value = 1640995200  # Fixed timestamp
        
        tx = cardano_service.build_payment_with_timeout(
            "addr_test1...", "addr_test1...", Value(coin=1000000), timeout=300
        )
        
        # Check that TTL is set correctly
        assert tx.transaction_body.ttl == 1640995200 + 300
```

### Rust Testing
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use cardano_serialization_lib as csl;

    #[test]
    fn test_build_transaction_output() {
        let address = csl::Address::from_bech32("addr_test1...").unwrap();
        let amount = csl::Value::new(&csl::BigNum::from_str("2000000").unwrap());
        
        let output = csl::TransactionOutput::new(&address, &amount);
        
        assert_eq!(output.amount().coin().to_str(), "2000000");
    }

    #[test]
    fn test_fee_calculation() {
        let linear_fee = csl::LinearFee::new(
            &csl::BigNum::from_str("44").unwrap(),
            &csl::BigNum::from_str("155381").unwrap(),
        );
        
        let tx_size = 300; // bytes
        let fee = linear_fee.min_fee(&csl::BigNum::from_str(&tx_size.to_string()).unwrap());
        
        // fee = 155381 + (44 * 300) = 168581
        assert_eq!(fee.to_str(), "168581");
    }

    #[test]
    #[should_panic(expected = "Invalid address")]
    fn test_invalid_address_panic() {
        csl::Address::from_bech32("invalid_address").unwrap();
    }
}
```

## Integration Testing

### API Integration Testing
```javascript
import { BlockFrost } from '@blockfrost/blockfrost-js';

describe('Blockfrost Integration', () => {
  let api;
  
  beforeAll(() => {
    api = new BlockFrost.BlockFrostAPI({
      projectId: process.env.BLOCKFROST_TEST_PROJECT_ID,
      network: 'testnet'
    });
  });

  test('should fetch latest block', async () => {
    const block = await api.blocksLatest();
    
    expect(block).toHaveProperty('hash');
    expect(block).toHaveProperty('height');
    expect(block).toHaveProperty('slot');
    expect(block.height).toBeGreaterThan(0);
  }, 10000);

  test('should fetch address UTxOs', async () => {
    const testAddress = 'addr_test1qz2fxv2umyhttkxyxp8x0dlpdt3k6cwng5pxj3jhsydzerpjqljdj9';
    
    const utxos = await api.addressesUtxos(testAddress);
    
    expect(Array.isArray(utxos)).toBe(true);
    if (utxos.length > 0) {
      expect(utxos[0]).toHaveProperty('tx_hash');
      expect(utxos[0]).toHaveProperty('amount');
    }
  }, 10000);

  test('should handle rate limiting gracefully', async () => {
    const promises = Array(20).fill().map(() => api.blocksLatest());
    
    // Should not throw rate limit errors due to internal handling
    const results = await Promise.allSettled(promises);
    const successful = results.filter(r => r.status === 'fulfilled');
    
    expect(successful.length).toBeGreaterThan(0);
  }, 30000);
});
```

### Database Integration Testing
```javascript
describe('Cardano DB Sync Integration', () => {
  let dbClient;
  
  beforeAll(async () => {
    dbClient = new Pool({
      host: process.env.DB_HOST || 'localhost',
      port: process.env.DB_PORT || 5432,
      database: process.env.DB_NAME || 'cardano_test',
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD
    });
    
    await dbClient.query('SELECT 1'); // Test connection
  });

  afterAll(async () => {
    await dbClient.end();
  });

  test('should query transaction by hash', async () => {
    const txHash = '\\x' + 'a1b2c3d4...'; // Test transaction hash
    
    const result = await dbClient.query(
      'SELECT encode(hash, \'hex\') as hash, out_sum FROM tx WHERE hash = $1',
      [Buffer.from(txHash.slice(2), 'hex')]
    );
    
    expect(result.rows.length).toBeGreaterThan(0);
    expect(result.rows[0]).toHaveProperty('hash');
    expect(result.rows[0]).toHaveProperty('out_sum');
  });

  test('should query address transactions', async () => {
    const address = 'addr_test1...';
    
    const result = await dbClient.query(`
      SELECT DISTINCT encode(tx.hash, 'hex') as tx_hash
      FROM tx
      JOIN tx_out ON tx.id = tx_out.tx_id
      WHERE tx_out.address = $1
      LIMIT 10
    `, [address]);
    
    expect(Array.isArray(result.rows)).toBe(true);
  });
});
```

## Smart Contract Testing

### Aiken Contract Testing
```rust
// tests/validators.ak
use aiken/dict
use aiken/list
use aiken/transaction.{ScriptContext, Spend, Transaction}
use validators/my_validator.{Datum, Redeemer, spend}

test spend_with_valid_signature() {
  let datum = Datum { owner: #"abc123", amount: 1000 }
  let redeemer = Redeemer { action: "unlock" }
  
  let tx = Transaction {
    inputs: [mock_input()],
    outputs: [mock_output()],
    extra_signatories: [#"abc123"],
    ..mock_transaction()
  }
  
  let context = ScriptContext {
    purpose: Spend(mock_output_reference()),
    transaction: tx
  }
  
  spend(datum, redeemer, context)
}

test spend_with_invalid_signature() {
  let datum = Datum { owner: #"abc123", amount: 1000 }
  let redeemer = Redeemer { action: "unlock" }
  
  let tx = Transaction {
    inputs: [mock_input()],
    outputs: [mock_output()],
    extra_signatories: [#"wrong_key"],
    ..mock_transaction()
  }
  
  let context = ScriptContext {
    purpose: Spend(mock_output_reference()),
    transaction: tx
  }
  
  !spend(datum, redeemer, context)
}

test spend_value_conservation() {
  let input_value = 10000000 // 10 ADA
  let output_value = 9800000 // 9.8 ADA (with fee)
  
  let datum = Datum { owner: #"abc123", amount: input_value }
  let redeemer = Redeemer { action: "unlock" }
  
  let context = mock_context_with_values(input_value, output_value)
  
  spend(datum, redeemer, context)
}

// Helper functions
fn mock_input() -> Input {
  Input {
    output_reference: mock_output_reference(),
    output: mock_output()
  }
}

fn mock_output() -> Output {
  Output {
    address: mock_address(),
    value: mock_value(2000000),
    datum: NoDatum,
    reference_script: None
  }
}
```

### Plutus Property-Based Testing
```haskell
import Test.QuickCheck
import Plutus.V2.Ledger.Api

-- Property: Validator should always succeed with correct signature
prop_validatorSucceedsWithCorrectSignature :: Integer -> PubKeyHash -> Property
prop_validatorSucceedsWithCorrectSignature amount pkh = 
  amount > 0 ==> 
    let datum = MyDatum { owner = pkh, amount = amount }
        redeemer = Unlock
        ctx = mockScriptContext { scriptContextTxInfo = 
                mockTxInfo { txInfoSignatories = [pkh] } }
    in myValidator datum redeemer ctx

-- Property: Validator should never succeed without signature
prop_validatorFailsWithoutSignature :: Integer -> PubKeyHash -> PubKeyHash -> Property
prop_validatorSucceedsWithCorrectSignature amount ownerPkh wrongPkh = 
  amount > 0 && ownerPkh /= wrongPkh ==> 
    let datum = MyDatum { owner = ownerPkh, amount = amount }
        redeemer = Unlock
        ctx = mockScriptContext { scriptContextTxInfo = 
                mockTxInfo { txInfoSignatories = [wrongPkh] } }
    in not (myValidator datum redeemer ctx)

-- Property: Fee calculation should be deterministic
prop_feeCalculationDeterministic :: [TxInInfo] -> [TxOut] -> Property
prop_feeCalculationDeterministic inputs outputs =
  length inputs > 0 && length outputs > 0 ==>
    let fee1 = calculateFee inputs outputs
        fee2 = calculateFee inputs outputs
    in fee1 == fee2

main :: IO ()
main = do
  quickCheck prop_validatorSucceedsWithCorrectSignature
  quickCheck prop_validatorFailsWithoutSignature
  quickCheck prop_feeCalculationDeterministic
```

### Smart Contract Simulation
```javascript
// Using Lucid for contract simulation
describe('Smart Contract Simulation', () => {
  let lucid;
  let validator;
  let scriptAddress;

  beforeAll(async () => {
    lucid = await Lucid.new(undefined, "Custom");
    
    // Load compiled validator
    validator = await import('./plutus/my-validator.json');
    scriptAddress = lucid.utils.validatorToAddress(validator);
  });

  test('should lock and unlock funds', async () => {
    // Create test wallet
    const privateKey = lucid.utils.generatePrivateKey();
    lucid.selectWalletFromPrivateKey(privateKey);
    
    // Add test UTxOs
    lucid.utils.addUtxos([
      {
        txHash: "a".repeat(64),
        outputIndex: 0,
        assets: { lovelace: 10000000n },
        address: await lucid.wallet.address(),
        datumHash: null,
        datum: null,
        scriptRef: null
      }
    ]);

    // Lock funds at script
    const ownerPkh = lucid.utils.getAddressDetails(await lucid.wallet.address()).paymentCredential.hash;
    const datum = Data.to(new Constr(0, [ownerPkh, 1000n]));

    const lockTx = await lucid
      .newTx()
      .payToContract(scriptAddress, { inline: datum }, { lovelace: 5000000n })
      .complete();

    const lockSigned = await lockTx.sign().complete();
    const lockTxHash = await lockSigned.submit();

    // Add the locked UTxO for unlocking
    const lockedUtxo = {
      txHash: lockTxHash,
      outputIndex: 0,
      assets: { lovelace: 5000000n },
      address: scriptAddress,
      datum: datum,
      datumHash: null,
      scriptRef: null
    };

    lucid.utils.addUtxos([lockedUtxo]);

    // Unlock funds
    const redeemer = Data.to(new Constr(0, [])); // Unlock redeemer

    const unlockTx = await lucid
      .newTx()
      .collectFrom([lockedUtxo], redeemer)
      .attachSpendingValidator(validator)
      .complete();

    const unlockSigned = await unlockTx.sign().complete();
    const unlockTxHash = await unlockSigned.submit();

    expect(lockTxHash).toMatch(/^[a-f0-9]{64}$/);
    expect(unlockTxHash).toMatch(/^[a-f0-9]{64}$/);
  });
});
```

## End-to-End Testing

### Testnet E2E Testing
```javascript
describe('End-to-End Testnet Tests', () => {
  let senderWallet;
  let recipientWallet;
  let lucid;

  beforeAll(async () => {
    // Setup testnet connection
    lucid = await Lucid.new(
      new Blockfrost(
        'https://cardano-preview.blockfrost.io/api/v0',
        process.env.BLOCKFROST_PREVIEW_KEY
      ),
      'Preview'
    );

    // Create test wallets
    senderWallet = lucid.utils.generateSeedPhrase();
    recipientWallet = lucid.utils.generateSeedPhrase();
    
    // Fund sender wallet from faucet
    await fundFromFaucet(
      lucid.utils.getAddressFromSeed(senderWallet, 'Preview')
    );
    
    // Wait for funding to be confirmed
    await waitForFunds(
      lucid.utils.getAddressFromSeed(senderWallet, 'Preview'),
      10000000n
    );
  });

  test('complete payment workflow', async () => {
    lucid.selectWalletFromSeed(senderWallet);
    
    const senderAddress = await lucid.wallet.address();
    const recipientAddress = lucid.utils.getAddressFromSeed(recipientWallet, 'Preview');
    
    // Check initial balances
    const initialSenderUtxos = await lucid.wallet.getUtxos();
    const initialSenderBalance = initialSenderUtxos.reduce(
      (sum, utxo) => sum + BigInt(utxo.assets.lovelace), 0n
    );
    
    const initialRecipientUtxos = await lucid.utxosAt(recipientAddress);
    const initialRecipientBalance = initialRecipientUtxos.reduce(
      (sum, utxo) => sum + BigInt(utxo.assets.lovelace), 0n
    );

    // Send payment
    const paymentAmount = 2000000n; // 2 ADA
    const tx = await lucid
      .newTx()
      .payToAddress(recipientAddress, { lovelace: paymentAmount })
      .complete();

    const signedTx = await tx.sign().complete();
    const txHash = await signedTx.submit();

    // Wait for confirmation
    await waitForConfirmation(txHash, 300000); // 5 minutes timeout

    // Verify final balances
    const finalSenderUtxos = await lucid.wallet.getUtxos();
    const finalSenderBalance = finalSenderUtxos.reduce(
      (sum, utxo) => sum + BigInt(utxo.assets.lovelace), 0n
    );
    
    const finalRecipientUtxos = await lucid.utxosAt(recipientAddress);
    const finalRecipientBalance = finalRecipientUtxos.reduce(
      (sum, utxo) => sum + BigInt(utxo.assets.lovelace), 0n
    );

    // Assertions
    expect(finalRecipientBalance).toBe(initialRecipientBalance + paymentAmount);
    expect(finalSenderBalance).toBeLessThan(initialSenderBalance - paymentAmount);
    
    // Verify transaction appears in blockchain
    const confirmedTx = await lucid.provider.getTransaction(txHash);
    expect(confirmedTx).toBeTruthy();
    expect(confirmedTx.outputs.some(output => 
      output.address === recipientAddress && 
      BigInt(output.amount.coin) === paymentAmount
    )).toBe(true);
  }, 600000); // 10 minutes timeout

  test('native token workflow', async () => {
    lucid.selectWalletFromSeed(senderWallet);
    
    // Create minting policy
    const { paymentCredential } = lucid.utils.getAddressDetails(await lucid.wallet.address());
    const mintingPolicy = lucid.utils.nativeScriptFromJson({
      type: "sig",
      keyHash: paymentCredential.hash
    });
    
    const policyId = lucid.utils.mintingPolicyToId(mintingPolicy);
    const assetName = "TestToken";
    const assetUnit = policyId + assetName;
    
    // Mint tokens
    const mintTx = await lucid
      .newTx()
      .mintAssets({ [assetUnit]: 100n })
      .attachMintingPolicy(mintingPolicy)
      .complete();
    
    const mintSigned = await mintTx.sign().complete();
    const mintTxHash = await mintSigned.submit();
    
    await waitForConfirmation(mintTxHash);
    
    // Send tokens
    const recipientAddress = lucid.utils.getAddressFromSeed(recipientWallet, 'Preview');
    const sendTx = await lucid
      .newTx()
      .payToAddress(recipientAddress, { 
        lovelace: 2000000n,
        [assetUnit]: 50n
      })
      .complete();
    
    const sendSigned = await sendTx.sign().complete();
    const sendTxHash = await sendSigned.submit();
    
    await waitForConfirmation(sendTxHash);
    
    // Verify recipient received tokens
    const recipientUtxos = await lucid.utxosAt(recipientAddress);
    const tokenBalance = recipientUtxos.reduce(
      (sum, utxo) => sum + BigInt(utxo.assets[assetUnit] || 0), 0n
    );
    
    expect(tokenBalance).toBe(50n);
  }, 600000);
});

// Helper functions
async function fundFromFaucet(address) {
  const response = await fetch('https://faucet.preview.world.dev.cardano.org/send-money', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ address, amount: 10000000000 }) // 10,000 ADA
  });
  
  if (!response.ok) {
    throw new Error(`Faucet request failed: ${response.statusText}`);
  }
}

async function waitForFunds(address, minimumAmount, maxWait = 300000) {
  const startTime = Date.now();
  
  while (Date.now() - startTime < maxWait) {
    const utxos = await lucid.utxosAt(address);
    const balance = utxos.reduce((sum, utxo) => sum + BigInt(utxo.assets.lovelace), 0n);
    
    if (balance >= minimumAmount) {
      return balance;
    }
    
    await new Promise(resolve => setTimeout(resolve, 10000)); // Wait 10 seconds
  }
  
  throw new Error(`Timeout waiting for funds at ${address}`);
}

async function waitForConfirmation(txHash, maxWait = 300000) {
  const startTime = Date.now();
  
  while (Date.now() - startTime < maxWait) {
    try {
      const tx = await lucid.provider.getTransaction(txHash);
      if (tx) {
        return tx;
      }
    } catch (error) {
      // Transaction not yet confirmed
    }
    
    await new Promise(resolve => setTimeout(resolve, 10000)); // Wait 10 seconds
  }
  
  throw new Error(`Transaction ${txHash} not confirmed within timeout`);
}
```

## Performance Testing

### Load Testing
```javascript
// Using Artillery.js for load testing
// artillery.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
  environments:
    production:
      target: 'https://api.myapp.com'

scenarios:
  - name: "Payment API Load Test"
    flow:
      - post:
          url: "/api/payments"
          json:
            recipient: "addr_test1..."
            amount: 2000000
            metadata:
              purpose: "load test"
          expect:
            - statusCode: 200
            - hasProperty: "txHash"
      
      - get:
          url: "/api/transactions/{{ txHash }}"
          expect:
            - statusCode: 200

  - name: "Balance Query Load Test"
    flow:
      - get:
          url: "/api/addresses/{{ $randomString() }}/balance"
          expect:
            - statusCode: [200, 404]

# Run with: artillery run artillery.yml
```

### Stress Testing
```javascript
describe('Stress Tests', () => {
  test('concurrent transaction building', async () => {
    const concurrentRequests = 50;
    const promises = [];
    
    for (let i = 0; i < concurrentRequests; i++) {
      promises.push(
        buildTransaction(
          `addr_test1_${i}`,
          { lovelace: 2000000n + BigInt(i * 1000) }
        )
      );
    }
    
    const results = await Promise.allSettled(promises);
    const successful = results.filter(r => r.status === 'fulfilled');
    const failed = results.filter(r => r.status === 'rejected');
    
    console.log(`Successful: ${successful.length}, Failed: ${failed.length}`);
    
    // Should handle at least 80% successfully
    expect(successful.length / concurrentRequests).toBeGreaterThan(0.8);
  });

  test('memory usage under load', async () => {
    const initialMemory = process.memoryUsage();
    
    // Generate many transactions
    for (let i = 0; i < 1000; i++) {
      await buildTransaction(`addr_test1_${i}`, { lovelace: 2000000n });
    }
    
    // Force garbage collection if available
    if (global.gc) {
      global.gc();
    }
    
    const finalMemory = process.memoryUsage();
    const memoryIncrease = finalMemory.heapUsed - initialMemory.heapUsed;
    
    // Memory increase should be reasonable (less than 100MB)
    expect(memoryIncrease).toBeLessThan(100 * 1024 * 1024);
  });
});
```

## Security Testing

### Vulnerability Testing
```javascript
describe('Security Tests', () => {
  test('should prevent SQL injection in address queries', async () => {
    const maliciousAddress = "addr1'; DROP TABLE transactions; --";
    
    await expect(
      queryAddressTransactions(maliciousAddress)
    ).rejects.toThrow('Invalid address format');
  });

  test('should validate transaction amounts', async () => {
    const invalidAmounts = [
      -1000000n,           // Negative amount
      0n,                  // Zero amount
      45000000000000001n,  // Exceeds max ADA supply
    ];
    
    for (const amount of invalidAmounts) {
      await expect(
        buildTransaction('addr_test1...', { lovelace: amount })
      ).rejects.toThrow();
    }
  });

  test('should sanitize metadata inputs', () => {
    const maliciousMetadata = {
      '<script>alert("xss")</script>': 'value',
      'key': '<script>alert("xss")</script>',
      'very_long_key_'.repeat(100): 'value'
    };
    
    const sanitized = sanitizeMetadata(maliciousMetadata);
    
    expect(Object.keys(sanitized).every(key => key.length <= 64)).toBe(true);
    expect(Object.values(sanitized).every(value => 
      typeof value !== 'string' || value.length <= 64
    )).toBe(true);
  });

  test('should rate limit API calls', async () => {
    const rapidRequests = Array(100).fill().map(() => 
      fetch('/api/balance/addr_test1...')
    );
    
    const results = await Promise.allSettled(rapidRequests);
    const rateLimited = results.filter(r => 
      r.status === 'rejected' || 
      (r.status === 'fulfilled' && r.value.status === 429)
    );
    
    expect(rateLimited.length).toBeGreaterThan(0);
  });
});
```

## Testing Tools

### Test Configuration
```json
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.js'],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js'],
  testTimeout: 30000,
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/test-utils/**'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

### Test Utilities
```javascript
// tests/test-utils.js
export const createMockLucid = (utxos = []) => ({
  newTx: () => ({
    payToAddress: jest.fn().mockReturnThis(),
    collectFrom: jest.fn().mockReturnThis(),
    attachSpendingValidator: jest.fn().mockReturnThis(),
    complete: jest.fn().mockResolvedValue({
      fee: () => 165381n,
      sign: () => ({
        complete: jest.fn().mockResolvedValue({
          submit: jest.fn().mockResolvedValue('tx_hash_123')
        })
      })
    })
  }),
  wallet: {
    address: jest.fn().mockResolvedValue('addr_test1...'),
    getUtxos: jest.fn().mockResolvedValue(utxos)
  },
  utils: {
    getAddressDetails: jest.fn().mockReturnValue({
      paymentCredential: { hash: 'payment_hash' }
    })
  }
});

export const createTestTransaction = (inputs, outputs) => ({
  inputs,
  outputs,
  fee: 165381,
  ttl: null,
  certificates: [],
  withdrawals: {},
  mint: {},
  metadataHash: null,
  validityStart: null,
  scriptDataHash: null,
  collateral: [],
  requiredSigners: [],
  networkId: 0
});

export const waitForCondition = async (condition, timeout = 30000) => {
  const startTime = Date.now();
  
  while (Date.now() - startTime < timeout) {
    if (await condition()) {
      return true;
    }
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  throw new Error('Condition not met within timeout');
};
```

### Continuous Integration
```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:coverage

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: cardano_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:integration
        env:
          DB_HOST: localhost
          DB_PASSWORD: postgres
          BLOCKFROST_TEST_PROJECT_ID: ${{ secrets.BLOCKFROST_TEST_PROJECT_ID }}

  e2e-tests:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:e2e
        env:
          BLOCKFROST_PREVIEW_KEY: ${{ secrets.BLOCKFROST_PREVIEW_KEY }}
```

---

**Next Steps**:
- Review [Best Practices](../best-practices/README.md) for testing guidelines
- Check [Smart Contracts](../smart-contracts/README.md) for contract-specific testing
- See [Development](../development/README.md) for testing tool setup
- Explore [Troubleshooting](../troubleshooting/README.md) for debugging failing tests