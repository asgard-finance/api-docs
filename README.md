# Asgard Spot Margin Trading API Documentation

Base URL: `https://v2-ultra-edge.asgard.finance/margin-trading`

## Authentication & Rate Limits

All endpoints support authentication via an API Key. While public access is allowed, it is strictly rate-limited by IP address.

- **Header**: `X-API-Key`
- **Value**: Your provided API key string.

| Access Type | Rate Limit |
|-------------|------------|
| **Public (No Key)** | 1 Request Per Second (IP-based) |
| **Authenticated** | Custom Limits (Key-based) |

**Recommendation**: For consistent integration and to avoid `429 Too Many Requests` errors, always include the `X-API-Key` header in your requests.

This API provides endpoints for margin trading operations including viewing markets, creating/closing positions, and managing active positions across multiple lending protocols (Marginfi, Kamino, Solend, Drift).

> **Contact**: To obtain an API Key, please contact us on Telegram: **@asgardfi**

## Conceptual Overview

To effectively integrate with the Asgard Spot Margin Trading API, it is crucial to understand how we define markets, trading pairs, and position direction. This abstraction unifies multiple underlying lending protocols (Marginfi, Kamino, Solend, Drift) into a single consistent interface.

### Market Structure & Token Roles

Markets are defined as pairs of two assets: **Token A** and **Token B**.

- **Token A**: Typically the "Base" asset in a trading pair (e.g., SOL in SOL/USDC).
- **Token B**: Typically the "Quote" asset (e.g., USDC in SOL/USDC).

However, unlike a simple swap, margin trading involves borrowing one asset to hold the other. The roles of "Collateral" (Long Token) and "Liability" (Short Token) are determined dynamically based on the **Direction** of the trade.

### Direction Logic

The system maps the concept of **Long** and **Short** positions to the underlying Token A / Token B structure as follows:

| Direction | Long Token (Collateral) | Short Token (Debt/Liability) | Logic |
|-----------|-------------------------|------------------------------|-------|
| **LONG**  | **Token A**             | **Token B**                  | You are **buying** Token A. You borrow Token B (liability) to purchase more Token A (collateral). Your exposure is positive to Token A's price. |
| **SHORT** | **Token B**             | **Token A**                  | You are **selling** Token A. You borrow Token A (liability) and sell it for Token B (collateral). Your exposure is negative to Token A's price (or effectively long Token B relative to Token A). |

#### Example: SOL (Token A) / USDC (Token B)

- **Long SOL Position**:
  - **Direction**: `LONG` (0)
  - **Long Token**: SOL (Token A) -> This is what you hold.
  - **Short Token**: USDC (Token B) -> This is what you owe.
  - *Scenario*: You believe SOL will go up against USDC.

- **Short SOL Position**:
  - **Direction**: `SHORT` (1)
  - **Long Token**: USDC (Token B) -> This is what you hold.
  - **Short Token**: SOL (Token A) -> This is what you owe.
  - *Scenario*: You believe SOL will go down against USDC.

**Integrator Note**: When constructing a `TradeRequestBody`, you only need to specify `tokenABank`, `tokenBBank`, and `direction`. The system automatically handles the complex logic of which token to borrow and which to deposit based on this mapping.

## Enums & Constants

### LendingProtocol
| Name | Value | Description |
|------|-------|-------------|
| MARGINFI | 0 | Marginfi Protocol |
| KAMINO | 1 | Kamino Finance |
| SOLEND | 2 | Solend |
| DRIFT | 3 | Drift Protocol |

### MarginPosition (Direction)
| Name | Value | Description |
|------|-------|-------------|
| LONG | 0 | Long Position |
| SHORT | 1 | Short Position |

### CollateralType
| Name | Value | Description |
|------|-------|-------------|
| LONG | 0 | Collateral is the long asset |
| SHORT | 1 | Collateral is the short asset |
| ANY | 2 | Reserved for future use (Currently throws error) |

### PositionUpdateType
Used for tracking position modifications in the audit trail.

| Name | Value | Description |
|------|-------|-------------|
| INCREASE_LEVERAGE | 0 | Borrow more → higher leverage |
| DELEVERAGE | 1 | Repay debt → lower leverage |
| ADD_COLLATERAL | 2 | Add margin → better health |
| PARTIAL_CLOSE | 4 | Close X%, withdraw equity |
| REPAY_DEBT | 5 | Manual debt repayment |

`The other values (3, 6, 7, 8, 9) exist in the schema for future use but are not documented in the API since they're not implemented yet.`
---

## Market Data

### Get Markets
Returns available trading strategies and market data across supported protocols.

