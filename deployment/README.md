# Cardano Deployment Guide

Comprehensive guide for deploying Cardano applications, smart contracts, and infrastructure to production.

## Table of Contents

- [Deployment Overview](#deployment-overview)
- [Smart Contract Deployment](#smart-contract-deployment)
- [DApp Deployment](#dapp-deployment)
- [Infrastructure Deployment](#infrastructure-deployment)
- [CI/CD Pipelines](#cicd-pipelines)
- [Monitoring and Observability](#monitoring-and-observability)
- [Security Considerations](#security-considerations)
- [Production Checklist](#production-checklist)

## Deployment Overview

### Deployment Environments
- **Local Development**: Testing and development
- **Testnet**: Integration and staging testing
- **Mainnet**: Production deployment

### Deployment Types
- **Smart Contracts**: On-chain contract deployment
- **DApps**: Frontend and backend services
- **Infrastructure**: Nodes, indexers, APIs
- **Wallets**: Integration and configuration

## Smart Contract Deployment

### Plutus Contract Deployment

#### Preparation Steps
```bash
# 1. Compile and test contract locally
cabal build all
cabal test

# 2. Generate contract artifacts
cabal run plutus-project -- write-script \
  --validator my-validator \
  --out-file validator.plutus

# 3. Calculate script address
cardano-cli address build \
  --payment-script-file validator.plutus \
  --testnet-magic 2 \
  --out-file validator.addr

# 4. Generate script hash
cardano-cli transaction policyid \
  --script-file validator.plutus > validator.hash
```

#### Testnet Deployment
```bash
# 1. Deploy reference script (optional but recommended)
cardano-cli transaction build \
  --tx-in $FUNDING_UTXO \
  --tx-out "$(cat validator.addr)+5000000" \
  --tx-out-reference-script-file validator.plutus \
  --change-address $DEPLOYER_ADDRESS \
  --testnet-magic 2 \
  --out-file deploy-reference.raw

cardano-cli transaction sign \
  --tx-body-file deploy-reference.raw \
  --signing-key-file deployer.skey \
  --testnet-magic 2 \
  --out-file deploy-reference.signed

cardano-cli transaction submit \
  --tx-file deploy-reference.signed \
  --testnet-magic 2

# 2. Test contract interaction
# Lock funds
cardano-cli transaction build \
  --tx-in $USER_UTXO \
  --tx-out "$(cat validator.addr)+10000000" \
  --tx-out-datum-hash $(cat datum.hash) \
  --change-address $USER_ADDRESS \
  --testnet-magic 2 \
  --out-file lock.raw

# Unlock funds (test validation)
cardano-cli transaction build \
  --tx-in $SCRIPT_UTXO \
  --tx-in-script-file validator.plutus \
  --tx-in-datum-file datum.json \
  --tx-in-redeemer-file redeemer.json \
  --tx-out "$USER_ADDRESS+9500000" \
  --change-address $USER_ADDRESS \
  --testnet-magic 2 \
  --out-file unlock.raw
```

#### Mainnet Deployment Process
```bash
# 1. Final security review
# - Code audit completed
# - Formal verification (if applicable)
# - Extensive testing on testnet

# 2. Mainnet deployment with minimal exposure
# Deploy reference script with small amount
cardano-cli transaction build \
  --tx-in $FUNDING_UTXO \
  --tx-out "$(cat validator.addr)+5000000" \
  --tx-out-reference-script-file validator.plutus \
  --change-address $DEPLOYER_ADDRESS \
  --mainnet \
  --out-file deploy-mainnet.raw

# 3. Gradual rollout
# Start with small transactions and limited user base
# Monitor for issues before full deployment
```

### Aiken Contract Deployment

#### Build and Deploy Process
```bash
# 1. Build and test
aiken build
aiken test

# 2. Generate deployment artifacts
aiken blueprint convert > plutus.json

# 3. Extract validator CBOR
cat plutus.json | jq -r '.validators[0].compiledCode' > validator.cbor

# 4. Create Plutus script file
echo "{\"type\": \"PlutusScriptV2\", \"cborHex\": \"$(cat validator.cbor)\"}" > validator.plutus

# 5. Deploy using standard cardano-cli process
cardano-cli address build \
  --payment-script-file validator.plutus \
  --testnet-magic 2 \
  --out-file validator.addr
```

### Contract Versioning and Upgrades

#### Version Management Strategy
```javascript
// Implement contract versioning
const CONTRACT_VERSIONS = {
  'v1.0.0': {
    scriptHash: 'script_hash_v1',
    address: 'addr1_v1...',
    deprecationDate: '2024-06-01'
  },
  'v1.1.0': {
    scriptHash: 'script_hash_v1_1',
    address: 'addr1_v1_1...',
    active: true
  }
};

class ContractManager {
  constructor(version = 'latest') {
    this.version = version === 'latest' 
      ? this.getLatestVersion() 
      : version;
    this.contract = CONTRACT_VERSIONS[this.version];
  }
  
  getLatestVersion() {
    return Object.keys(CONTRACT_VERSIONS)
      .filter(v => CONTRACT_VERSIONS[v].active)
      .sort()
      .pop();
  }
  
  isDeprecated() {
    const contract = CONTRACT_VERSIONS[this.version];
    return contract.deprecationDate && 
           new Date() > new Date(contract.deprecationDate);
  }
  
  migrate(fromVersion, toVersion) {
    // Implementation for contract migration
    // May involve moving funds from old to new contract
  }
}
```

#### Upgrade Strategies
```rust
// Aiken: Upgradeable contract pattern
type UpgradeToken = ByteArray

validator upgrade_proxy {
  fn spend(
    datum: UpgradeDatum, 
    redeemer: UpgradeRedeemer, 
    context: ScriptContext
  ) -> Bool {
    when redeemer is {
      // Delegate to current implementation
      Delegate(impl_redeemer) -> {
        // Validate upgrade token is present
        let has_upgrade_token = 
          value.quantity_of(context.transaction.mint, upgrade_policy, upgrade_token) > 0
        
        if has_upgrade_token {
          // Call new implementation
          new_validator(datum.user_datum, impl_redeemer, context)
        } else {
          // Call current implementation  
          current_validator(datum.user_datum, impl_redeemer, context)
        }
      }
      
      // Admin can upgrade
      Upgrade(new_hash) -> {
        // Verify admin signature and update implementation hash
        check_admin_signature(context) && update_implementation(new_hash)
      }
    }
  }
}
```

## DApp Deployment

### Frontend Deployment

#### Static Site Deployment (Netlify/Vercel)
```yaml
# netlify.toml
[build]
  publish = "dist"
  command = "npm run build"

[build.environment]
  NODE_VERSION = "18"
  NPM_VERSION = "8"

[[redirects]]
  from = "/api/*"
  to = "https://api.myapp.com/api/:splat"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    Content-Security-Policy = "default-src 'self'; script-src 'self' 'unsafe-inline'"
```

```javascript
// Environment-specific configuration
const config = {
  development: {
    CARDANO_NETWORK: 'Preview',
    BLOCKFROST_URL: 'https://cardano-preview.blockfrost.io/api/v0',
    BLOCKFROST_PROJECT_ID: process.env.BLOCKFROST_PREVIEW_ID
  },
  production: {
    CARDANO_NETWORK: 'Mainnet',
    BLOCKFROST_URL: 'https://cardano-mainnet.blockfrost.io/api/v0', 
    BLOCKFROST_PROJECT_ID: process.env.BLOCKFROST_MAINNET_ID
  }
};

export default config[process.env.NODE_ENV || 'development'];
```

#### Docker Deployment
```dockerfile
# Frontend Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Handle SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass https://api.myapp.com/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
}
```

### Backend API Deployment

#### Node.js API Deployment
```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - CARDANO_NETWORK=${CARDANO_NETWORK}
      - BLOCKFROST_PROJECT_ID=${BLOCKFROST_PROJECT_ID}
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - postgres
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: cardano_app
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

```dockerfile
# API Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S cardano -u 1001

USER cardano

EXPOSE 3000

CMD ["node", "server.js"]
```

#### Production API Configuration
```javascript
// server.js
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const cors = require('cors');

const app = express();

// Security middleware
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests, please try again later.',
});
app.use('/api/', limiter);

// CORS configuration
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
}));

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0'
  });
});

