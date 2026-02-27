# Tokenize Escrow Flow - Current Architecture

## Overview
This document maps the current tokenize escrow flow, identifying all components, file paths, and the sequence of operations. It is aligned with the codebase as of the last update: deployment order is Token Sale first (with placeholder token), then Token Factory with immutable `mint_authority`, then Token Sale updated via `set_token`; the UI collects escrow ID, token name, and token symbol; the backoffice app exposes the primary deploy API.

## File Paths

### Backoffice UI Components
- **Route/Page**: `apps/backoffice-tokenization/src/app/manage-escrows/page.tsx`
- **View Component**: `apps/backoffice-tokenization/src/features/manage-escrows/ManageEscrowsView.tsx`
- **Dialog Component**: `apps/backoffice-tokenization/src/features/tokens/deploy/dialog/TokenizeEscrow.tsx`
- **Hook**: `apps/backoffice-tokenization/src/features/tokens/deploy/dialog/useTokenizeEscrow.ts`
- **Service**: `apps/backoffice-tokenization/src/features/tokens/services/token.service.ts`
- **Success Dialog**: `apps/backoffice-tokenization/src/features/tokens/deploy/dialog/TokenizeEscrowSuccessDialog.tsx`

### API Endpoint (Backoffice – flujo principal)
- **Route Handler**: `apps/backoffice-tokenization/src/app/api/deploy/route.ts`
- **Deployment Service**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts`
- **Soroban Client**: `apps/backoffice-tokenization/src/lib/sorobanClient.ts`

### API Endpoint (Investor – equivalente)
- **Route Handler**: `apps/investor-tokenization/src/app/api/deploy/route.ts`
- **Deployment Service**: `apps/investor-tokenization/src/lib/tokenDeploymentService.ts`
- **Soroban Client**: `apps/investor-tokenization/src/lib/sorobanClient.ts`

### Smart Contracts
- **Token Contract (Token Factory)**: `apps/smart-contracts/contracts/token-factory/src/contract.rs`
- **Token Sale Contract**: `apps/smart-contracts/contracts/token-sale/src/sale.rs`
- **Token Metadata Module**: `apps/smart-contracts/contracts/token-factory/src/metadata.rs` (standard metadata + immutable escrow_id and mint_authority)

## Form Fields

### TokenizeEscrow Dialog
The UI form contains three fields (all required):
- **escrowId** (string, required): The Escrow contract ID to tokenize
- **tokenName** (string, required): Display name of the token (e.g. "Trustless Work Token")
- **tokenSymbol** (string, required): Ticker/symbol, max 12 characters, uppercase letters and numbers only (e.g. "TRUST")

**Location**: `apps/backoffice-tokenization/src/features/tokens/deploy/dialog/TokenizeEscrow.tsx` (lines 49-128).  
Form values are typed in `useTokenizeEscrow.ts` as `TokenizeEscrowFormValues` and sent to the API via `TokenService.deployToken({ escrowContractId, tokenName, tokenSymbol })`.

## Sequence Diagram

```
User (Browser)
    │
    ├─> [1] Navigate to /manage-escrows
    │   └─> ManageEscrowsView.tsx
    │       └─> TokenizeEscrowDialog (Button trigger)
    │
    ├─> [2] User clicks "Tokenize Escrow" button
    │   └─> Opens TokenizeEscrowDialog
    │
    ├─> [3] User enters Escrow ID, Token Name, Token Symbol and submits form
    │   └─> useTokenizeEscrow.onSubmit()
    │       └─> TokenService.deployToken({ escrowContractId, tokenName, tokenSymbol })
    │
    ├─> [4] HTTP POST to /api/deploy (baseURL from NEXT_PUBLIC_API_URL or "/api")
    │   └─> apps/backoffice-tokenization/src/app/api/deploy/route.ts (when using backoffice UI)
    │       └─> deployTokenContracts(sorobanClient, { escrowContractId, tokenName, tokenSymbol })
    │
    ├─> [5] Upload WASM and deploy Token Sale first (placeholder token)
    │   └─> tokenDeploymentService.ts:deployTokenContracts()
    │       ├─> Upload Token Factory WASM, then Token Sale WASM
    │       ├─> Create Token Sale Contract (first, to get its address)
    │       │   └─> Constructor Args:
    │       │       ├─> escrow_contract: escrowContractId
    │       │       ├─> sale_token: client.publicKey (placeholder)
    │       │       └─> admin: client.publicKey (deployer; can call set_token)
    │       │   └─> sale.rs:__constructor() → write_config(), write_admin()
    │
    ├─> [6] Deploy Token Factory with mint_authority = Token Sale
    │   └─> tokenDeploymentService.ts:deployTokenContracts()
    │       ├─> Create Token Factory Contract
    │       │   └─> Constructor Args:
    │       │       ├─> name: tokenName (user-provided)
    │       │       ├─> symbol: tokenSymbol (user-provided)
    │       │       ├─> escrow_id: escrowContractId (string)
    │       │       ├─> decimal: 7
    │       │       └─> mint_authority: tokenSaleAddress
    │       │   └─> contract.rs:__constructor()
    │       │       ├─> write_metadata(&e, TokenMetadata { decimal, name, symbol })
    │       │       ├─> write_escrow_id(&e, &escrow_id)
    │       │       └─> write_mint_authority(&e, &mint_authority)  [Immutable]
    │
    ├─> [7] Update Token Sale with real token address
    │   └─> tokenDeploymentService.ts: callContract(tokenSaleAddress, "set_token", [tokenFactoryAddress])
    │       └─> sale.rs:set_token()
    │           ├─> read_admin(&env).require_auth()
    │           └─> write token address in config (escrow unchanged)
    │
    └─> [8] Return deployment results
        └─> Response: { success, tokenFactoryAddress, tokenSaleAddress }
            └─> TokenizeEscrowSuccessDialog with contract addresses