- **URL**: `/markets`
- **Method**: `GET`
- **Response**:
  ```ts
  {
    "strategies": {
      "Strategy Name": {
        "name": "string",
        "description": "string",
        "tokenAMint": "string",
        "tokenBMint": "string",
        "market": ["string"], // e.g., ["MARGIN"]
        "tag": ["string"],    // e.g., ["TRADING"]
        "liquiditySources": [
          {
            "lendingProtocol": number, // IMPORTANT: Identifies which protocol provides this specific market/strategy.
                                       // 0=Marginfi, 1=Kamino, 2=Solend, 3=Drift
            "isActive": boolean,
            "marketKey": "string", // Identifier for the market within the protocol (e.g., "Main Market")
            
            // Core Bank/Market Addresses
            "tokenABank": "string",
            "tokenBBank": "string",

            // Leverage Constraints
            "longMaxLeverage": number,
            "shortMaxLeverage": number,

            // Capacity Limits (String format to preserve precision)
            "tokenAMaxDepositCapacity": "string", 
            "tokenAMaxBorrowCapacity": "string",
            "tokenBMaxDepositCapacity": "string",
            "tokenBMaxBorrowCapacity": "string",

            // APY Rates (Decimal format, e.g., 0.05 = 5%)
            "tokenALendingApyRate": number,
            "tokenABorrowingApyRate": number,
            "tokenBLendingApyRate": number,
            "tokenBBorrowingApyRate": number,

            // Oracle Data (Protocol Specific - Treat as opaque/optional)
            "tokenAOracle": object, 
            "tokenBOracle": object,

            // Rewards (Optional)
            "tokenARewardAPY": [],
            "tokenBRewardAPY": [],
            
            // ... other protocol-specific fields (emissions, staking rates, etc.)
          }
        ]
      }
    },
    "totalStrategies": number
  }
  ```

#### Integrator Notes for `/markets`
1.  **`lendingProtocol` is Key**: This field within `liquiditySources` tells you exactly which underlying protocol is facilitating the trade. You must pass this `protocol` ID when calling `/create-position`.
2.  **Capacity Fields**: `tokenAMaxDepositCapacity`, etc., are returned as strings. These are crucial for UI checks to prevent users from trying to open positions larger than the pool can handle.
3.  **Protocol Specifics**: Fields like `tokenAOracle`, `oracleSetup`, and emission rates are highly specific to the underlying protocol (e.g., Pyth vs. Switchboard). Integrators should generally **ignore these** unless building advanced analytics features. Focus on `lendingProtocol`, `banks`, `leverage`, `capacities`, and `apyRates`.

---

## Position Management - Creation

### Create Position (Transaction Builder)
Generates the necessary transactions to open a position.



- **URL**: `/create-position`
- **Method**: `POST`
- **Body** (`TradeRequestBody`):
  ```ts
  {
    "protocol": number, // LendingProtocol
    "tokenABank": "string", // Long asset bank/market address
    "tokenBBank": "string", // Short asset bank/market address
    "direction": number, // MarginPosition (0=Long, 1=Short)
    
    "collateralType": number, // Specifies which token the user is depositing from their wallet to fund the position.
                              // Asgard allows funding with either asset in the pair.
                              
                              // 0 (Use Long Token): Deposit the asset defined as "Collateral" for this direction.
                              //    - Example (Long SOL/USDC): User deposits SOL (Token A).
                              //    - Example (Short SOL/USDC): User deposits USDC (Token B).
                              
                              // 1 (Use Short Token): Deposit the asset defined as "Liability" for this direction.
                              //    - Example (Long SOL/USDC): User deposits USDC (Token B).
                              //    - Example (Short SOL/USDC): User deposits SOL (Token A).
    
    "collateralAmountNative": number, // The user's deductible amount (initial deposit) in RAW native units.
                                      // This amount MUST be present in the user's wallet.
                                      // If the wallet balance is insufficient, the transaction will fail on-chain.
                                      // e.g., for 1.5 SOL (9 decimals), this would be 1500000000.
                                      // e.g., for 100 USDC (6 decimals), this would be 100000000.
                                      
    "collateralMint": "string", // The mint address of the token you are depositing.
                                // MUST match the mint of the token implied by `collateralType`.
    
    "collateralUnitPrice": number, // USD price
    "leverage": number, // e.g., 2.5
    "jitoTipLamport": number, // Tip for Jito Block Engine (in Lamports). See: https://docs.jito.wtf/lowlatencytxnsend/#get-tip-information
    "slippageBps": number,
    "walletPubkey": "string",
    "useDirectRoute": boolean
  }
  ```
