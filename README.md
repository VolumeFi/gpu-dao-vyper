# GPU DAO Smart Contract

A Vyper smart contract for managing GPU DAO token purchases and cross-chain operations through the Paloma network.

## Contract Overview

The GPU DAO contract facilitates token purchases with cross-chain bridging capabilities. Users can purchase tokens using various assets, which are then deposited into a PUSD manager and bridged to the Paloma network.

## Contract Details

- **Language**: Vyper 0.4.1
- **License**: Apache-2.0
- **Author**: Volume.finance
- **EVM Version**: Cancun
- **Gas Optimization**: Enabled

## State Variables

### Immutable Variables
- `pusd_manager`: Address of the PUSD manager contract
- `WETH9`: Address of WETH9 token contract
- `pgwt`: Address of the PGWT token contract
- `pgwt_amount`: Fixed amount of PGWT tokens required per purchase
- `purchase_limit`: Maximum contribution limit per user

### Mutable Variables
- `compass`: Address of the Compass contract for cross-chain operations
- `refund_wallet`: Address that receives gas fees
- `gas_fee`: Gas fee amount in wei
- `service_fee_collector`: Address that receives service fees
- `service_fee`: Service fee percentage (basis points)
- `paloma`: Paloma network identifier (bytes32)
- `contributions`: Mapping of user addresses to their total contributions
- `send_nonces`: Mapping of nonces to prevent replay attacks

## Function Documentation

### Constructor

```vyper
@deploy
def __init__(_compass: address, _pusd_manager: address, _weth9: address, _pgwt: address,
             _pgwt_amount: uint256, _purchase_limit: uint256, _refund_wallet: address,
             _gas_fee: uint256, _service_fee_collector: address, _service_fee: uint256)
```

**Purpose**: Initializes the contract with all required parameters.

**Security Considerations**:
- Validates that `_compass` is not the zero address
- Ensures `_service_fee` is less than `DENOMINATOR` (100%)
- Emits events for all parameter updates for transparency

**Usage Example**:
```python
# Deploy with parameters
constructor_params = [
    compass_address,
    pusd_manager_address,
    weth9_address,
    pgwt_address,
    pgwt_amount,
    purchase_limit,
    refund_wallet,
    gas_fee,
    service_fee_collector,
    service_fee
]
contract = GPU_DAO.deploy(*constructor_params, sender=deployer)
```

### Internal Helper Functions

#### `_safe_approve(_token: address, _to: address, _value: uint256)`

**Purpose**: Safely approves token spending with error handling.

**Security Considerations**:
- Uses `default_return_value=True` to handle non-standard ERC20 tokens
- Reverts on failed approvals to prevent silent failures

#### `_safe_transfer(_token: address, _to: address, _value: uint256)`

**Purpose**: Safely transfers tokens with error handling.

**Security Considerations**:
- Uses `default_return_value=True` for non-standard ERC20 compatibility
- Reverts on failed transfers

#### `_safe_transfer_from(_token: address, _from: address, _to: address, _value: uint256)`

**Purpose**: Safely transfers tokens from one address to another with error handling.

**Security Considerations**:
- Uses `default_return_value=True` for non-standard ERC20 compatibility
- Reverts on failed transfers

### External Functions

#### `purchase(path: Bytes[204], amount: uint256, min_amount: uint256 = 0)`

**Purpose**: Allows users to purchase tokens by providing assets and receiving PUSD tokens.

**Function Flow**:
1. Transfers required PGWT tokens from user to contract
2. Handles gas fee deduction and refund
3. Determines the source token (from path or PUSD manager asset)
4. Processes token deposit (WETH or ERC20)
5. Calculates and transfers service fees
6. Approves and deposits tokens to PUSD manager
7. Updates user contribution and checks purchase limit
8. Sends PGWT tokens to Paloma via Compass
9. Emits Purchase event

**Security Considerations**:
- **Access Control**: No restrictions - any user can call
- **Reentrancy**: No external calls after state changes
- **Input Validation**: Validates `min_amount` when path is provided
- **Overflow Protection**: Uses safe math operations
- **Purchase Limit**: Enforces per-user contribution limits

**Usage Example**:
```python
# Purchase with ETH (WETH)
contract.purchase(
    path=b"",  # Empty path for default asset
    amount=1e18,  # 1 ETH
    min_amount=0,
    value=1e18,  # Send 1 ETH
    sender=user
)

# Purchase with custom token
contract.purchase(
    path=token_address + swap_path,  # Token address + swap path
    amount=1000e6,  # 1000 USDC
    min_amount=950e18,  # Minimum PUSD received
    sender=user
)
```

#### `claim()`

**Purpose**: Allows users to claim rewards or refunds.

**Function Flow**:
1. Handles gas fee deduction and refund
2. Returns any excess ETH to the caller
3. Emits Claim event

**Security Considerations**:
- **Access Control**: No restrictions - any user can call
- **Gas Fee**: Deducts gas fee if configured
- **ETH Refund**: Returns excess ETH to caller

**Usage Example**:
```python
# Claim with gas fee
contract.claim(value=gas_fee, sender=user)
```

#### `update_compass(new_compass: address)`

**Purpose**: Updates the Compass contract address.

**Security Considerations**:
- **Access Control**: Only current compass can call
- **SLC Check**: Verifies SLC is available before update
- **Event Emission**: Logs the update for transparency

**Usage Example**:
```python
# Only callable by current compass
contract.update_compass(new_compass_address, sender=current_compass)
```

#### `set_paloma()`

**Purpose**: Sets the Paloma network identifier.

**Security Considerations**:
- **Access Control**: Only compass can call
- **One-time Setup**: Can only be set once (when paloma is empty)
- **Data Validation**: Validates message data length and extracts paloma ID