```

## Mint Authority Assignment

### Current Flow
1. **Token Sale is deployed first** with a placeholder `sale_token` (deployer address) and `admin` (deployer). This yields `tokenSaleAddress` needed for the Token Factory constructor.
   - **Location**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts` (Token Sale creation, lines 49-56)
   - **Contract**: `apps/smart-contracts/contracts/token-sale/src/sale.rs:__constructor(env, escrow_contract, sale_token, admin)`

2. **Token Factory is deployed with immutable mint_authority**: The Token Factory constructor receives `mint_authority: tokenSaleAddress` directly. There is no admin role and no `set_admin`; mint authority is set once at deployment and cannot be changed.
   - **Location**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts` (Token Factory creation, lines 64-71)
   - **Contract**: `apps/smart-contracts/contracts/token-factory/src/contract.rs:__constructor(e, name, symbol, escrow_id, decimal, mint_authority)`
   - **Storage**: `write_mint_authority(&e, &mint_authority)` — immutable (panics if already set)

3. **Token Sale is updated with the real token address** via `set_token(tokenFactoryAddress)` (admin-only), so the sale contract can call `mint` on the token.
   - **Location**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts` (lines 77-82)
   - **Contract**: `apps/smart-contracts/contracts/token-sale/src/sale.rs:set_token(env, new_token)`

### Key Code Locations

**Token Factory constructor (mint_authority set at deploy)**:
```typescript
// apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts (Token Factory createContract args)
[
  client.nativeString(tokenName),
  client.nativeString(tokenSymbol),
  client.nativeString(escrowContractId),
  client.nativeU32(7),
  client.nativeAddress(tokenSaleAddress), // mint_authority (Token Sale contract)
]
```

**Token contract: no set_admin; mint_authority immutable**:
```rust
// apps/smart-contracts/contracts/token-factory/src/contract.rs:__constructor()
write_mint_authority(&e, &mint_authority);  // Set once; metadata.rs enforces immutability

// apps/smart-contracts/contracts/token-factory/src/contract.rs:mint()
let mint_authority = read_mint_authority(&e);
mint_authority.require_auth();
```

## Escrow ID and Metadata Entry Points

### Where Escrow ID, Token Name, and Token Symbol Enter the Flow

1. **User Input**:
   - **Component**: `apps/backoffice-tokenization/src/features/tokens/deploy/dialog/TokenizeEscrow.tsx`
   - **Fields**: `escrowId` (line 51), `tokenName` (line 73), `tokenSymbol` (line 94)
   - **Form Hook**: `useTokenizeEscrow` (line 27)