- **Response**: Returns transaction data (structure varies by protocol, generally includes signed/unsigned transactions and metadata).

### Simulate Create Position
Simulates the position creation to estimate results without executing.

- **URL**: `/create-position-simulate`
- **Method**: `POST`
- **Body**: `TradeRequestBody` (same as `/create-position`)
- **Response**: Simulation results.

### Submit Create Position Transaction
Submits the signed transactions to the blockchain and records the position in the database.

**Important Flow Note**: The input body for this endpoint mirrors the output of `/create-position`, with one critical change: the `signedTxs` array must now contain the **actual base64-encoded signatures** (or fully signed transaction blobs) that the user has approved.

- **URL**: `/submit-create-position-tx`
- **Method**: `POST`
- **Body** (`SendAndConfirmPositionRequestBody`):
  ```ts
  {
    // ... All other fields from the `/create-position` response ...
    // Simply pass back the exact ts object you received, modifying ONLY the `signedTxs` field below.
    
    // CRITICAL: This must contain the signed transaction payloads
    "signedTxs": ["string"] 
  }
  ```
- **Response**: Database record of the created position.
  ```ts
  {
    "id": number,
    "ownerAddress": "string",
    "positionPda": "string",
    "protocol": number,
    "state": number, // 0 = Active
    "tokenATokenMint": "string",
    "tokenBTokenMint": "string",
    "leverage": "string", // Decimal string
    "tokenAEntryPriceUSD": "string",
    "tokenBEntryPriceUSD": "string",
    "openTxHash": "string", // The confirmed transaction hash on-chain
    "openedAt": "string" // ISO Date
    // ... matches the 'marginPositions' database schema
  }
  ```

---

## Position Management - Closing

### Close Position (Transaction Builder)
Generates transactions to close an existing position.

