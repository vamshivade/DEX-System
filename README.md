# DEX System Architecture & Technical Audit

## 1. System Architecture Overview

The platform implements a sophisticated, microservices-based automated trading and liquidity management system on the Solana blockchain. Queries are distributed across specialized services to separate trading execution, pool monitoring, and reporting analytics.

### High-Level Service Map

```mermaid
graph TD
    User[User / Admin] -->|HTTPS| FE_MM[MM-FRONT]
    User -->|HTTPS| FE_REP[REPORTS-FRONT]

    subgraph "Frontend Layer"
        FE_MM
        FE_REP
    end

    FE_MM -->|REST API| MM_BACK[MM-BACK Team]
    FE_MM -->|REST API| POOL_BACK[POOL-BACK Team]
    FE_REP -->|REST API| REP_BACK[REPORTS-BACK Team]

    subgraph "Backend Services"
        MM_BACK -->|Trading Logic| SolRPC[Solana RPC]
        POOL_BACK -->|Monitoring| SolRPC
        REP_BACK -->|Data Aggregation| ExtAPI[External APIs]
    end

    subgraph "External Integrations"
        ExtAPI -->|Price/Vol| DexScreener
        ExtAPI -->|Tx History| Solscan
        SolRPC -->|Swap/Liquidity| Raydium
        POOL_BACK -->|Alerts| Telegram
    end

    subgraph "Data Persistence"
        MM_BACK --> DB[(MongoDB Cluster)]
        POOL_BACK --> DB
        REP_BACK --> DB
    end
```

### Execution Lifecycle (Request Flow)

1.  **Client Request**: Initiated from `MM-FRONT` (e.g., "Create Bot").
2.  **API Gateway**: `MM-BACK` Express server receives request on `/api/*`.
3.  **Controller Layer**: Validates input using `express-validator` or custom logic.
4.  **Service Layer**:
    - Fetches wallet keys (decrypted via AWS Secrets Manager/S3).
    - Consults `Raydium SDK` for pool state.
    - Simulates transaction computation.
5.  **Execution**: Signs transaction with `Keypair` and sends to Solana network via Helius RPC.
6.  **Persistence**: On success (`txId` confirmation), transaction details are saved to MongoDB (`SwapModel`, `SwappedData`).
7.  **Response**: JSON payload returned to client.

---

## 2. Frontend Deep Architecture (`MM-FRONT`)

### Architectural Patterns

- **React + Vite**: Uses a modern, fast-build toolchain.
- **Context API for State**: `AuthContext.jsx` manages global user session and JWT handling, avoiding Redux boilerplate for this specific scope.
- **Provider Pattern**: Wraps app in `WalletContext` (Solana adapters) and `AuthProvider`.
- **Protected Routing**: HOC pattern (`ProtectedRoute`, `PublicRoute`) guards routes based on authentication and role (Admin/User).

### Key Components

- **Wallet Integration**: Uses `@solana/wallet-adapter-react` to connect Phantom/Solflare. This is crucial for _signing_ transactions on the client side (if implemented) or just wallet verification.
- **API Access**: `services/api.js` acts as a Singleton Axios wrapper with interceptors for:
  - Automatically attaching `Bearer` tokens.
  - Global error handling (auto-logout on 401).

### Authentication Lifecycle

1.  **Login**: User submits credentials -> `authRoutes/login` -> Returns JWT.
2.  **Storage**: JWT stored in `localStorage` (Persistent).
3.  **Session**: Validated on every route change via `AuthContext`.
4.  **Expiry**: Token expiration triggers interceptor -> Clear Storage -> Redirect Login.

---

## 3. Backend Deep Architecture (`MM-BACK`, `POOL-BACK`)

### Design Pattern: Layered Architecture (Controller-Service-Model)

- **Controllers**: Handle HTTP `req/res`, parsing, and status codes.
- **Services (Implicit)**: Business logic often resides directly in controllers (e.g., `controller/cpmm.js`), which functions as a service layer for Raydium interactions.
- **Models**: Mongoose schemas define strict data structure.

### MM-BACK (Market Maker)

- **Core Responsibility**: Executing automated swaps (Market Making).
- **Automation**: `setInterval` in `server.js` triggers `checkBots()`.
- **Logic**:
  - **Price Check**: Fetches price from Binance/DexScreener.
  - **Threshold Match**: Compares `currentPrice` vs `previousPrice`.
  - **Strategy**: Executes `inverse` (Buy Low/Sell High) or `reverse` (Momentum) strategies.
  - **Concurrency Control**: valid `swapLocks` map prevents double-execution for the same bot.

### POOL-BACK (Pool Management)

- **Core Responsibility**: Monitoring Liquidity Channels (CLMM).
- **Automation**: Polls every 60s (`PoolController.checkAndAlert`).
- **Logic**:
  - **Range Check**: Compares current `tick` vs position `lower/upper` bounds.
  - **Alerting**: Sends Telegram messages on "Out of Range".
  - **Auto-Action**: Triggers `harvest` and `reback` (rebalance) transactions automatically when ranges are breached.

### Security Implementation

- **Secret Management**:
  - **AWS Secrets Manager**: Stores encryption keys `getEncryptionKeyFromSecretsManager`.
  - **AWS S3**: Stores _encrypted_ wallet files.
