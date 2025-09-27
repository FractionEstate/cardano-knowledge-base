# Wallet Integration for Cardano DApps

Comprehensive guide to integrating Cardano wallets into decentralized applications.

## Table of Contents

- [Wallet Integration Overview](#wallet-integration-overview)
- [CIP-30 Browser Wallets](#cip-30-browser-wallets)
- [Wallet Detection and Selection](#wallet-detection-and-selection)
- [Transaction Signing](#transaction-signing)
- [State Management](#state-management)
- [Error Handling](#error-handling)
- [Security Considerations](#security-considerations)
- [Popular Wallets](#popular-wallets)

## Wallet Integration Overview

### Integration Methods
- **CIP-30**: Browser extension wallets (Nami, Eternl, Flint, etc.)
- **WalletConnect**: Mobile wallet connection
- **Deep Links**: Mobile app integration
- **QR Codes**: Cross-platform connection

### Core Functionality
- Wallet detection and connection
- Address retrieval
- Balance queries
- Transaction signing
- UTxO management
- Network validation

## CIP-30 Browser Wallets

### Basic Wallet Connection
```javascript
// Check if wallet is available
async function detectWallets() {
  const wallets = [];
  
  if (window.cardano) {
    for (const [key, wallet] of Object.entries(window.cardano)) {
      if (wallet && typeof wallet.enable === 'function') {
        wallets.push({
          name: key,
          displayName: wallet.name || key,
          icon: wallet.icon || null,
          apiVersion: wallet.apiVersion || '0.1.0',
          isEnabled: await wallet.isEnabled().catch(() => false)
        });
      }
    }
  }
  
  return wallets;
}

// Connect to specific wallet
async function connectWallet(walletName) {
  if (!window.cardano || !window.cardano[walletName]) {
    throw new Error(`${walletName} wallet not found`);
  }
  
  const wallet = window.cardano[walletName];
  
  try {
    const api = await wallet.enable();
    
    // Get basic wallet info
    const [networkId, changeAddress, rewardAddresses, utxos] = await Promise.all([
      api.getNetworkId(),
      api.getChangeAddress(),
      api.getRewardAddresses(),
      api.getUtxos()
    ]);
    
    return {
      api,
      networkId,
      changeAddress,
      rewardAddresses,
      utxos
    };
  } catch (error) {
    if (error.code === -1) {
      throw new Error('User rejected wallet connection');
    } else if (error.code === -2) {
      throw new Error('Wallet already connected to another DApp');
    }
    throw error;
  }
}
```

### React Wallet Hook
```javascript
import { useState, useEffect, useCallback } from 'react';

export function useCardanoWallet() {
  const [wallets, setWallets] = useState([]);
  const [connectedWallet, setConnectedWallet] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState(null);

  // Detect available wallets
  useEffect(() => {
    async function detectWallets() {
      const detected = [];
      
      if (window.cardano) {
        for (const [key, wallet] of Object.entries(window.cardano)) {
          if (wallet && typeof wallet.enable === 'function') {
            try {
              const isEnabled = await wallet.isEnabled();
              detected.push({
                key,
                name: wallet.name || key,
                icon: wallet.icon,
                apiVersion: wallet.apiVersion || '0.1.0',
                isEnabled
              });
            } catch (err) {
              console.warn(`Error detecting wallet ${key}:`, err);
            }
          }
        }
      }
      
      setWallets(detected);
    }

    detectWallets();
    
    // Listen for wallet installation
    const interval = setInterval(detectWallets, 1000);
    return () => clearInterval(interval);
  }, []);

  const connect = useCallback(async (walletKey) => {
    setIsConnecting(true);
    setError(null);
    
    try {
      const wallet = window.cardano[walletKey];
      const api = await wallet.enable();
      
      const [networkId, balance, changeAddress, rewardAddresses] = await Promise.all([
        api.getNetworkId(),
        api.getBalance(),
        api.getChangeAddress(),
        api.getRewardAddresses()
      ]);
      
      const walletInfo = {
        key: walletKey,
        name: wallet.name || walletKey,
        api,
        networkId,
        balance,
        changeAddress,
        rewardAddresses
      };
      
      setConnectedWallet(walletInfo);
      
      // Store connection preference
      localStorage.setItem('preferred_wallet', walletKey);
      
      return walletInfo;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setIsConnecting(false);
    }
  }, []);

  const disconnect = useCallback(() => {
    setConnectedWallet(null);
    localStorage.removeItem('preferred_wallet');
  }, []);

  const refreshBalance = useCallback(async () => {
    if (connectedWallet?.api) {
      try {
        const balance = await connectedWallet.api.getBalance();
        setConnectedWallet(prev => ({ ...prev, balance }));
        return balance;
      } catch (err) {
        setError('Failed to refresh balance');
        throw err;
      }
    }
  }, [connectedWallet]);

  // Auto-connect to previously connected wallet
  useEffect(() => {
    const preferredWallet = localStorage.getItem('preferred_wallet');
    if (preferredWallet && wallets.length > 0 && !connectedWallet) {
      const wallet = wallets.find(w => w.key === preferredWallet && w.isEnabled);
      if (wallet) {
        connect(preferredWallet).catch(console.error);
      }
    }
  }, [wallets, connectedWallet, connect]);

  return {
    wallets,
    connectedWallet,
    isConnecting,
    error,
    connect,
    disconnect,
    refreshBalance
  };
}
```

### Wallet Connection Component
```jsx
import React from 'react';
import { useCardanoWallet } from './useCardanoWallet';

export function WalletConnector() {
  const { 
    wallets, 
    connectedWallet, 
    isConnecting, 
    error, 
    connect, 
    disconnect 
  } = useCardanoWallet();

  if (connectedWallet) {
    return (
      <div className="wallet-connected">
        <div className="wallet-info">
          <img src={connectedWallet.icon} alt={connectedWallet.name} />
          <span>{connectedWallet.name}</span>
          <span>Network: {connectedWallet.networkId === 1 ? 'Mainnet' : 'Testnet'}</span>
        </div>
        <button onClick={disconnect}>Disconnect</button>
      </div>
    );
  }

  return (
    <div className="wallet-selection">
      <h3>Connect Wallet</h3>
      
      {error && (
        <div className="error">
          {error}
        </div>
      )}
      
      {wallets.length === 0 ? (
        <div className="no-wallets">
          <p>No Cardano wallets detected.</p>
          <p>Please install a wallet extension:</p>
          <div className="wallet-links">
            <a href="https://namiwallet.io" target="_blank" rel="noopener noreferrer">
              Nami Wallet
            </a>
            <a href="https://eternl.io" target="_blank" rel="noopener noreferrer">
              Eternl
            </a>
            <a href="https://flint-wallet.com" target="_blank" rel="noopener noreferrer">
              Flint Wallet
            </a>
          </div>
        </div>
      ) : (
        <div className="wallet-list">
          {wallets.map((wallet) => (
            <button
              key={wallet.key}
              className="wallet-option"
              onClick={() => connect(wallet.key)}
              disabled={isConnecting}
            >
              {wallet.icon && <img src={wallet.icon} alt={wallet.name} />}
              <span>{wallet.name}</span>
              {wallet.isEnabled && <span className="enabled">✓</span>}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

## Wallet Detection and Selection

### Multi-Wallet Detection
```javascript
class WalletManager {
  constructor() {
    this.wallets = new Map();
    this.connectedWallet = null;
    this.listeners = new Set();
  }

  async detectWallets() {
    const detectedWallets = new Map();
    
    // Known wallet identifiers
    const knownWallets = {
      nami: {
        name: 'Nami',
        icon: 'data:image/svg+xml;base64,...'
      },
      eternl: {
        name: 'Eternl',
        icon: 'data:image/svg+xml;base64,...'
      },
      flint: {
        name: 'Flint',
        icon: 'data:image/svg+xml;base64,...'
      },
      typhoncip30: {
        name: 'Typhon',
        icon: 'data:image/svg+xml;base64,...'
      },
      cardwallet: {
        name: 'CardWallet',
        icon: 'data:image/svg+xml;base64,...'
      },
      gerowallet: {
        name: 'Gero Wallet',
        icon: 'data:image/svg+xml;base64,...'
      }
    };

    if (window.cardano) {
      for (const [key, cardanoWallet] of Object.entries(window.cardano)) {
        if (cardanoWallet && typeof cardanoWallet.enable === 'function') {
          try {
            const walletInfo = {
              key,
              name: cardanoWallet.name || knownWallets[key]?.name || key,
              icon: cardanoWallet.icon || knownWallets[key]?.icon,
              apiVersion: cardanoWallet.apiVersion || '0.1.0',
              supportedExtensions: cardanoWallet.supportedExtensions || [],
              isEnabled: await cardanoWallet.isEnabled().catch(() => false),
              wallet: cardanoWallet
            };
            
            detectedWallets.set(key, walletInfo);
          } catch (error) {
            console.warn(`Error detecting wallet ${key}:`, error);
          }
        }
      }
    }

    this.wallets = detectedWallets;
    this.notifyListeners('walletsDetected', Array.from(detectedWallets.values()));
    
    return detectedWallets;
  }

  async connectWallet(walletKey, options = {}) {
    const walletInfo = this.wallets.get(walletKey);
    if (!walletInfo) {
      throw new Error(`Wallet ${walletKey} not found`);
    }

    try {
      // Enable wallet with optional extensions
      const extensions = options.extensions || [];
      const api = await walletInfo.wallet.enable(extensions);
      
      // Validate API version compatibility
      if (api.apiVersion && !this.isVersionCompatible(api.apiVersion)) {
        console.warn(`Wallet API version ${api.apiVersion} may not be fully compatible`);
      }

      // Get wallet details
      const [networkId, balance, changeAddress, rewardAddresses, utxos] = await Promise.allSettled([
        api.getNetworkId(),
        api.getBalance(),
        api.getChangeAddress(),
        api.getRewardAddresses(),
        api.getUtxos()
      ]);

      const connectedWalletInfo = {
        ...walletInfo,
        api,
        networkId: networkId.status === 'fulfilled' ? networkId.value : null,
        balance: balance.status === 'fulfilled' ? balance.value : '0',
        changeAddress: changeAddress.status === 'fulfilled' ? changeAddress.value : null,
        rewardAddresses: rewardAddresses.status === 'fulfilled' ? rewardAddresses.value : [],
        utxos: utxos.status === 'fulfilled' ? utxos.value : [],
        connectedAt: new Date()
      };

      this.connectedWallet = connectedWalletInfo;
      this.notifyListeners('walletConnected', connectedWalletInfo);
      
      return connectedWalletInfo;
    } catch (error) {
      this.handleConnectionError(error);
      throw error;
    }
  }

  handleConnectionError(error) {
    let userMessage = 'Failed to connect wallet';
    
    if (error.code === -1) {
      userMessage = 'Connection rejected by user';
    } else if (error.code === -2) {
      userMessage = 'Wallet is already connected to another application';
    } else if (error.message?.includes('User declined')) {
      userMessage = 'Connection declined by user';
    }
    
    this.notifyListeners('connectionError', { error, userMessage });
  }

  isVersionCompatible(version) {
    const [major, minor] = version.split('.').map(Number);
    // Support API versions 0.1.0 and above
    return major >= 0 && (major > 0 || minor >= 1);
  }

  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  notifyListeners(event, data) {
    this.listeners.forEach(listener => {
      try {
        listener(event, data);
      } catch (error) {
        console.error('Error in wallet listener:', error);
      }
    });
  }
}
```

## Transaction Signing

### Transaction Signing Process
```javascript
async function signAndSubmitTransaction(walletApi, transaction) {
  try {
    // Convert transaction to CBOR if needed
    const txCbor = typeof transaction === 'string' 
      ? transaction 
      : transaction.toCBOR();
    
    // Sign transaction
    const witnessSet = await walletApi.signTx(txCbor, true); // partialSign = true
    
    // Assemble final transaction
    const signedTx = await transaction.assemble([witnessSet]);
    
    // Submit to network
    const txHash = await walletApi.submitTx(signedTx.toCBOR());
    
    return {
      txHash,
      signedTx
    };
  } catch (error) {
    handleSigningError(error);
    throw error;
  }
}

function handleSigningError(error) {
  if (error.code === -1) {
    throw new Error('Transaction signing rejected by user');
  } else if (error.code === -2) {
    throw new Error('Transaction signing failed due to proof generation error');
  } else if (error.message?.includes('User declined')) {
    throw new Error('Transaction declined by user');
  } else if (error.message?.includes('Insufficient funds')) {
    throw new Error('Insufficient funds for transaction');
  }
  
  // Re-throw original error if not handled
  throw error;
}
```

### Batch Transaction Signing
```javascript
async function signMultipleTransactions(walletApi, transactions) {
  const results = [];
  
  for (let i = 0; i < transactions.length; i++) {
    try {
      console.log(`Signing transaction ${i + 1} of ${transactions.length}`);
      
      const result = await signAndSubmitTransaction(walletApi, transactions[i]);
      results.push({
        index: i,
        success: true,
        txHash: result.txHash
      });
      
      // Wait between transactions to avoid overwhelming the user
      if (i < transactions.length - 1) {
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    } catch (error) {
      results.push({
        index: i,
        success: false,
        error: error.message
      });
      
      // Ask user if they want to continue with remaining transactions
      const continueWithRest = confirm(
        `Transaction ${i + 1} failed: ${error.message}\n\nContinue with remaining transactions?`
      );
      
      if (!continueWithRest) {
        break;
      }
    }
  }
  
  return results;
}
```

### Message Signing (CIP-8)
```javascript
async function signMessage(walletApi, message, address) {
  try {
    // Create message payload
    const payload = {
      address,
      payload: Buffer.from(message, 'utf8').toString('hex')
    };
    
    // Sign the message
    const signature = await walletApi.signData(address, JSON.stringify(payload));
    
    return {
      signature: signature.signature,
      key: signature.key,
      message,
      address
    };
  } catch (error) {
    if (error.code === -1) {
      throw new Error('Message signing rejected by user');
    }
    throw error;
  }
}

// Verify message signature
function verifyMessageSignature(signedMessage) {
  try {
    const { signature, key, message, address } = signedMessage;
    
    // Implementation would depend on cryptographic library
    // This is a simplified example
    const publicKey = CardanoWasm.PublicKey.from_bytes(Buffer.from(key, 'hex'));
    const messageBytes = Buffer.from(message, 'utf8');
    const signatureBytes = Buffer.from(signature, 'hex');
    
    return publicKey.verify(messageBytes, CardanoWasm.Ed25519Signature.from_bytes(signatureBytes));
  } catch (error) {
    console.error('Signature verification failed:', error);
    return false;
  }
}
```

## State Management

### Redux Wallet State
```javascript
// walletSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Async thunks
export const detectWallets = createAsyncThunk(
  'wallet/detectWallets',
  async () => {
    const walletManager = new WalletManager();
    return await walletManager.detectWallets();
  }
);

export const connectWallet = createAsyncThunk(
  'wallet/connectWallet',
  async (walletKey, { rejectWithValue }) => {
    try {
      const walletManager = new WalletManager();
      return await walletManager.connectWallet(walletKey);
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

export const refreshWalletData = createAsyncThunk(
  'wallet/refreshWalletData',
  async (_, { getState }) => {
    const { wallet } = getState();
    if (!wallet.connected?.api) {
      throw new Error('No wallet connected');
    }
    
    const api = wallet.connected.api;
    const [balance, utxos] = await Promise.all([
      api.getBalance(),
      api.getUtxos()
    ]);
    
    return { balance, utxos };
  }
);

const walletSlice = createSlice({
  name: 'wallet',
  initialState: {
    available: [],
    connected: null,
    isConnecting: false,
    isRefreshing: false,
    error: null
  },
  reducers: {
    disconnectWallet: (state) => {
      state.connected = null;
      state.error = null;
    },
    clearError: (state) => {
      state.error = null;
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(detectWallets.fulfilled, (state, action) => {
        state.available = action.payload;
      })
      .addCase(connectWallet.pending, (state) => {
        state.isConnecting = true;
        state.error = null;
      })
      .addCase(connectWallet.fulfilled, (state, action) => {
        state.isConnecting = false;
        state.connected = action.payload;
        state.error = null;
      })
      .addCase(connectWallet.rejected, (state, action) => {
        state.isConnecting = false;
        state.error = action.payload;
      })
      .addCase(refreshWalletData.pending, (state) => {
        state.isRefreshing = true;
      })
      .addCase(refreshWalletData.fulfilled, (state, action) => {
        state.isRefreshing = false;
        if (state.connected) {
          state.connected.balance = action.payload.balance;
          state.connected.utxos = action.payload.utxos;
        }
      })
      .addCase(refreshWalletData.rejected, (state, action) => {
        state.isRefreshing = false;
        state.error = action.error.message;
      });
  }
});

export const { disconnectWallet, clearError } = walletSlice.actions;
export default walletSlice.reducer;
```

### Context API Wallet Provider
```jsx
import React, { createContext, useContext, useReducer, useEffect } from 'react';

const WalletContext = createContext();

const initialState = {
  availableWallets: [],
  connectedWallet: null,
  isConnecting: false,
  error: null
};

function walletReducer(state, action) {
  switch (action.type) {
    case 'SET_AVAILABLE_WALLETS':
      return { ...state, availableWallets: action.payload };
    
    case 'CONNECT_START':
      return { ...state, isConnecting: true, error: null };
    
    case 'CONNECT_SUCCESS':
      return { 
        ...state, 
        isConnecting: false, 
        connectedWallet: action.payload,
        error: null 
      };
    
    case 'CONNECT_ERROR':
      return { 
        ...state, 
        isConnecting: false, 
        error: action.payload 
      };
    
    case 'DISCONNECT':
      return { 
        ...state, 
        connectedWallet: null,
        error: null 
      };
    
    case 'UPDATE_BALANCE':
      return {
        ...state,
        connectedWallet: state.connectedWallet 
          ? { ...state.connectedWallet, balance: action.payload }
          : null
      };
    
    default:
      return state;
  }
}

export function WalletProvider({ children }) {
  const [state, dispatch] = useReducer(walletReducer, initialState);

  const detectWallets = async () => {
    const walletManager = new WalletManager();
    const wallets = await walletManager.detectWallets();
    dispatch({ type: 'SET_AVAILABLE_WALLETS', payload: Array.from(wallets.values()) });
  };

  const connectWallet = async (walletKey) => {
    dispatch({ type: 'CONNECT_START' });
    
    try {
      const walletManager = new WalletManager();
      const wallet = await walletManager.connectWallet(walletKey);
      dispatch({ type: 'CONNECT_SUCCESS', payload: wallet });
      
      // Save preference
      localStorage.setItem('preferred_wallet', walletKey);
      
      return wallet;
    } catch (error) {
      dispatch({ type: 'CONNECT_ERROR', payload: error.message });
      throw error;
    }
  };

  const disconnectWallet = () => {
    dispatch({ type: 'DISCONNECT' });
    localStorage.removeItem('preferred_wallet');
  };

  const refreshBalance = async () => {
    if (state.connectedWallet?.api) {
      try {
        const balance = await state.connectedWallet.api.getBalance();
        dispatch({ type: 'UPDATE_BALANCE', payload: balance });
      } catch (error) {
        console.error('Failed to refresh balance:', error);
      }
    }
  };

  // Auto-detect wallets on mount
  useEffect(() => {
    detectWallets();
    
    const interval = setInterval(detectWallets, 5000);
    return () => clearInterval(interval);
  }, []);

  // Auto-connect to preferred wallet
  useEffect(() => {
    const preferredWallet = localStorage.getItem('preferred_wallet');
    if (preferredWallet && state.availableWallets.length > 0 && !state.connectedWallet) {
      const wallet = state.availableWallets.find(w => w.key === preferredWallet && w.isEnabled);
      if (wallet) {
        connectWallet(preferredWallet).catch(console.error);
      }
    }
  }, [state.availableWallets]);

  const value = {
    ...state,
    connectWallet,
    disconnectWallet,
    refreshBalance,
    detectWallets
  };

  return (
    <WalletContext.Provider value={value}>
      {children}
    </WalletContext.Provider>
  );
}

export function useWallet() {
  const context = useContext(WalletContext);
  if (!context) {
    throw new Error('useWallet must be used within a WalletProvider');
  }
  return context;
}
```

## Error Handling

### Comprehensive Error Handling
```javascript
class WalletError extends Error {
  constructor(message, code, originalError = null) {
    super(message);
    this.name = 'WalletError';
    this.code = code;
    this.originalError = originalError;
  }
}

const WalletErrorCodes = {
  WALLET_NOT_FOUND: 'WALLET_NOT_FOUND',
  CONNECTION_REJECTED: 'CONNECTION_REJECTED',
  ALREADY_CONNECTED: 'ALREADY_CONNECTED',
  SIGNING_REJECTED: 'SIGNING_REJECTED',
  INSUFFICIENT_FUNDS: 'INSUFFICIENT_FUNDS',
  NETWORK_ERROR: 'NETWORK_ERROR',
  INVALID_TRANSACTION: 'INVALID_TRANSACTION',
  UNKNOWN_ERROR: 'UNKNOWN_ERROR'
};

function handleWalletError(error) {
  if (error instanceof WalletError) {
    return error;
  }

  // Map common CIP-30 error codes
  switch (error.code) {
    case -1:
      return new WalletError(
        'User rejected the request',
        WalletErrorCodes.CONNECTION_REJECTED,
        error
      );
    case -2:
      return new WalletError(
        'Wallet is already connected elsewhere',
        WalletErrorCodes.ALREADY_CONNECTED,
        error
      );
    default:
      break;
  }

  // Map common error messages
  if (error.message?.includes('User declined')) {
    return new WalletError(
      'User declined the request',
      WalletErrorCodes.CONNECTION_REJECTED,
      error
    );
  }
  
  if (error.message?.includes('Insufficient funds')) {
    return new WalletError(
      'Insufficient funds for transaction',
      WalletErrorCodes.INSUFFICIENT_FUNDS,
      error
    );
  }
  
  if (error.message?.includes('Network')) {
    return new WalletError(
      'Network connection error',
      WalletErrorCodes.NETWORK_ERROR,
      error
    );
  }

  return new WalletError(
    error.message || 'Unknown wallet error',
    WalletErrorCodes.UNKNOWN_ERROR,
    error
  );
}

// Error boundary component
class WalletErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error: handleWalletError(error) };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Wallet error boundary caught an error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="wallet-error">
          <h3>Wallet Error</h3>
          <p>{this.state.error.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try Again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

## Security Considerations

### Security Best Practices
```javascript
// Validate network ID matches expected network
function validateNetwork(walletNetworkId, expectedNetworkId) {
  if (walletNetworkId !== expectedNetworkId) {
    throw new WalletError(
      `Network mismatch. Expected ${expectedNetworkId}, got ${walletNetworkId}`,
      WalletErrorCodes.NETWORK_ERROR
    );
  }
}

// Validate transaction before signing
function validateTransaction(transaction, limits = {}) {
  const errors = [];
  
  // Check transaction size
  if (limits.maxSize && transaction.body.length > limits.maxSize) {
    errors.push(`Transaction too large: ${transaction.body.length} > ${limits.maxSize}`);
  }
  
  // Check output amounts
  if (limits.maxOutputValue) {
    for (const output of transaction.body.outputs) {
      if (BigInt(output.amount.coin) > BigInt(limits.maxOutputValue)) {
        errors.push(`Output value too large: ${output.amount.coin}`);
      }
    }
  }
  
  // Check fee reasonableness
  if (limits.maxFee && BigInt(transaction.body.fee) > BigInt(limits.maxFee)) {
    errors.push(`Fee too high: ${transaction.body.fee}`);
  }
  
  if (errors.length > 0) {
    throw new WalletError(
      `Transaction validation failed: ${errors.join(', ')}`,
      WalletErrorCodes.INVALID_TRANSACTION
    );
  }
}

// Rate limiting for wallet operations
class WalletRateLimiter {
  constructor(maxRequests = 10, windowMs = 60000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = new Map();
  }
  
  checkRateLimit(walletAddress) {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    if (!this.requests.has(walletAddress)) {
      this.requests.set(walletAddress, []);
    }
    
    const userRequests = this.requests.get(walletAddress);
    
    // Remove old requests
    const recentRequests = userRequests.filter(timestamp => timestamp > windowStart);
    
    if (recentRequests.length >= this.maxRequests) {
      throw new Error('Rate limit exceeded. Please try again later.');
    }
    
    recentRequests.push(now);
    this.requests.set(walletAddress, recentRequests);
  }
}
```

## Popular Wallets

### Wallet-Specific Integrations

#### Nami Wallet
```javascript
async function connectNami() {
  if (!window.cardano?.nami) {
    throw new WalletError('Nami wallet not found', WalletErrorCodes.WALLET_NOT_FOUND);
  }
  
  const api = await window.cardano.nami.enable();
  
  // Nami-specific features
  const walletInfo = {
    name: 'Nami',
    api,
    features: ['basic', 'signing', 'multi-asset'],
    version: api.apiVersion
  };
  
  return walletInfo;
}
```

#### Eternl Wallet
```javascript
async function connectEternl() {
  if (!window.cardano?.eternl) {
    throw new WalletError('Eternl wallet not found', WalletErrorCodes.WALLET_NOT_FOUND);
  }
  
  const api = await window.cardano.eternl.enable();
  
  // Eternl-specific features
  const walletInfo = {
    name: 'Eternl',
    api,
    features: ['basic', 'signing', 'multi-asset', 'multi-sig'],
    version: api.apiVersion
  };
  
  return walletInfo;
}
```

#### Universal Wallet Connector
```javascript
const WALLET_CONFIGS = {
  nami: {
    name: 'Nami',
    downloadUrl: 'https://namiwallet.io',
    features: ['basic', 'signing']
  },
  eternl: {
    name: 'Eternl', 
    downloadUrl: 'https://eternl.io',
    features: ['basic', 'signing', 'multi-sig']
  },
  flint: {
    name: 'Flint',
    downloadUrl: 'https://flint-wallet.com',
    features: ['basic', 'signing']
  },
  typhoncip30: {
    name: 'Typhon',
    downloadUrl: 'https://typhonwallet.io',
    features: ['basic', 'signing', 'multi-asset']
  }
};

class UniversalWalletConnector {
  constructor() {
    this.supportedWallets = Object.keys(WALLET_CONFIGS);
  }
  
  async detectAvailableWallets() {
    const available = [];
    
    for (const walletKey of this.supportedWallets) {
      if (window.cardano?.[walletKey]) {
        const config = WALLET_CONFIGS[walletKey];
        const wallet = window.cardano[walletKey];
        
        try {
          const isEnabled = await wallet.isEnabled();
          available.push({
            key: walletKey,
            name: config.name,
            icon: wallet.icon,
            features: config.features,
            isEnabled,
            downloadUrl: config.downloadUrl
          });
        } catch (error) {
          console.warn(`Error checking wallet ${walletKey}:`, error);
        }
      }
    }
    
    return available;
  }
  
  async connectToWallet(walletKey) {
    const config = WALLET_CONFIGS[walletKey];
    if (!config) {
      throw new WalletError(`Unsupported wallet: ${walletKey}`, WalletErrorCodes.WALLET_NOT_FOUND);
    }
    
    const wallet = window.cardano?.[walletKey];
    if (!wallet) {
      throw new WalletError(`${config.name} not installed`, WalletErrorCodes.WALLET_NOT_FOUND);
    }
    
    const api = await wallet.enable();
    
    return {
      key: walletKey,
      name: config.name,
      api,
      features: config.features
    };
  }
}
```

---

**Next Steps**:
- Review [Transaction Building](../transaction-building/README.md) for wallet transaction integration
- Check [Metadata Standards](../metadata-standards/README.md) for CIP-30 compliance
- See [Best Practices](../best-practices/README.md) for wallet security guidelines
- Explore [Testing](../testing/README.md) for wallet integration testing