2. **Service Call**:
   - **File**: `apps/backoffice-tokenization/src/features/tokens/deploy/dialog/useTokenizeEscrow.ts`
   - **Line**: 37-41 - `tokenService.deployToken({ escrowContractId: values.escrowId, tokenName: values.tokenName, tokenSymbol: values.tokenSymbol })`

3. **API Request**:
   - **File**: `apps/backoffice-tokenization/src/features/tokens/services/token.service.ts`
   - **Line**: 31-35 - POST to `/deploy` with `{ escrowContractId, tokenName, tokenSymbol }`

4. **API Handler**:
   - **File**: `apps/backoffice-tokenization/src/app/api/deploy/route.ts` (when backoffice UI is used)
   - **Line**: 9-10 - Extracts `escrowContractId`, `tokenName`, `tokenSymbol` from request body
   - **Line**: 55-60 - Passes to `deployTokenContracts(sorobanClient, { escrowContractId, tokenName, tokenSymbol })`

5. **Deployment Service**:
   - **File**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts`
   - **Line**: 24 - Receives `{ escrowContractId, tokenName, tokenSymbol }` as parameter
   - **Token Sale**: constructor receives `client.nativeAddress(escrowContractId)` (escrow_contract), placeholder sale_token, admin
   - **Token Factory**: constructor receives `client.nativeString(tokenName)`, `client.nativeString(tokenSymbol)`, `client.nativeString(escrowContractId)` (escrow_id), decimal 7, and `client.nativeAddress(tokenSaleAddress)` (mint_authority)

6. **Token Sale Contract**:
   - **File**: `apps/smart-contracts/contracts/token-sale/src/sale.rs`
   - **Constructor**: `__constructor(env, escrow_contract, sale_token, admin)` — stores escrow and token in config, admin for `set_token`
   - **buy()**: Uses `read_config()` to get `escrow_contract` and transfers USDC to escrow

## Token Initialization Arguments

### Token Sale Contract Constructor (deployed first)
**Location**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts` (lines 49-56)

```typescript
const tokenSaleAddress = await client.createContract(
  tokenSaleWasmHash,
  [
    client.nativeAddress(escrowContractId),   // escrow_contract
    client.nativeAddress(client.publicKey),   // sale_token (placeholder)
    client.nativeAddress(client.publicKey),   // admin (deployer; can call set_token)
  ],
  "TokenSale contract creation",
);
```

**Contract Implementation**: `apps/smart-contracts/contracts/token-sale/src/sale.rs:__constructor(env, escrow_contract, sale_token, admin)` (line 55)
- Parameters: `escrow_contract: Address, sale_token: Address, admin: Address`
- Stores config via `write_config(&env, &escrow_contract, &sale_token)` and `write_admin(&env, &admin)`

### Token Factory Contract Constructor (deployed second)
**Location**: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts` (lines 64-71)

```typescript
const tokenFactoryAddress = await client.createContract(
  tokenFactoryWasmHash,
  [
    client.nativeString(tokenName),           // name (user-provided)
    client.nativeString(tokenSymbol),        // symbol (user-provided)
    client.nativeString(escrowContractId),    // escrow_id
    client.nativeU32(7),                     // decimal
    client.nativeAddress(tokenSaleAddress),  // mint_authority (Token Sale contract)
  ],
  "TokenFactory contract creation",
);
```

**Contract Implementation**: `apps/smart-contracts/contracts/token-factory/src/contract.rs:__constructor()` (line 36)
- Parameters: `name: String, symbol: String, escrow_id: String, decimal: u32, mint_authority: Address`
- Writes metadata (name, symbol, decimals), `escrow_id`, and `mint_authority`; all immutable after init

## Quick Reference: Mint Authority and Deployment Order

**To locate mint authority and deployment flow in <2 minutes:**

1. Open: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts`
2. **Order**: Token Sale is deployed first (lines 49-56) with placeholder token and admin; then Token Factory (lines 64-71) with `mint_authority: tokenSaleAddress`; then `set_token` (lines 77-82) to set the real token on Token Sale.
3. Token contract: `apps/smart-contracts/contracts/token-factory/src/contract.rs` — no `set_admin`; `mint_authority` is set in `__constructor()` and in `metadata.rs` (immutable). `mint()` uses `read_mint_authority(&e).require_auth()`.