app.listen(3000, '0.0.0.0', () => {
  console.log('Server running on port 3000');
});
```

## Infrastructure Deployment

### Cardano Node Deployment

#### Production Node Setup
```bash
#!/bin/bash
# deploy-node.sh

# System requirements check
check_requirements() {
    # Check RAM (minimum 8GB)
    RAM=$(free -g | awk '/^Mem:/{print $2}')
    if [ $RAM -lt 8 ]; then
        echo "Error: Insufficient RAM. Minimum 8GB required."
        exit 1
    fi
    
    # Check disk space (minimum 150GB for mainnet)
    DISK=$(df -h / | awk 'NR==2{print $4}' | sed 's/G//')
    if [ $DISK -lt 150 ]; then
        echo "Error: Insufficient disk space. Minimum 150GB required."
        exit 1
    fi
}

# Install Cardano Node
install_node() {
    # Download and install cardano-node
    curl -sLJ https://github.com/input-output-hk/cardano-node/releases/latest/download/cardano-node-linux.tar.gz \
        -o cardano-node.tar.gz
    
    tar -xzf cardano-node.tar.gz
    sudo mv cardano-* /usr/local/bin/
    
    # Verify installation
    cardano-node --version
    cardano-cli --version
}

# Configure node
configure_node() {
    # Create directory structure
    mkdir -p ~/cardano/{config,db,logs,scripts}
    cd ~/cardano/config
    
    # Download mainnet configuration
    curl -O https://book.world.dev.cardano.org/environments/mainnet/config.json
    curl -O https://book.world.dev.cardano.org/environments/mainnet/topology.json
    curl -O https://book.world.dev.cardano.org/environments/mainnet/byron-genesis.json
    curl -O https://book.world.dev.cardano.org/environments/mainnet/shelley-genesis.json
    curl -O https://book.world.dev.cardano.org/environments/mainnet/alonzo-genesis.json
    curl -O https://book.world.dev.cardano.org/environments/mainnet/conway-genesis.json
    
    # Update configuration for production
    jq '.TraceBlockFetchDecisions = false | .TracingVerbosity = "MinimalVerbosity"' config.json > config-prod.json
}

