# conrail dHYP–fHYP Smart Contract Specification

## Overview

This repository implements a two-asset smart contract system with constrained transfer semantics.

- **fHYP** — a fixed-supply fungible token minted once at deployment and never mintable again.
- **dHYP** — a collateralized synthetic token minted and burned against **STRK** via **binding quotes**.  
  It can be transferred **only** between *registered Starknet wallets* and only through a **two-leg CPMM path**:  
  **dHYP → fHYP → dHYP**, executed atomically inside a single transaction.

There is **no other external demand or market** for dHYP besides these transfers.

-----

## Roles

|Role                 |Description                                                                      |
|---------------------|---------------------------------------------------------------------------------|
|**Deployer**         |Deploys and initializes the contract, sets immutable parameters.                 |
|**Registered Wallet**|Starknet wallet authorized to mint/burn/transfer dHYP via collateralization.     |
|**Recipient**        |Another registered wallet receiving dHYP.                                        |
|**Quoter (optional)**|Off-chain signer providing **binding STRK↔dHYP quotes** for mint/burn operations.|

-----

## Assets

### fHYP (Fixed Token)

- Fungible token (ERC-20/ARC-20 compatible).
- Minted **once** at deployment, total supply immutable.
- Freely transferable.
- Used as the **intermediate** asset in dHYP transfers.

### dHYP (Collateralized Token)

- Fungible interface with **restricted transfers**:
  - Only transferable through `transfer_dHYP_via_pool()`.
  - Direct `transfer()` or `transferFrom()` calls revert.
- Minted by **depositing STRK** with a **binding quote**.
- Burned when collateral is withdrawn.
- Can be held **only** by registered wallets.

-----

## Dependencies

- **Starknet wallet registry** — to verify wallet registration.
- **STRK token interface** — to handle collateral deposits and withdrawals.
- **Quote verification** — off-chain signed binding quotes verified on-chain (ECDSA or Cairo-native).

-----

## Invariants

- `fHYP` total supply fixed after initialization.
- Only registered wallets may hold or receive `dHYP`.
- `dHYP` mint amount ≤ STRK collateral × quote price.
- All `dHYP` transfers must follow the **two-leg CPMM** path:
1. dHYP → fHYP swap
2. fHYP → dHYP swap
- CPMM invariant:  
  $$
  (x + e_1) (y + e_2) = k
  $$
  where `x`, `y` are reserves (dHYP, fHYP), and $e₁$, $e₂$ are transaction deltas.

-----

## Storage Layout

```text
struct Reserves {
    uint256 X_dHYP;
    uint256 Y_fHYP;
}

bool     initialized;
address  deployer;

// Registration
mapping(address => bool) isRegistered;

// fHYP
string   fHYP_name;
string   fHYP_symbol;
uint8    fHYP_decimals;
uint256  fHYP_totalSupply;
mapping(address => uint256) fHYP_balance;

// dHYP
string   dHYP_name;
string   dHYP_symbol;
uint8    dHYP_decimals;
uint256  dHYP_totalSupply;
mapping(address => uint256) dHYP_balance;

// STRK Collateral
address  STRK_token;
mapping(address => uint256) strk_collateral;
mapping(bytes32 => bool) usedQuoteNonce;

// CPMM & Fees
Reserves reserves;
uint16   swapFeeBps;   // per-leg fee (0-10000)
address  feeRecipient;

// Admin
address  pauser;
bool     paused;
```

-----

## Constant-Product Formula

For one swap leg:

$$
\text{out} = \frac{R_\text{out} \cdot (\Delta_\text{in} (1 - \tau))}{R_\text{in} + \Delta_\text{in} (1 - \tau)}
$$

where:

- $\tau$ = swap fee fraction (e.g., 0.003 for 0.3%)
- $R_\text{in}$, $R_\text{out}$ are reserves before trade

In your notation:

$$
(x + e_1)(y + e_2) = k
$$

### Two-leg Transfer

1. Leg-1: dHYP→fHYP (sender → pool)
2. Leg-2: fHYP→dHYP (pool → recipient)

If swapFeeBps = 0, reserves return to original;  
if >0, X_dHYP increases slightly over time → dHYP becomes cheaper vs fHYP.

-----

## Public Interfaces

### Initialization

`initialize(config)`

Sets metadata, STRK address, fee, and fHYP total supply.  
Marks the contract as initialized.

-----

### Registration

`registerWallet(starknetAddress, proof)`  
`unregisterWallet()`