**Summary**:
- Mint authority: Set once in Token Factory constructor as `tokenSaleAddress`; immutable (no transfer step).
- Token Sale: Gets real token address via `set_token(tokenFactoryAddress)` after Token Factory is deployed (deployer signs as admin).

---

## Standards Gap Analysis: Current Token vs T-REX-Aligned Soroban Fungible Token

### Current Public Methods

#### TokenInterface Methods (Standard Soroban Fungible Token Interface)
All methods are implemented via `impl TokenInterface for Token` in `contract.rs`:

1. **`allowance(e: Env, from: Address, spender: Address) -> i128`** (line 78)
   - Returns the allowance amount for a spender
   - Standard interface method ✅

2. **`approve(e: Env, from: Address, spender: Address, amount: i128, expiration_ledger: u32)`** (line 85)
   - Sets allowance for a spender
   - Emits `Approve` event
   - Standard interface method ✅

3. **`balance(e: Env, id: Address) -> i128`** (line 104)
   - Returns token balance for an address
   - Standard interface method ✅

4. **`transfer(e: Env, from: Address, to_muxed: MuxedAddress, amount: i128)`** (line 111)
   - Transfers tokens from one address to another
   - Supports muxed addresses
   - Emits `Transfer` event
   - Standard interface method ✅

5. **`transfer_from(e: Env, spender: Address, from: Address, to: Address, amount: i128)`** (line 132)
   - Allows approved spender to transfer on behalf of owner
   - Emits `Transfer` event
   - Standard interface method ✅

6. **`burn(e: Env, from: Address, amount: i128)`** (line 154)
   - Burns tokens from an address
   - Emits `Burn` event
   - Standard interface method ✅

7. **`burn_from(e: Env, spender: Address, from: Address, amount: i128)`** (line 167)
   - Burns tokens via allowance
   - Emits `Burn` event
   - Standard interface method ✅

8. **`decimals(e: Env) -> u32`** (line 181)
   - Returns token decimals
   - Metadata getter ✅

9. **`name(e: Env) -> String`** (line 185)
   - Returns token name
   - Metadata getter ✅

10. **`symbol(e: Env) -> String`** (line 189)
    - Returns token symbol
    - Metadata getter ✅

#### Custom Methods (Non-Interface)
Implemented in `impl Token` block:

11. **`__constructor(e: Env, name: String, symbol: String, escrow_id: String, decimal: u32, mint_authority: Address)`** (line 36)
    - Contract constructor
    - Initializes metadata (name, symbol, decimals), immutable `escrow_id` and `mint_authority`
    - Required for contract deployment ✅

12. **`mint(e: Env, to: Address, amount: i128)`** (line 66)
    - Mints new tokens
    - Requires `mint_authority` authorization (read from storage; immutable)
    - Emits `MintWithAmountOnly` event
    - Standard pattern for mintable tokens ✅

13. **`escrow_id(e: Env) -> String`** (line 205, additional `impl Token` block)
    - Returns the immutable escrow contract ID associated with this token (T-REX-aligned metadata getter) ✅

### Comparison Table: Current Token vs Soroban Fungible Token Standard

| Method | Present | Required | Action | Notes |
|--------|---------|----------|--------|-------|
| `allowance` | ✅ | ✅ | **Keep** | Standard interface method |
| `approve` | ✅ | ✅ | **Keep** | Standard interface method |
| `balance` | ✅ | ✅ | **Keep** | Standard interface method |
| `transfer` | ✅ | ✅ | **Keep** | Standard interface method (supports muxed addresses) |
| `transfer_from` | ✅ | ✅ | **Keep** | Standard interface method |
| `burn` | ✅ | ✅ | **Keep** | Standard interface method |
| `burn_from` | ✅ | ✅ | **Keep** | Standard interface method |
| `decimals` | ✅ | ✅ | **Keep** | Standard metadata getter |
| `name` | ✅ | ✅ | **Keep** | Standard metadata getter |
| `symbol` | ✅ | ✅ | **Keep** | Standard metadata getter |
| `__constructor` | ✅ | ✅ | **Keep** | Required for deployment (name, symbol, escrow_id, decimal, mint_authority) |
| `mint` | ✅ | ✅ | **Keep** | Standard pattern; only mint_authority can mint |
| `escrow_id` | ✅ | ✅ | **Keep** | T-REX-aligned immutable metadata getter |