# Create systemd service
create_service() {
    sudo tee /etc/systemd/system/cardano-node.service > /dev/null <<EOF
[Unit]
Description=Cardano Node
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=10
User=cardano
WorkingDirectory=/home/cardano/cardano
ExecStart=/usr/local/bin/cardano-node run \\
    --config config/config-prod.json \\
    --topology config/topology.json \\
    --database-path db \\
    --socket-path db/socket \\
    --port 3001
StandardOutput=journal
StandardError=journal
SyslogIdentifier=cardano-node

[Install]
WantedBy=multi-user.target
EOF

    # Enable and start service
    sudo systemctl daemon-reload
    sudo systemctl enable cardano-node
    sudo systemctl start cardano-node
}

# Main deployment
main() {
    check_requirements
    install_node
    configure_node
    create_service
    
    echo "Cardano node deployment completed!"
    echo "Check status with: sudo systemctl status cardano-node"
    echo "View logs with: journalctl -u cardano-node -f"
}

main "$@"
```

#### Monitoring Node Health
```bash
#!/bin/bash
# monitor-node.sh

check_node_health() {
    local socket_path="$HOME/cardano/db/socket"
    
    # Check if node is running
    if ! pgrep -f cardano-node > /dev/null; then
        echo "ERROR: Cardano node is not running"
        return 1
    fi
    
    # Check socket file exists
    if [ ! -S "$socket_path" ]; then
        echo "ERROR: Socket file not found at $socket_path"
        return 1
    fi
    
    # Query node tip
    local tip_output
    tip_output=$(cardano-cli query tip --mainnet 2>&1)
    
    if [ $? -eq 0 ]; then
        echo "Node is healthy:"
        echo "$tip_output"
        return 0
    else
        echo "ERROR: Unable to query node tip"
        echo "$tip_output"
        return 1
    fi
}