- **Runtime Decryption**: Private keys are decrypted _only_ in memory during execution and never logged (sanitized in logs).
- **Environment**: strict `.env` usage for RPC URLs and API Keys.

---

## 4. Database Engineering (MongoDB)

### Schema Design & Scalability

- **Document-Oriented**: Suitable for storing variable-structure blockchain data (transaction receipts, pool info).
- **Collections**:
  - `users`: Auth & Roles.
  - `swapmodels`: Bot configurations (Target Pool, Mint A/B, Strategy).
  - `swappeddatas`: Immutable ledger of executed trades (Audit trail).
  - `wallets`: Managed wallet private keys (Encrypted references) and metadata.
  - `rebackmodels`: Tracking rebalancing events for CLMM pools.

### Data Consistency

- **Transactional Integrity**: Logic often checks on-chain confirmation (`connection.confirmTransaction`) _before_ writing to DB. This ensures DB reflects on-chain reality (Event Sourcing pattern).
- **Optimistic Locking**: Not explicitly used, but `swapLocks` in memory serve as a mutex for bot operations.

---

## 5. Blockchain & External API Integration

### Integration Strategy

The system handles high-frequency data from multiple sources:

| Source                  | Type     | Usage                                                 | rate_limit Handling                            |
| :---------------------- | :------- | :---------------------------------------------------- | :--------------------------------------------- |
| **Solana RPC (Helius)** | JSON-RPC | Transaction submission, Account info, Blockhash       | High-throughput paid tier assumed              |
| **Raydium SDK**         | Library  | Swap instruction construction, Pool state parsing     | Client-side computation (Off-chain)            |
| **DexScreener**         | REST API | Real-time price feeds (`/latest/dex/pairs/...`)       | Caching not explicit; reliance on direct fetch |
| **Solscan**             | REST API | Historical transaction analysis (`/account/transfer`) | Pagination implementation (`page`, `limit`)    |
| **Binance**             | REST API | Benchmark price fetching for arbitrage logic          | Standard public limits                         |

### Trading Engines

#### 1. CPMM Engine (`controller/cpmm.js`)

- **Protocol**: Standard AMM (x \* y = k).
- **Flow**:
  1. `fetchPoolById`: Get reserves.
  2. `CurveCalculator.swapBaseInput`: Off-chain simulation to calculate specific output.
  3. `balance.lt(input)`: Pre-flight balance check.
  4. `raydium.cpmm.swap`: Compile instructions.

#### 2. CLMM Engine (`controller/manualSwap.js`, `PoolController.js`)

- **Protocol**: Concentrated Liquidity.
- **Flow**:
  1. Fetch `tickArrayCache`: Mandatory for traversing liquidity bins.
  2. `PoolUtils.computeAmountOutFormat`: Complex calculation considering tick crossings.
  3. **Rebalancing**: If price moves out of tick range, the system closes position (withdraw), swaps tokens to ratio, and opens new position (deposit) centered on current price.

---

## 6. Calculation & Financial Logic

### Precision Handling

- **BN.js / BigNumber**: ALL token amounts are handled as Big Numbers to prevent floating-point errors.
- **Decimal Normalization**:
  - Inputs: Human readable (e.g., "1.5 SOL").
  - Internal: Converted to Lamports/Raw units (`amount * 10^decimals`).
  - Outputs: Converted back for DB storage.

### Profit & Strategy Logic

- **Inverse Mode**:
  - $\Delta Price \ge Threshold \rightarrow Sell$
  - $\Delta Price \le -Threshold \rightarrow Buy$
- **Reverse Mode**:
  - $\Delta Price \ge Threshold \rightarrow Buy$ (Momentum)
  - $\Delta Price \le -Threshold \rightarrow Sell$ (Panic/Stop-loss)
- **Slippage Protection**: Hardcoded or configurable user setting (e.g., 1%). Passed to SDK to generate `minAmountOut`. Transaction fails on-chain if slippage is exceeded.

---

## 7. Security Model

1.  **Wallet Security**:
    - **Non-Custodial Logic**: App acts as a signer using stored keys.
    - **Encryption**: Keys are AES-encrypted before storage.
2.  **Access Control**:
    - `authRoutes`: JWT protection for API endpoints.
    - `adminOnly` middleware: Restricts bot configuration to authorized personnel.
3.  **Input Validation**:
    - Frontend: Type checking and required field validation.
    - Backend: Mongoose casting ensures data types (e.g., Numbers are numbers).
4.  **Infrastructure**:
    - `cors`: configured to allow specific origins (or `*` in dev).
    - `trust proxy`: Enabled for correct IP logging behind load balancers.

---

## 8. Scalability & Production Considerations

- **Statefulness**: The backend services are largely _stateless_ (except for in-memory `botPrices` and `swapLocks`). This allows for horizontal scaling if sticky sessions or a Redis store replaces the in-memory maps.
- **Bottlenecks**:
  - **RPC Limits**: Heavy polling (`setInterval` every 60s * N bots) will hit RPC rate limits. *Recommendation: Switch to WebSocket subscriptions for account updates.\*
  - **Database**: MongoDB handles high read/write well, but historical report generation (`REPORTS-BACK`) aggregation pipelines can represent a heavy load.
- **Failover**:
  - Error handling in `checkBots` ensures one failing bot doesn't crash the loop.
  - `pm2` or `nodemon` ensures process restarts.

---

_Verified by System Audit - 2026_