### Metadata Analysis

#### Current Metadata Implementation

**Storage**: Uses `soroban_token_sdk::TokenUtils` which stores metadata in **instance storage** (on-chain)
- **Location**: `apps/smart-contracts/contracts/token-factory/src/metadata.rs`
- **Storage Method**: `util.metadata().set_metadata(&metadata)` (line 21)
- **Retrieval Method**: `util.metadata().get_metadata()` (lines 6, 11, 16)

**Metadata Fields** (standard via TokenUtils):
- `decimal: u32` - Token decimal places
- `name: String` - Token name
- `symbol: String` - Token symbol

**Immutable metadata** (instance storage, set once in constructor):
- `escrow_id: String` - Escrow contract ID (see `metadata.rs`: `DataKey::EscrowId`, `read_escrow_id` / `write_escrow_id`)
- `mint_authority: Address` - Only this address can call `mint` (see `metadata.rs`: `DataKey::MintAuthority`, `read_mint_authority` / `write_mint_authority`)

**Getters**: Standard metadata via TokenInterface; `escrow_id()` in additional `impl Token` block:
- `decimals()` → returns `u32`
- `name()` → returns `String`
- `symbol()` → returns `String`

#### Metadata Getters Status

| Getter | Present | Required | On-Chain Storage | Readable |
|--------|---------|----------|------------------|----------|
| `decimals()` | ✅ | ✅ | ✅ | ✅ |
| `name()` | ✅ | ✅ | ✅ | ✅ |
| `symbol()` | ✅ | ✅ | ✅ | ✅ |
| `escrow_id()` | ✅ | ✅ | ✅ | ✅ (custom getter) |

**Conclusion**: All required metadata getters are present and functional. Metadata (including escrow_id and mint_authority) is stored on-chain; standard fields via TokenUtils, immutable fields in instance storage.

### Non-Standard Patterns Identified

1. **Immutable mint_authority (no set_admin)**
   - **Status**: Mint authority is set once in the constructor and cannot be changed (enforced in `metadata.rs`: `write_mint_authority` panics if already set).
   - **Justification**: Token Sale is the only minter; no admin transfer step. Architecture is deploy Token Sale first, then Token Factory with `mint_authority = tokenSaleAddress`.
   - **Action**: **Keep unchanged** - Required for architecture and security.

2. **Parameterized Metadata (name, symbol)**
   - **Status**: Token name and symbol are provided by the user in the UI and passed through the API to the deployment service and Token Factory constructor.
   - **Location**: `TokenizeEscrow.tsx` (form), `token.service.ts`, `tokenDeploymentService.ts`, `contract.rs:__constructor(name, symbol, ...)`.
   - **Action**: **Already implemented** - No longer hardcoded; decimal remains 7 in deployment.

3. **Token Sale `set_token` (admin-only)**
   - **Status**: Token Sale is deployed with a placeholder token address; after Token Factory is deployed, deployer (admin) calls `set_token(tokenFactoryAddress)` to set the real token. Escrow address is set at construction and not changed.
   - **Action**: **Keep unchanged** - Required for deployment order (Token Sale first, then Token Factory).

### Things That MUST Remain Unchanged

Per master instructions, the following **MUST NOT** be modified:

1. **Token Sale Contract Logic** ❌ **DO NOT MODIFY**
   - File: `apps/smart-contracts/contracts/token-sale/src/sale.rs`
   - The `buy()` function and contract structure must remain unchanged

2. **Mint Authority and Deployment Order** ❌ **DO NOT MODIFY**
   - Token Sale deployed first with placeholder; Token Factory deployed with `mint_authority = tokenSaleAddress`; then `set_token` to set real token on Token Sale.
   - File: `apps/backoffice-tokenization/src/lib/tokenDeploymentService.ts` (and equivalent in investor-tokenization)
   - Token Sale contract **MUST** be the only mint_authority (immutable in Token Factory).

3. **TokenInterface Implementation** ❌ **DO NOT MODIFY**
   - All standard interface methods must remain unchanged
   - File: `apps/smart-contracts/contracts/token-factory/src/contract.rs` (impl TokenInterface and impl Token blocks)
   - This ensures compatibility with wallets and other Soroban tools