# Run health check
check_node_health
```

### Database and Indexing

#### Cardano DB Sync Deployment
```yaml
# docker-compose.db-sync.yml
version: '3.8'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: cardano
      POSTGRES_USER: cardano
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgresql.conf:/etc/postgresql/postgresql.conf
    ports:
      - "5432:5432"
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

  cardano-db-sync:
    image: inputoutput/cardano-db-sync:latest
    depends_on:
      - postgres
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_DB: cardano
      POSTGRES_USER: cardano
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ./config:/config
      - ./state:/state
      - cardano_node_socket:/node-socket
    command: >
      cardano-db-sync
      --config /config/config.json
      --socket-path /node-socket/socket
      --state-dir /state
      --schema-dir /schema

volumes:
  postgres_data:
  cardano_node_socket:
    external: true
```

```sql
-- Optimize PostgreSQL for Cardano DB Sync
-- postgresql.conf optimizations

# Memory settings
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 256MB
maintenance_work_mem = 1GB

# Connection settings
max_connections = 100
max_worker_processes = 8
max_parallel_workers = 8
max_parallel_workers_per_gather = 4

# Write ahead log
wal_buffers = 64MB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

# Query planner
random_page_cost = 1.1
effective_io_concurrency = 200

# Logging
log_statement = 'none'
log_duration = off
log_min_duration_statement = 1000
```

## CI/CD Pipelines

### GitHub Actions Pipeline
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build contracts
        run: |
          if [ -f "aiken.toml" ]; then
            aiken build
          fi
      
      - name: Run security audit
        run: npm audit --audit-level moderate

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          # Update staging deployment
          kubectl set image deployment/cardano-app \
            cardano-app=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --namespace=staging

  deploy-production:
    needs: [build-and-push, deploy-staging]
    runs-on: ubuntu-latest
    environment: production
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - name: Deploy to production
        run: |
          # Update production deployment
          kubectl set image deployment/cardano-app \
            cardano-app=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --namespace=production
          
          # Wait for rollout
          kubectl rollout status deployment/cardano-app --namespace=production
```

### Smart Contract Deployment Pipeline
```yaml
# .github/workflows/deploy-contracts.yml
name: Deploy Smart Contracts

on:
  push:
    paths: ['contracts/**']
    branches: [main]

jobs:
  deploy-contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Aiken
        run: |
          curl -sSfL https://install.aiken-lang.org | bash
          echo "$HOME/.aiken/bin" >> $GITHUB_PATH
      
      - name: Build contracts
        run: |
          cd contracts
          aiken build
          aiken test
      
      - name: Generate deployment artifacts
        run: |
          cd contracts
          aiken blueprint convert > plutus.json
          cat plutus.json | jq -r '.validators[0].compiledCode' > validator.cbor
      
      - name: Deploy to testnet
        env:
          CARDANO_TESTNET_MAGIC: 2
          DEPLOYER_KEY: ${{ secrets.DEPLOYER_TESTNET_KEY }}
        run: |
          # Generate script address
          echo "{\"type\": \"PlutusScriptV2\", \"cborHex\": \"$(cat contracts/validator.cbor)\"}" > validator.plutus
          cardano-cli address build \
            --payment-script-file validator.plutus \
            --testnet-magic $CARDANO_TESTNET_MAGIC \
            --out-file validator.addr
          
          # Deploy reference script
          ./scripts/deploy-reference-script.sh
      
      - name: Deploy to mainnet
        if: startsWith(github.ref, 'refs/tags/v')
        env:
          DEPLOYER_KEY: ${{ secrets.DEPLOYER_MAINNET_KEY }}
        run: |
          # Deploy to mainnet with additional safety checks
          ./scripts/deploy-mainnet.sh
```

## Monitoring and Observability