**Usage Example**:
```python
# Set paloma ID (called by compass)
paloma_id = b"paloma_network_id_32_bytes_long"
contract.set_paloma(data=paloma_id, sender=compass)
```

#### `update_refund_wallet(new_refund_wallet: address)`

**Purpose**: Updates the refund wallet address.

**Security Considerations**:
- **Access Control**: Only compass with valid paloma signature
- **Event Emission**: Logs the update

**Usage Example**:
```python
# Update refund wallet (called by compass with paloma signature)
contract.update_refund_wallet(new_wallet, sender=compass)
```

#### `update_gas_fee(new_gas_fee: uint256)`

**Purpose**: Updates the gas fee amount.

**Security Considerations**:
- **Access Control**: Only compass with valid paloma signature
- **Event Emission**: Logs the update

**Usage Example**:
```python
# Update gas fee (called by compass with paloma signature)
contract.update_gas_fee(new_gas_fee, sender=compass)
```

#### `update_service_fee_collector(new_service_fee_collector: address)`

**Purpose**: Updates the service fee collector address.

**Security Considerations**:
- **Access Control**: Only compass with valid paloma signature
- **Event Emission**: Logs the update

**Usage Example**:
```python
# Update service fee collector (called by compass with paloma signature)
contract.update_service_fee_collector(new_collector, sender=compass)
```

#### `update_service_fee(new_service_fee: uint256)`

**Purpose**: Updates the service fee percentage.

**Security Considerations**:
- **Access Control**: Only compass with valid paloma signature
- **Input Validation**: Ensures fee is less than 100%
- **Event Emission**: Logs the update

**Usage Example**:
```python
# Update service fee (called by compass with paloma signature)
contract.update_service_fee(new_fee, sender=compass)
```

### Internal Functions

#### `_paloma_check()`

**Purpose**: Validates that the caller is the compass and has a valid paloma signature.

**Security Considerations**:
- **Access Control**: Only compass can pass
- **Signature Validation**: Verifies paloma signature in message data
- **Reused Logic**: Centralized validation for paloma-authorized functions

## Events

### `Purchase`
- **sender**: Address of the purchaser (indexed)
- **from_token**: Address of the token used for purchase
- **amount**: Amount of tokens purchased
- **pusd_amount**: Amount of PUSD tokens received

### `Claim`
- **sender**: Address of the claimant (indexed)

### `UpdateCompass`
- **old_compass**: Previous compass address
- **new_compass**: New compass address

### `UpdateRefundWallet`
- **old_refund_wallet**: Previous refund wallet address
- **new_refund_wallet**: New refund wallet address

### `SetPaloma`
- **paloma**: Paloma network identifier

### `UpdateGasFee`
- **old_gas_fee**: Previous gas fee amount
- **new_gas_fee**: New gas fee amount

### `UpdateServiceFeeCollector`
- **old_service_fee_collector**: Previous service fee collector address
- **new_service_fee_collector**: New service fee collector address

### `UpdateServiceFee`
- **old_service_fee**: Previous service fee percentage
- **new_service_fee**: New service fee percentage

## Security Considerations for Auditors

### Access Control
- **Public Functions**: `purchase()` and `claim()` are unrestricted
- **Compass-Only**: `update_compass()`, `set_paloma()` require compass authorization
- **Paloma-Authorized**: Update functions require compass + paloma signature validation

### Reentrancy Protection
- No external calls after state changes in critical functions
- Safe token transfer patterns used throughout

### Input Validation
- Address validation in constructor
- Service fee bounds checking
- Purchase limit enforcement
- Message data validation for paloma operations

### Token Safety
- Safe ERC20 transfer patterns with `default_return_value=True`
- Balance checks before and after transfers
- Proper approval management

### Economic Considerations
- Gas fee collection mechanism
- Service fee calculation and distribution
- Purchase limit per user
- Excess ETH refund handling

## Testing

This project uses the Ape Framework for testing. To run tests:

### Prerequisites
```bash
# Install Ape Framework
pip install eth-ape

# Install Vyper compiler
pip install vyper
```

### Running Tests
```bash
# Run all tests
ape test

# Run tests with verbose output
ape test -v

# Run specific test file
ape test tests/test_gpu_dao.py

# Run tests with coverage
ape test --coverage
```

### Test Structure
Create test files in a `tests/` directory with the following structure:

```python
# tests/test_gpu_dao.py
import pytest
from ape import accounts, Contract

def test_constructor():
    # Test constructor parameters
    pass

def test_purchase():
    # Test purchase functionality
    pass

def test_claim():
    # Test claim functionality
    pass

def test_admin_functions():
    # Test admin/compass functions
    pass
```

## Deployment

### Using Ape Framework
```bash
# Deploy to local network
ape run deploy

# Deploy to testnet
ape run deploy --network ethereum:goerli

# Deploy to mainnet
ape run deploy --network ethereum:mainnet
```

### Deployment Script Example
```python
# scripts/deploy.py
from ape import accounts, Contract

def main():
    deployer = accounts.load("deployer")
    
    # Deploy GPU DAO
    gpu_dao = Contract.deploy(
        compass_address,
        pusd_manager_address,
        weth9_address,
        pgwt_address,
        pgwt_amount,
        purchase_limit,
        refund_wallet,
        gas_fee,
        service_fee_collector,
        service_fee,
        sender=deployer
    )
    
    print(f"GPU DAO deployed at: {gpu_dao.address}")
```

## Dependencies

### External Contracts
- **ERC20**: Standard ERC20 token interface
- **PusdManager**: PUSD management contract
- **Weth**: WETH9 wrapper contract
- **Compass**: Cross-chain bridge contract

### Constants
- `DENOMINATOR`: 10^18 (for fee calculations)

## License

Apache-2.0 License - see LICENSE file for details. 