4. **Existing Tokenize Escrow Flow** ❌ **DO NOT MODIFY**
   - The deployment sequence and integration points must continue to work
   - Files: All files in the tokenize escrow flow documented above

5. **Metadata Storage Mechanism** ⚠️ **CAN ENHANCE, NOT REPLACE**
   - Current on-chain storage via `TokenUtils` must continue to work
   - Can add additional metadata fields, but existing fields must remain accessible

### Why This Token is T-REX Aligned

This token contract is **already T-REX aligned** for the following reasons:

#### 1. **Standard Interface Compliance** ✅
- Implements the complete Soroban Fungible Token interface (`TokenInterface`)
- All 10 standard methods are present and correctly implemented
- Compatible with Stellar wallets, DEXs, and other Soroban tools

#### 2. **On-Chain Metadata Storage** ✅
- Metadata is stored **on-chain** using Soroban's instance storage
- Uses `soroban_token_sdk::TokenUtils` which is the standard approach
- Storage location: `apps/smart-contracts/contracts/token-factory/src/metadata.rs`
- Storage is persistent and accessible via contract storage reads

#### 3. **Metadata Readability** ✅
- All metadata fields are readable via standard getters:
  - `decimals()` - returns `u32`
  - `name()` - returns `String`
  - `symbol()` - returns `String`
- These getters are part of the standard `TokenInterface`
- Consumed by: `apps/investor-tokenization/src/app/api/token-metadata/route.ts` (name, symbol, decimals)

#### 4. **Metadata Initialization** ✅
- Metadata is provided at initialization via constructor
- Constructor parameters: `name, symbol, escrow_id, decimal, mint_authority`
- Metadata (including escrow_id and mint_authority) is written to on-chain storage at deployment and is immutable
- Location: `contract.rs:__constructor()` (lines 36-62)

#### 5. **Architecture Compliance** ✅
- Mint authority is set at deployment to the Token Sale contract address (immutable)
- Token Sale contract is the only minter; no `set_admin` or transfer step
- Deployment order: Token Sale first → Token Factory with mint_authority → Token Sale updated via `set_token`

#### 6. **Soroban Best Practices** ✅
- Uses standard Soroban SDK patterns
- Proper TTL management for storage entries
- Standard event emissions for all operations
- Proper authorization checks (`require_auth()`); mint only by `mint_authority`

#### 7. **T-REX Tokenization Requirements** ✅
Based on Stellar T-REX tokenization standards:
- ✅ Token implements standard fungible token interface
- ✅ Metadata stored on-chain (name, symbol, decimals via TokenUtils; escrow_id and mint_authority in instance storage)
- ✅ Metadata readable via standard getters plus `escrow_id()`
- ✅ Metadata (and mint_authority) provided at initialization and immutable
- ✅ Compatible with Stellar ecosystem tools
- ✅ Follows Soroban contract conventions

### Gaps and Potential Enhancements

While the token is T-REX aligned, potential enhancements (not requirements) could include:

1. **Additional Metadata Fields** (Optional)
   - Could add fields like `description`, `image_uri`, `documentation_uri`
   - Would require extending metadata (TokenUtils and/or instance storage)
   - **Not required** for T-REX alignment

2. **Decimal Parameterization** (Optional)
   - Currently decimal is fixed to 7 in the deployment service
   - Could be made configurable from the UI like name and symbol
   - **Not required** for T-REX alignment

3. **Metadata Update Mechanism** (Not Recommended)
   - Metadata (including escrow_id and mint_authority) is immutable after initialization
   - T-REX alignment doesn't require mutable metadata
   - **Keep immutable** for security and compliance

### Conclusion

**The current token contract is fully T-REX aligned.** All required elements are present:
- ✅ Standard interface implementation
- ✅ On-chain metadata storage
- ✅ Metadata getters
- ✅ Metadata initialization
- ✅ Architecture compliance

**No breaking changes are required** to achieve T-REX alignment. Any future enhancements should:
- Maintain backward compatibility
- Not modify Token Sale contract logic
- Preserve mint authority assignment flow
- Keep existing tokenize escrow flow functional