### Application Monitoring
```javascript
// monitoring.js
const prometheus = require('prom-client');
const express = require('express');

// Create metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});

const cardanoTransactionCount = new prometheus.Counter({
  name: 'cardano_transactions_total',
  help: 'Total number of Cardano transactions processed',
  labelNames: ['status', 'network']
});

const walletConnectionsGauge = new prometheus.Gauge({
  name: 'wallet_connections_active',
  help: 'Number of active wallet connections'
});

// Middleware to collect metrics
function metricsMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
  });
  
  next();
}

// Custom metrics
class MetricsCollector {
  static recordTransaction(status, network) {
    cardanoTransactionCount.labels(status, network).inc();
  }
  
  static setActiveConnections(count) {
    walletConnectionsGauge.set(count);
  }
  
  static async collectNodeMetrics() {
    try {
      // Query node for metrics
      const tip = await queryNodeTip();
      
      // Update custom metrics
      const blockHeight = new prometheus.Gauge({
        name: 'cardano_block_height',
        help: 'Current block height'
      });
      
      blockHeight.set(tip.block);
    } catch (error) {
      console.error('Failed to collect node metrics:', error);
    }
  }
}

module.exports = {
  metricsMiddleware,
  MetricsCollector,
  register: prometheus.register
};
```

### Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "id": null,
    "title": "Cardano DApp Monitoring",
    "tags": ["cardano", "dapp"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Transaction Volume",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(cardano_transactions_total[5m])",
            "legendFormat": "Transactions/sec"
          }
        ]
      },
      {
        "id": 2,
        "title": "Active Wallet Connections",
        "type": "singlestat",
        "targets": [
          {
            "expr": "wallet_connections_active",
            "legendFormat": "Active Connections"
          }
        ]
      },
      {
        "id": 3,
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ]
      }
    ]
  }
}
```

### Log Aggregation
```yaml
# docker-compose.logging.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.14.0
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:7.14.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
```

```conf
# logstash.conf
input {
  beats {
    port => 5044
  }
  
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  if [fields][service] == "cardano-node" {
    grok {
      match => { 
        "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{WORD:level} %{GREEDYDATA:log_message}" 
      }
    }
  }
  
  if [fields][service] == "cardano-app" {
    json {
      source => "message"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "cardano-logs-%{+YYYY.MM.dd}"
  }
}
```

## Security Considerations

### Infrastructure Security
```bash
#!/bin/bash
# security-hardening.sh

# System hardening
harden_system() {
    # Update system
    sudo apt update && sudo apt upgrade -y
    
    # Configure firewall
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw allow 3001/tcp  # Cardano node
    sudo ufw allow 443/tcp   # HTTPS
    sudo ufw enable
    
    # Disable root login
    sudo passwd -l root
    
    # Configure SSH
    sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
    sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
    sudo systemctl restart ssh
}

# Secure secrets management
setup_secrets() {
    # Install age for secret encryption
    curl -L https://github.com/FiloSottile/age/releases/latest/download/age-linux-amd64.tar.gz | tar xz
    sudo mv age/age /usr/local/bin/
    
    # Generate key pair
    age-keygen -o ~/.age/key.txt
    chmod 600 ~/.age/key.txt
    
    # Encrypt sensitive files
    age -r $(age-keygen -y ~/.age/key.txt) payment.skey > payment.skey.age
    rm payment.skey
}

# Network security
configure_network_security() {
    # Install fail2ban
    sudo apt install fail2ban -y
    
    # Configure fail2ban
    sudo tee /etc/fail2ban/jail.local > /dev/null <<EOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
EOF
    
    sudo systemctl enable fail2ban
    sudo systemctl start fail2ban
}

main() {
    harden_system
    setup_secrets
    configure_network_security
    
    echo "Security hardening completed!"
}

main "$@"
```

### Application Security
```javascript
// security-middleware.js
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');
const validator = require('validator');

// Rate limiting configuration
const createRateLimiter = (windowMs, max, message) => 
  rateLimit({
    windowMs,
    max,
    message: { error: message },
    standardHeaders: true,
    legacyHeaders: false
  });

// Input validation middleware
const validateInput = (validationRules) => (req, res, next) => {
  const errors = [];
  
  for (const [field, rules] of Object.entries(validationRules)) {
    const value = req.body[field];
    
    for (const rule of rules) {
      if (!rule.validate(value)) {
        errors.push(`${field}: ${rule.message}`);
      }
    }
  }
  
  if (errors.length > 0) {
    return res.status(400).json({ errors });
  }
  
  next();
};

// Cardano-specific validation
const cardanoValidation = {
  address: {
    validate: (addr) => /^addr[0-9a-z_]+$/.test(addr) && addr.length > 50,
    message: 'Invalid Cardano address format'
  },
  
  amount: {
    validate: (amount) => {
      const num = BigInt(amount);
      return num > 0n && num <= 45000000000000000n; // Max ADA supply
    },
    message: 'Invalid amount'
  },
  
  txHash: {
    validate: (hash) => /^[0-9a-f]{64}$/.test(hash),
    message: 'Invalid transaction hash format'
  }
};

module.exports = {
  // General rate limiting
  generalLimiter: createRateLimiter(15 * 60 * 1000, 100, 'Too many requests'),
  
  // Strict rate limiting for transaction endpoints
  transactionLimiter: createRateLimiter(60 * 1000, 10, 'Too many transaction requests'),
  
  // Security headers
  securityHeaders: helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https:"],
        connectSrc: ["'self'", "https://*.blockfrost.io"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    }
  }),
  
  // Input validation
  validateInput,
  cardanoValidation
};
```

## Production Checklist

### Pre-Deployment Checklist
- [ ] **Security Audit**
  - [ ] Smart contracts audited by professionals
  - [ ] Penetration testing completed
  - [ ] Dependencies scanned for vulnerabilities
  - [ ] Infrastructure hardened

- [ ] **Testing**
  - [ ] Unit tests passing (>90% coverage)
  - [ ] Integration tests on testnet
  - [ ] End-to-end testing completed
  - [ ] Performance testing under load
  - [ ] Security testing completed

- [ ] **Infrastructure**
  - [ ] Production environment configured
  - [ ] Monitoring and alerting setup
  - [ ] Backup and recovery procedures tested
  - [ ] SSL certificates configured
  - [ ] CDN and caching configured

- [ ] **Documentation**
  - [ ] API documentation updated
  - [ ] Deployment procedures documented
  - [ ] Runbooks for common issues
  - [ ] Emergency contact information
  - [ ] User guides and tutorials

### Post-Deployment Checklist
- [ ] **Verification**
  - [ ] All services healthy and responding
  - [ ] Database connections working
  - [ ] External API integrations functioning
  - [ ] SSL certificates valid
  - [ ] Monitoring dashboards active

- [ ] **Performance**
  - [ ] Response times within acceptable limits
  - [ ] Database queries optimized
  - [ ] CDN cache hit ratios acceptable
  - [ ] Resource utilization monitored

- [ ] **Security**
  - [ ] Security headers configured
  - [ ] Rate limiting active
  - [ ] Access logs being collected
  - [ ] Vulnerability scanning scheduled

- [ ] **Business Continuity**
  - [ ] Backup procedures verified
  - [ ] Disaster recovery plan tested
  - [ ] Support team notified
  - [ ] Rollback procedures confirmed

### Ongoing Maintenance
- [ ] **Regular Updates**
  - [ ] Security patches applied monthly
  - [ ] Dependencies updated quarterly
  - [ ] Performance reviews quarterly
  - [ ] Disaster recovery tests semi-annually

- [ ] **Monitoring**
  - [ ] Alert thresholds reviewed monthly
  - [ ] Log retention policies enforced
  - [ ] Capacity planning updated quarterly
  - [ ] Security scans automated

---

**Success Metrics**: Track deployment success with key metrics like uptime (>99.9%), response time (<200ms), error rate (<0.1%), and user satisfaction scores. Regular monitoring and optimization ensure long-term success of your Cardano applications.