- **URL**: `/close-position`
- **Method**: `POST`
- **Body** (`CloseIsolatedPositionRequestBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "collateralType": number, // Desired output token type
    "jitoTipLamport": number,
    "slippageBps": number,
    "useDirectRoute": boolean
  }
  ```
- **Response**: Transaction data for closing the position.

### Simulate Close Position
Simulates the closing of a position.

- **URL**: `/close-position-simulate`
- **Method**: `POST`
- **Body**: `CloseIsolatedPositionRequestBody` (same as `/close-position`)
- **Response**: Simulation results.

### Submit Close Position Transaction
Submits signed transactions to close a position and updates the database.

**Important Flow Note**: Similar to position creation, the input body here mirrors the output of `/close-position` (or `/close-position-simulate`), with the `signedTxs` field populated with actual signatures.

- **URL**: `/submit-close-position-tx`
- **Method**: `POST`
- **Body** (`SubmitClosePositionTxRequestBody`):
  ```ts
  {
    // ... All other fields from the `/close-position` response ...
    // Simply pass back the exact ts object you received, modifying ONLY the `signedTxs` field below.

    // CRITICAL: This must contain the signed transaction payloads
    "signedTxs": ["string"]
  }
  ```
- **Response**: Updated position record and transaction metadata.

---

## Position Management - Modifications

These endpoints allow users to modify existing positions without fully closing them. All modification endpoints support **Kamino** and **Drift** protocols.

### Conceptual Overview

| Operation | What Changes | Equity | Leverage | Use Case |
|-----------|--------------|--------|----------|----------|
| **Partial Close** | Position size ↓ | Withdrawn to wallet | Same | "Take some profits" |
| **Deleverage** | Risk ↓ | Stays in position | Lower | "Reduce my risk" |
| **Increase Leverage** | Risk ↑ | Stays in position | Higher | "Maximize exposure" |
| **Add Collateral** | Safety ↑ | Increases | Lower | "Improve health" |
| **Repay Debt** | Debt ↓ | Increases | Lower | "Pay down loan" |

### Partial Close Position

Closes a specified percentage of the position and withdraws proportional equity to the user's wallet. **Leverage stays the same** - only position size decreases.

**Example (60% partial close at 5x leverage):**
- Before: 5 SOL ($500), 400 USDC debt, $100 equity
- After: 2 SOL ($200), 160 USDC debt, $40 equity in position, **$60 withdrawn**
- Leverage: Still 5x

- **URL**: `/position/partial-close`
- **Method**: `POST`
- **Supported Protocols**: Kamino (1), Drift (3)
- **Body** (`PartialClosePositionRequestBody`):
  ```ts
  {
    "protocol": number,           // LendingProtocol (1=Kamino, 3=Drift)
    "walletPubkey": "string",
    "positionPDA": "string",
    "closePercentage": number,    // 1-99 (percentage to close)
    "jitoTipLamport": number,
    "slippageBps": number,
    "useDirectRoute": boolean,
    "swapPlatform": "dflow" | "jupiter"  // Optional
  }
  ```
- **Response** (`PartialClosePositionResponseBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "closePercentage": number,

    // Before state
    "depositAmountUIBefore": number,
    "borrowAmountUIBefore": number,
    "leverageBefore": number,
    "marginAmountUIBefore": number,

    // After state (estimated)
    "depositAmountUIAfter": number,
    "borrowAmountUIAfter": number,
    "leverageAfter": number,
    "marginAmountUIAfter": number,

    // What user receives
    "equityWithdrawnUI": number,
    "equityWithdrawnMint": "string",

    // Prices
    "longTokenPriceUSD": number,
    "shortTokenPriceUSD": number,

    // Fees & APY
    "feesMetadata": object,
    "lendingApyAtClose": number,
    "borrowingApyAtClose": number,

    // Transactions (to be signed)
    "signedTxs": ["string"],

    // Data for submission
    "updateRecordData": object
  }
  ```

### Deleverage Position

Reduces leverage to a target level while **keeping the same equity**. User doesn't receive anything - equity stays in the position but with less risk.

**Example (5x → 3x deleverage):**
- Before: 5 SOL ($500), 400 USDC debt, $100 equity, 5x leverage
- After: 3 SOL ($300), 200 USDC debt, $100 equity, **3x leverage**
- User receives: Nothing (equity stays in position)

- **URL**: `/position/deleverage`
- **Method**: `POST`
- **Supported Protocols**: Kamino (1), Drift (3)
- **Body** (`DeleveragePositionRequestBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "targetLeverage": number,     // Must be >= 1.0 and < current leverage
    "jitoTipLamport": number,
    "slippageBps": number,
    "useDirectRoute": boolean,
    "swapPlatform": "dflow" | "jupiter"  // Optional
  }
  ```
- **Response** (`DeleveragePositionResponseBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",

    // Leverage change
    "leverageBefore": number,
    "leverageAfter": number,
    "targetLeverage": number,

    // Before state
    "depositAmountUIBefore": number,
    "borrowAmountUIBefore": number,
    "marginAmountUIBefore": number,

    // After state (estimated)
    "depositAmountUIAfter": number,
    "borrowAmountUIAfter": number,
    "marginAmountUIAfter": number,

    // Prices
    "longTokenPriceUSD": number,
    "shortTokenPriceUSD": number,

    // Fees & APY
    "feesMetadata": object,
    "lendingApyAtUpdate": number,
    "borrowingApyAtUpdate": number,

    // Transactions
    "signedTxs": ["string"],

    // Data for submission
    "updateRecordData": object
  }
  ```

### Increase Leverage Position

Increases leverage to a target level while **keeping the same equity**. Position grows with same equity but more risk.

**Example (3x → 5x increase):**
- Before: 3 SOL ($300), 200 USDC debt, $100 equity, 3x leverage
- After: 5 SOL ($500), 400 USDC debt, $100 equity, **5x leverage**
- User provides: Nothing (borrows more to increase position)

- **URL**: `/position/increase-leverage`
- **Method**: `POST`
- **Supported Protocols**: Kamino (1), Drift (3)
- **Body** (`IncreaseLeverageRequestBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "targetLeverage": number,     // Must be > current leverage
    "jitoTipLamport": number,
    "slippageBps": number,
    "useDirectRoute": boolean,
    "swapPlatform": "dflow" | "jupiter"  // Optional
  }
  ```
- **Response** (`IncreaseLeverageResponseBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",

    // Leverage change
    "leverageBefore": number,
    "leverageAfter": number,
    "targetLeverage": number,

    // Before state
    "depositAmountUIBefore": number,
    "borrowAmountUIBefore": number,
    "marginAmountUIBefore": number,

    // After state (estimated)
    "depositAmountUIAfter": number,
    "borrowAmountUIAfter": number,
    "marginAmountUIAfter": number,

    // Prices
    "longTokenPriceUSD": number,
    "shortTokenPriceUSD": number,

    // Fees & APY
    "feesMetadata": object,
    "lendingApyAtUpdate": number,
    "borrowingApyAtUpdate": number,

    // Transactions
    "signedTxs": ["string"],

    // Data for submission
    "updateRecordData": object
  }
  ```

### Add Collateral to Position

Adds collateral to an existing position, **reducing leverage and improving health**. Supports adding collateral in either the LONG or SHORT token.

**Example (adding $50 LONG collateral at 5x):**
- Before: 5 SOL ($500), 400 USDC debt, $100 equity, 5x leverage
- After: 5.5 SOL ($550), 400 USDC debt, $150 equity, **~3.67x leverage**

- **URL**: `/position/add-collateral`
- **Method**: `POST`
- **Supported Protocols**: Kamino (1), Drift (3)
- **Body** (`AddCollateralRequestBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "collateralAmountNative": number,  // Amount to add (in native units)
    "collateralType": number,          // 0=LONG (direct deposit), 1=SHORT (swap then deposit)
    "collateralMint": "string",        // Mint of the token being provided
    "jitoTipLamport": number,
    "slippageBps": number,             // Only needed if collateralType=SHORT (swap required)
    "useDirectRoute": boolean,
    "swapPlatform": "dflow" | "jupiter"  // Optional
  }
  ```
- **Response** (`AddCollateralResponseBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",

    // What was added
    "collateralAddedNative": number,
    "collateralAddedUI": number,
    "collateralMint": "string",
    "collateralType": number,

    // Leverage change
    "leverageBefore": number,
    "leverageAfter": number,

    // Before state
    "depositAmountUIBefore": number,
    "borrowAmountUIBefore": number,
    "marginAmountUIBefore": number,

    // After state
    "depositAmountUIAfter": number,
    "borrowAmountUIAfter": number,
    "marginAmountUIAfter": number,

    // Prices
    "longTokenPriceUSD": number,
    "shortTokenPriceUSD": number,

    // Fees & APY
    "feesMetadata": object,
    "lendingApyAtUpdate": number,
    "borrowingApyAtUpdate": number,

    // Transactions
    "signedTxs": ["string"],

    // Data for submission
    "updateRecordData": object
  }
  ```

### Repay Debt on Position

Manually repays debt using user's funds (SHORT token), **reducing leverage and improving margin**. This is a direct debt repayment without any swap.

**Example (repaying 100 USDC debt):**
- Before: 5 SOL ($500), 400 USDC debt, 5x leverage
- After: 5 SOL ($500), 300 USDC debt, **~2.5x leverage**

- **URL**: `/position/repay-debt`
- **Method**: `POST`
- **Supported Protocols**: Kamino (1), Drift (3)
- **Body** (`RepayDebtRequestBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "repayAmountNative": number,   // Amount to repay (in SHORT/debt token native units)
    "jitoTipLamport": number
  }
  ```
- **Response** (`RepayDebtResponseBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",

    // What was repaid
    "repayAmountNative": number,
    "repayAmountUI": number,
    "debtTokenMint": "string",

    // Before state
    "depositAmountUIBefore": number,
    "borrowAmountUIBefore": number,
    "leverageBefore": number,
    "marginAmountUIBefore": number,

    // After state
    "depositAmountUIAfter": number,   // Same as before
    "borrowAmountUIAfter": number,    // Lower
    "leverageAfter": number,          // Lower

    // Prices
    "longTokenPriceUSD": number,
    "shortTokenPriceUSD": number,

    // Fees & APY
    "feesMetadata": object,
    "lendingApyAtUpdate": number,
    "borrowingApyAtUpdate": number,

    // Transactions
    "signedTxs": ["string"],

    // Data for submission
    "updateRecordData": object
  }
  ```

### Submit Position Update Transaction

**Unified submission endpoint** for ALL position modifications. After signing the transactions from any of the above endpoints, submit them here.

- **URL**: `/submit-position-update-tx`
- **Method**: `POST`
- **Body** (`SubmitPositionUpdateTxRequestBody`):
  ```ts
  {
    "protocol": number,
    "walletPubkey": "string",
    "positionPDA": "string",
    "positionId": number,            // margin_positions.id (from your position record)
    "tokenATokenMint": "string",
    "tokenBTokenMint": "string",
    
    // CRITICAL: Signed transactions
    "signedTxs": ["string"],
    
    // Pass back the updateRecordData from the response
    "updateRecordData": {
      "updateType": number,          // PositionUpdateType enum
      "updatePercentage": number | null,
      
      "preTokenAAmountUI": number,
      "preTokenBAmountUI": number,
      "preLeverage": number,
      "preMarginAmountUI": number | null,
      
      "collateralType": number | null,
      "collateralMint": "string" | null,
      "collateralAmountNative": number | null,
      "collateralAmountUI": number | null,
      
      "swapInAmount": number | null,
      "swapInMint": "string" | null,
      "swapOutAmount": number | null,
      "swapOutMint": "string" | null,
      
      "postTokenAAmountUI": number,
      "postTokenBAmountUI": number,
      "postLeverage": number,
      "postMarginAmountUI": number | null,
      
      "tokenAPriceUSD": number,
      "tokenBPriceUSD": number
    },
    
    // Optional metadata
    "feesMetadata": object,
    "swapRouteMetadata": object,
    "lendingApyAtUpdate": number,
    "borrowingApyAtUpdate": number
  }
  ```
- **Response** (`SubmitPositionUpdateTxResponseBody`):
  ```ts
  {
    "success": boolean,
    "txHash": "string",
    "txMetadata": object,
    
    // Created audit record
    "positionUpdate": {
      "id": number,
      "updateType": number,
      "status": number   // 1 = confirmed
    },
    
    // Updated position state
    "position": {
      "id": number,
      "tokenAAmountUICurrent": "string",
      "tokenBAmountUICurrent": "string",
      "leverageCurrent": "string",
      "updateCount": number
    }
  }
  ```

### Integration Flow for Position Modifications

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Call modification endpoint (e.g., /position/deleverage)    │
│     → Returns unsigned transactions + updateRecordData         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Sign transactions with user's wallet                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Call /submit-position-update-tx                            │
│     → Pass signed txs + updateRecordData                       │
│     → Creates audit trail in position_updates table            │
│     → Updates margin_positions with current state              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Receive confirmation with txHash and updated position      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Position Queries

### Get All User Positions
Retrieves all positions (active and closed) for a user.

- **URL**: `/positions`
- **Method**: `POST`
- **Body**:
  ```ts
  {
    "walletPubkey": "string"
  }
  ```
- **Response**: Array of position objects with attached liquidation events.

### Get Open User Positions
Retrieves only active positions for a user.

- **URL**: `/positions/open`
- **Method**: `POST`
- **Body**:
  ```ts
  {
    "walletPubkey": "string"
  }
  ```
- **Response**: Array of active position objects.

### Get Closed User Positions
Retrieves only closed positions for a user.

- **URL**: `/positions/closed`
- **Method**: `POST`
- **Body**:
  ```ts
  {
    "walletPubkey": "string"
  }
  ```
- **Response**: Array of closed position objects.

### Get Position by PDA
Retrieves a specific position by its PDA (Program Derived Address).

- **URL**: `/positions/by-pda?pda=<position_pda>`
- **Method**: `POST`
- **Body**: Empty (uses query param)
- **Response**: Single position object.

### Refresh Positions
Fetches live data from the blockchain for specified positions to update their status, including real-time health factors and asset balances.

- **URL**: `/refresh-positions`
- **Method**: `POST`
- **Body** (`RefreshPositionRequestBody`):
  ```ts
  {
    "positionPDAs": [
      {
        "protocol": number,
        "positionPDA": "string"
      }
    ]
  }
  ```
- **Response**:
  ```ts
  {
    "positions": {
      "<pda>": {
        "accountAddress": "string", // Position PDA
        "authority": "string",
        "summary": {
          "healthFactor": number, // How much the entire market can drop until your position is eligible for liquidation. This accounts for all the assets in your position, assuming stablecoins retain their 1:1 peg, and all volatile assets drop in value by this percentage.
          "booksAndBalances": {
             "balances": [
                {
                    "<bank_address_or_market_index>": {
                        "mint": "string",
                        "assetsQt": number,  // Amount held (Collateral). If > 0, this is your Long position.
                        "liabsQt": number,   // Amount owed (Debt). If > 0, this is your Short position.

                        // The following USD fields are optional and protocol-dependent. Avoid relying on them.
                        "assetsUsd": number, 
                        "liabsUsd": number
                    }
                }
             ]
          }
        },
        "liquidationEvents": []
      }
    }
  }
  ```

#### Interpreting the Response Data

**⚠️ INTEGRATION PRIORITY: Focus on `healthFactor` and `booksAndBalances`**

These two fields are the **standardized sources of truth** for position health and composition across all protocols.
*   **`healthFactor`**: Your single metric for safety.
*   **`booksAndBalances`**: Your single source for asset composition (what is held vs. what is owed).

**Note on Other Fields**: The response object may include various other fields (e.g., `lendingAmount`, `borrowingAmount`, `riskMetrics`, `balance`, `assetsUsd`, `liabsUsd`). **These fields are optional, protocol-dependent, and not guaranteed to be present or consistent.**

**Integrator Action**: Treat all fields *except* `healthFactor` and `booksAndBalances` as unstable unless explicitly documented otherwise by us. **Do not build core application logic around them.**

1.  **Health Factor (Liquidation Risk)**
    *   The `healthFactor` (and `liquidationBuffer` in `riskMetrics`) represents the **safety margin** of the position.
    *   **Value Meaning**: A value of `40.94` means the collateral asset's price can drop by roughly **40.94%** before the position is liquidated.
    *   **Critical Action**: If this number approaches 0, liquidation is imminent.

2.  **Identifying Long vs. Short Assets (`booksAndBalances`)**
    *   The `booksAndBalances.balances` array contains the specific state of each asset in the position.
    *   **To find the LONG asset (Collateral)**: Look for the entry where `assetsQt` > 0.
    *   **To find the SHORT asset (Debt)**: Look for the entry where `liabsQt` > 0.
    *   *Note*: In isolated margin trading, one asset will typically be purely collateral (`assetsQt` > 0, `liabsQt` = 0) and the other purely debt (`assetsQt` = 0, `liabsQt` > 0).

3.  **Protocol Differences**
    *   **Marginfi/Kamino**: Keys in `balances` are typically **Bank Addresses** (Public Keys).
    *   **Drift**: Keys in `balances` are **Market Indexes** (e.g., "0", "1"). The logic remains the same.

### Get All Liquidation Events
Retrieves a history of all liquidation events.

- **URL**: `/all-liquidation-events`
- **Method**: `POST`
- **Response**: Array of liquidation event records.

---

## Token Metadata

### Get All Token Metadata
Returns metadata for all supported tokens.

- **URL**: `/token-metadata`
- **Method**: `GET`
- **Response**:
  ```ts
  {
    "tokens": [
      {
        "symbol": "string",
        "name": "string",
        "mint": "string",
        "decimals": number,
        "logoURI": "string",
        "price": number,
        // ...
      }
    ],
    "stats": object
  }
  ```

### Get Token Metadata by Mint
Returns metadata for a specific token.

- **URL**: `/token-metadata/:mint`
- **Method**: `GET`
- **Response**: Token metadata object.
---

## Database Schema & Position Lifecycle

The core of the integration revolves around the `margin_positions` table. This is the definitive record of all user positions.

### Core Position Schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | Integer | Unique primary key for the position record. |
| `ownerAddress` | String | Wallet address of the user who owns the position. |
| `positionPda` | String | **Critical**: The on-chain Program Derived Address that holds the position's state. Use this for all updates/closes. |
| `state` | Enum (Int) | **0 = Active**: Position is open.<br>**1 = Closed**: Position has been fully repaid/closed.<br>**2 = Liquidated** (Deprecated, see events). |
| `protocol` | Enum (Int) | The underlying protocol used (0=Marginfi, 1=Kamino, etc.). |

### Asset & Financial Data

| Field | Type | Description |
|-------|------|-------------|
| `tokenATokenMint` | String | Mint address of Token A (Base Asset). |
| `tokenBTokenMint` | String | Mint address of Token B (Quote Asset). |
| `positionSide` | Enum (Int) | **0 = Long**: Long Token A, Short Token B.<br>**1 = Short**: Short Token A, Long Token B. |
| `leverage` | Decimal | The leverage ratio at the time of opening (e.g., "3.5"). |
| `tokenAEntryPriceUSD` | Decimal | Fallback/Estimated price of Token A at open. **Use `entryMarketMetadata` for executed price.** |
| `tokenBEntryPriceUSD` | Decimal | Fallback/Estimated price of Token B at open. **Use `entryMarketMetadata` for executed price.** |
| `tokenAExitPriceUSD` | Decimal | Fallback/Estimated price of Token A at close. **Use `exitMarketMetadata` for executed price.** |
| `tokenBExitPriceUSD` | Decimal | Fallback/Estimated price of Token B at close. **Use `exitMarketMetadata` for executed price.** |

### On-Chain Execution Data (Hedge Fund Specific)

For high-precision accounting, rely on the `entryMarketMetadata` and `exitMarketMetadata` ts blobs. These contain the *actual* swapped amounts confirmed on-chain.

> **Note**: These fields may take up to **60 seconds** to populate after transaction submission. We are working to make this instantaneous.

**Deriving Realized Execution Price:**
```javascript
// Example: Calculate realized Token A entry price
const meta = position.entryMarketMetadata.onChainExecutedMetadata;
const executedPrice = meta.outputAmount / meta.inputAmount; 
```

| Field | Type | Description |
|-------|------|-------------|
| `entryMarketMetadata` | ts | Contains `onChainExecutedMetadata` with `inputMint`, `outputMint`, `inputAmount`, `outputAmount` for the opening swap. |
| `exitMarketMetadata` | ts | Contains `onChainExecutedMetadata` for the closing swap. |
| `openFeesMetadata` | ts | Breakdown of all fees paid to open (Jito tips, Protocol fees, DEX fees). |
| `closeFeesMetadata` | ts | Breakdown of fees paid to close. |

### Transaction Tracking

| Field | Type | Description |
|-------|------|-------------|
| `openTxHash` | String | The transaction hash that created the position. |
| `closeTxHash` | String | The transaction hash that closed the position (Null if active). |
| `openedAt` | Timestamp | UTC timestamp of creation. |
| `closedAt` | Timestamp | UTC timestamp of closure (Null if active). |

### Current State Tracking (Position Modifications)

These fields track the **current state** of the position after modifications. If null, use the Entry values (position hasn't been modified).

| Field | Type | Description |
|-------|------|-------------|
| `tokenAAmountUICurrent` | Decimal | Current deposit/long amount (null if unmodified). |
| `tokenBAmountUICurrent` | Decimal | Current borrow/short amount (null if unmodified). |
| `leverageCurrent` | Decimal | Current leverage after modifications (null if unmodified). |
| `lastUpdateId` | Integer | Reference to the most recent `position_updates.id`. |
| `updateCount` | Integer | Number of times the position has been modified. |

**Integrator Note**: To get the current position state:
```javascript
// Get current amounts (fallback to entry if never modified)
const currentDeposit = position.tokenAAmountUICurrent ?? position.tokenAAmountUIEntry;
const currentBorrow = position.tokenBAmountUICurrent ?? position.tokenBAmountUIEntry;
const currentLeverage = position.leverageCurrent ?? position.leverage;
```

---

## Position Updates (Audit Trail)

Every position modification is recorded in the `position_updates` table for complete audit trail and historical analysis.

### Position Updates Schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | Integer | Unique primary key. |
| `positionId` | Integer | Reference to `margin_positions.id`. |
| `positionPda` | String | Position PDA for easy querying. |
| `ownerAddress` | String | Wallet address. |
| `protocol` | Integer | LendingProtocol enum. |
| `updateType` | Integer | PositionUpdateType enum (see Enums section). |
| `updatePercentage` | Decimal | For PARTIAL_CLOSE: percentage closed (1-99). |

### Pre-Update State

| Field | Type | Description |
|-------|------|-------------|
| `preTokenAAmountUI` | Decimal | Deposit/Long amount before update. |
| `preTokenBAmountUI` | Decimal | Borrow/Short amount before update. |
| `preLeverage` | Decimal | Leverage before update. |
| `preMarginAmountUI` | Decimal | Net margin before update. |

### Update Details

| Field | Type | Description |
|-------|------|-------------|
| `collateralType` | Integer | CollateralType enum - what token was provided/received. |
| `collateralMint` | String | Mint address of collateral token. |
| `collateralAmountNative` | Decimal | Amount of collateral added/removed (native units). |
| `collateralAmountUI` | Decimal | Amount of collateral added/removed (UI units). |
| `swapInAmount` | Decimal | Amount swapped in (if applicable). |
| `swapInMint` | String | Mint of swap input. |
| `swapOutAmount` | Decimal | Amount received from swap. |
| `swapOutMint` | String | Mint of swap output. |

### Post-Update State

| Field | Type | Description |
|-------|------|-------------|
| `postTokenAAmountUI` | Decimal | Deposit/Long amount after update. |
| `postTokenBAmountUI` | Decimal | Borrow/Short amount after update. |
| `postLeverage` | Decimal | Leverage after update. |
| `postMarginAmountUI` | Decimal | Net margin after update. |
| `tokenAPriceUSD` | Decimal | TokenA price at update time. |
| `tokenBPriceUSD` | Decimal | TokenB price at update time. |

### Transaction & Status

| Field | Type | Description |
|-------|------|-------------|
| `txHash` | String | Transaction hash for the update. |
| `txMetadata` | JSON | Full transaction metadata. |
| `swapRouteMetadata` | JSON | Swap routing details (if applicable). |
| `feesMetadata` | JSON | Detailed fee breakdown. |
| `status` | Integer | 0=pending, 1=confirmed, 2=failed. |
| `finalized` | Boolean | Whether transaction is finalized on-chain. |
| `createdAt` | Timestamp | When the update was initiated. |
| `confirmedAt` | Timestamp | When the update was confirmed. |

---

## Error Handling

Errors are returned with appropriate HTTP status codes (typically 422 for logical errors, 500 for server errors).

**Common Error Format:**
```ts
{
  "error": {
    "name": "string",
    "code": "string",
    "message": "string",
    "details": any
  }
}
```