Registers/unregisters Starknet wallets.  
Unregistration allowed only with zero dHYP balance.

-----

### Collateralization & Quotes

`mint_dHYP_with_STRK(Quote q, uint256 strkIn, uint256 minDhypOut)`  
`burn_dHYP_for_STRK(Quote q, uint256 dhypIn, uint256 minStrkOut)`

- Verify quote (price, slippage, nonce, expiry).
- Adjust collateral and total supply.
- Emit events `MintedDhyp`, `BurnedDhyp`.

-----

### Transfer via Pool

`transfer_dHYP_via_pool(address to, uint256 dx_in, uint256 min_dx_out, uint256 deadline, uint256 nonce)`

Executes the atomic two-leg CPMM swap:

1. dHYP → fHYP
2. fHYP → dHYP

Reverts if:

- Caller or recipient not registered
- Deadline expired
- Nonce reused
- Slippage > tolerance

Emits `DhypTransferViaPool`.

-----

## Events

- `Initialized(address deployer, uint256 fHYP_totalSupply, uint16 swapFeeBps)`
- `WalletRegistered(address account, bytes starknetBindingData)`
- `MintedDhyp(address account, uint256 strkIn, uint256 dhypOut, bytes32 quoteHash)`
- `BurnedDhyp(address account, uint256 dhypIn, uint256 strkOut, bytes32 quoteHash)`
- `DhypTransferViaPool(address from, address to, uint256 dx_in, uint256 dx_out, uint256 dy_tmp, uint16 feeBps)`
- `Paused(address by)`
- `Unpaused(address by)`

-----

## Error Codes

|Error                        |Meaning                          |
|-----------------------------|---------------------------------|
|`NotInitialized()`           |Contract not yet initialized     |
|`Paused()`                   |Contract paused                  |
|`NotRegistered()`            |Caller not registered            |
|`DeadlineExpired()`          |Transaction expired              |
|`NonceUsed()`                |Nonce already used               |
|`QuoteExpired()`             |Quote deadline passed            |
|`RestrictedTransfer()`       |Direct dHYP transfer attempt     |
|`SlippageTooHigh()`          |Output < minimum expected        |
|`FixedSupply()`              |Attempt to mint more fHYP        |
|`WithdrawExceedsCollateral()`|Withdrawal exceeds deposited STRK|

-----

## Security

- Non-reentrancy guard on state-changing functions.
- Check-effects-interactions order.
- Nonce replay protection for quotes & transfers.
- Circuit breaker (`pause()` / `unpause()`).
- Strict registration for dHYP ownership.

-----

## Economic Behavior

|Mode                   |Effect on Price                   |Notes                        |
|-----------------------|----------------------------------|-----------------------------|
|Zero Fee (swapFeeBps=0)|No change — price stable          |Recommended default          |
|Fee > 0                |dHYP price drifts downward vs fHYP|Drift ≈ 2τ * trade_volume / X|

### Mitigations

- Use protocol-level fee instead of CPMM fee.
- Batch transfers to minimize drift.
- Deep liquidity reserves.

-----

## Example Swap Function

```solidity
function cpmm_swap_in(
    uint256 Rin,
    uint256 Rout,
    uint256 amountIn,
    uint16 feeBps
) internal pure returns (uint256 amountOut, uint256 RinNew, uint256 RoutNew) {
    uint256 effIn = amountIn * (10000 - feeBps) / 10000;
    amountOut = (Rout * effIn) / (Rin + effIn);
    RinNew = Rin + effIn;
    RoutNew = Rout - amountOut;
}
```

-----

## Test Scenarios

1. ✅ Initialize → fHYP supply fixed
2. ✅ Register wallets
3. ✅ Mint/Burn dHYP with valid quotes
4. ✅ dHYP transfer via pool (fee=0) keeps price stable
5. ✅ Fee>0 causes expected reserve drift
6. ✅ Direct dHYP transfer reverts
7. ✅ Pause/unpause enforcement
8. ✅ Quote expiry, nonce replay prevention

-----

## Deployment Checklist

1. Deploy contract.
2. `initialize()` with:
- fHYP metadata & immutable supply
- STRK token address
- swapFeeBps = 0 (recommended)
3. Fund fHYP reserves (seed liquidity).
4. Register wallets.
5. Mint dHYP via STRK quotes.
6. Perform transfers between registered accounts.

-----

## License

MIT © OnEdge Network
