# AbstractARM.sol Detailed Breakdown

## Purpose and Function

`AbstractARM.sol` serves as a generic base contract for Automated Redemption Modules (ARMs). Its primary purpose is to manage liquidity pools where users can deposit a `liquidityAsset` (e.g., WETH) in exchange for LP shares and later redeem those shares back for the `liquidityAsset`. The ARM contract itself acts as an ERC20 token representing these LP shares, inheriting from `ERC20Upgradeable`.

The contract handles the core logic for swaps between the `liquidityAsset` and a `baseAsset` (e.g., a yield-bearing token like stETH). This is managed using configured trade rates (`traderate0`, `traderate1`) and a `crossPrice`, which defines the boundaries for these rates and is used for valuing `baseAsset` held within the ARM.

It features a deposit/redemption queue mechanism (`requestRedeem`, `claimRedeem` with an associated `claimDelay`) to manage liquidity flow and user withdrawals. The ARM also includes functionalities for performance fee collection on generated yield, distributing fees to a designated `feeCollector`.

A key feature is its ability to interact with an `activeMarket` – an external lending market (typically an `IERC4626` compliant vault) – to deploy idle `liquidityAsset` for additional yield generation. The amount of assets kept in the ARM versus deployed to the `activeMarket` is controlled by the `armBuffer`.

The contract is designed to work in conjunction with a `CapManager` contract to enforce deposit caps, ensuring controlled growth and risk management. Furthermore, `AbstractARM` inherits from `OwnableOperable`, providing distinct roles for contract ownership (strategic control) and operational tasks (day-to-day management).

Child contracts are expected to implement specific logic, particularly for interacting with unique external withdrawal queues (e.g., Lido's unstaking process) via the virtual `_externalWithdrawQueue()` function.

## Key State Variables

### Constants
*   `MAX_CROSS_PRICE_DEVIATION`: Maximum allowed deviation for `crossPrice` from `PRICE_SCALE` (1.0), e.g., 0.2%.
*   `PRICE_SCALE`: Decimal scaling for prices (1e36).
*   `MIN_TOTAL_SUPPLY`: Minimum LP shares minted to `DEAD_ACCOUNT` to prevent zero supply issues.
*   `DEAD_ACCOUNT`: Address for initial mint of `MIN_TOTAL_SUPPLY`.
*   `FEE_SCALE`: Decimal scaling for performance fee (10000 = 100%).

### Immutable Variables
*   `minSharesToRedeem`: The minimum amount of shares that can be redeemed from the `activeMarket`.
*   `allocateThreshold`: Minimum difference in liquidity to trigger `allocate()` to/from `activeMarket`, preventing frequent small adjustments.
*   `liquidityAsset`: (IERC20) The token used for deposits, withdrawals, and as the quote asset in trades (e.g., WETH).
*   `baseAsset`: (IERC20) The yield-bearing or other token being managed/traded against `liquidityAsset` (e.g., stETH).
*   `token0`: (IERC20) One token in the trading pair, typically `liquidityAsset` or `baseAsset`.
*   `token1`: (IERC20) The other token in the trading pair.
*   `claimDelay`: The delay (in seconds) before a redemption request (`requestRedeem`) can be finalized (`claimRedeem`).

### Storage Variables
*   `traderate0`: Exchange rate for `token0` to `token1`. If `token0` is WETH and `token1` is stETH, this is WETH/stETH rate from ARM's perspective (ARM sells `token1`). Scaled to `PRICE_SCALE`.
*   `traderate1`: Exchange rate for `token1` to `token0`. If `token0` is WETH and `token1` is stETH, this is stETH/WETH rate from ARM's perspective (ARM buys `token1`). Scaled to `PRICE_SCALE`.
*   `crossPrice`: A pivotal price (scaled to `PRICE_SCALE`) that `traderate0` (adjusted) and `traderate1` cannot cross. `baseAsset` held by the ARM is valued at this price for `totalAssets()` calculation.
*   `withdrawsQueued`: Cumulative total of `liquidityAsset` amount requested for withdrawal via `requestRedeem`.
*   `withdrawsClaimed`: Cumulative total of `liquidityAsset` amount already claimed via `claimRedeem`.
*   `nextWithdrawalIndex`: Counter for new withdrawal requests, used as `requestId`.
*   `withdrawalRequests`: `mapping(uint256 requestId => WithdrawalRequest struct)` storing details of each user's redemption request (withdrawer, amount, claim timestamp, etc.).
*   `fee`: Performance fee percentage charged on asset appreciation, in basis points (e.g., 500 = 5%). Max 50%.
*   `lastAvailableAssets`: Stores the `availableAssets` value from the last time fees were collected or adjusted. Used to calculate new asset increase for fee accrual.
*   `feeCollector`: Address that receives collected performance fees.
*   `capManager`: Address of the `ICapManager` contract used for deposit cap enforcement.
*   `activeMarket`: Address of the active `IERC4626` compliant lending market where idle `liquidityAsset` is deployed.
*   `supportedMarkets`: `mapping(address market => bool supported)` indicating which lending markets can be set as `activeMarket`.
*   `armBuffer`: Percentage (scaled to 1e18, where 1e18 = 100%) of available liquid assets that should be kept within the ARM contract itself and not deployed to the `activeMarket`.

## Key Functions

### Initialization
*   `constructor(...)`: Sets immutable variables like `token0`, `token1`, `liquidityAsset`, `claimDelay`.
*   `_initARM(address _operator, string _name, string _symbol, uint256 _fee, address _feeCollector, address _capManager)`: Internal initializer called by proxy setup. Sets operator, LP token name/symbol, initial fee parameters, cap manager, and mints `MIN_TOTAL_SUPPLY` to `DEAD_ACCOUNT`.

### Swapping Logic
*   `swapExactTokensForTokens(...)`: Allows traders to swap a known amount of input tokens for an estimated amount of output tokens, based on `traderate0` or `traderate1`.
*   `swapTokensForExactTokens(...)`: Allows traders to swap an estimated amount of input tokens for a known amount of output tokens.
*   `setPrices(uint256 buyT1, uint256 sellT1)`: (Operator/Owner) Sets `traderate0` and `traderate1`. Prices are set from the perspective of `token1` relative to `token0`.
*   `setCrossPrice(uint256 newCrossPrice)`: (Owner) Sets the `crossPrice`. Requires specific conditions if lowering the price to prevent losses on existing `baseAsset` inventory.

### Liquidity Provision (LP)
*   `deposit(uint256 assets)` / `deposit(uint256 assets, address receiver)`: Allows users to deposit `liquidityAsset` and mints LP shares (the ARM token itself) to the user or specified receiver. Interacts with `capManager`.
*   `requestRedeem(uint256 shares)`: Allows users to burn their LP shares to request a future withdrawal of `liquidityAsset`. Creates a `WithdrawalRequest` and queues it.
*   `claimRedeem(uint256 requestId)`: Allows users to claim their `liquidityAsset` from a completed `WithdrawalRequest` after the `claimDelay` has passed. May trigger withdrawal from `activeMarket` if ARM internal balance is insufficient.
*   `previewDeposit(uint256 assets)`: View function to estimate shares minted for a given asset amount.
*   `previewRedeem(uint256 shares)`: View function to estimate assets received for a given share amount.

### Asset Management & Valuation
*   `totalAssets()`: View function returning the total value of assets managed by the ARM. This considers `liquidityAsset` in the ARM, assets in the `activeMarket`, `baseAsset` in the ARM (valued at `crossPrice`), and assets in the `_externalWithdrawQueue()`, less accrued fees.
*   `convertToShares(uint256 assets)`: Calculates how many LP shares correspond to a given amount of `liquidityAsset` based on `totalAssets()` and `totalSupply()`.
*   `convertToAssets(uint256 shares)`: Calculates how much `liquidityAsset` corresponds to a given amount of LP shares.
*   `_availableAssets()`: Internal view function to calculate assets available for yield generation or withdrawal, excluding amounts reserved for the ARM's internal withdrawal queue.
*   `_externalWithdrawQueue()`: **Virtual function** intended to be implemented by child contracts. It should return the amount of `baseAsset` currently in an external withdrawal process specific to that `baseAsset` (e.g., Lido's unstaking queue).

### Fee Mechanism
*   `collectFees()`: Public function that calculates accrued performance fees based on asset appreciation since `lastAvailableAssets` and transfers them to the `feeCollector`. Updates `lastAvailableAssets`.
*   `feesAccrued()`: View function to see currently accrued fees without collecting them.
*   `_feesAccrued()`: Internal view function that calculates fee amounts and the current `newAvailableAssets`.
*   `setFee(uint256 _fee)`: (Owner) Sets the performance `fee` percentage.
*   `setFeeCollector(address _feeCollector)`: (Owner) Sets the `feeCollector` address.

### Lending Market Interaction
*   `addMarkets(address[] calldata _markets)`: (Owner) Adds market addresses to `supportedMarkets`.
*   `removeMarket(address _market)`: (Owner) Removes a market from `supportedMarkets` (cannot be the `activeMarket`).
*   `setActiveMarket(address _market)`: (Operator/Owner) Sets a new `activeMarket`. If there was a previous `activeMarket`, it withdraws all assets from it. Triggers `_allocate()`.
*   `allocate()`: Public function that triggers `_allocate()`.
*   `_allocate()`: Internal function that rebalances `liquidityAsset` between the ARM contract and the `activeMarket` based on the `armBuffer` percentage and `availableAssets`. If ARM has excess liquidity, it deposits into `activeMarket`. If ARM has a deficit, it withdraws.
*   `setARMBuffer(uint256 _armBuffer)`: (Owner) Sets the `armBuffer` percentage.

### Admin & Configuration
*   `setCapManager(address _capManager)`: (Owner) Sets the `capManager` contract address.

## Interactions

`AbstractARM.sol` interacts with several types of external contracts and actors:

*   **Users:**
    *   Call `deposit()` to provide liquidity.
    *   Call `requestRedeem()` and `claimRedeem()` to withdraw liquidity.
    *   Call `swapExactTokensForTokens()` or `swapTokensForExactTokens()` to trade.
    *   Call view functions like `previewDeposit()`, `previewRedeem()`, `totalAssets()`.
*   **Owner (via `OwnableOperable`):**
    *   Calls administrative functions like `setFee()`, `setFeeCollector()`, `setCapManager()`, `setCrossPrice()`, `addMarkets()`, `removeMarket()`, `setARMBuffer()`.
    *   Can also call operator functions.
*   **Operator (via `OwnableOperable`):**
    *   Calls operational functions like `setPrices()`, `setActiveMarket()`, `allocate()`.
*   **`ICapManager` (`capManager` variable):**
    *   The ARM calls `ICapManager(capManager).postDepositHook()` after a deposit to inform the cap manager, allowing it to enforce caps.
*   **`IERC4626` (`activeMarket` variable):**
    *   Represents an external yield-bearing vault/market.
    *   ARM calls `deposit()` on `activeMarket` to lend `liquidityAsset`.
    *   ARM calls `withdraw()` or `redeem()` on `activeMarket` to retrieve `liquidityAsset`.
    *   ARM calls `maxWithdraw()`, `previewRedeem()`, `balanceOf()` (shares in market), `asset()` (underlying asset of market) for querying market state.
*   **`IERC20` (for `liquidityAsset`, `baseAsset`, `token0`, `token1`):**
    *   Uses `transferFrom()` to pull tokens from users during `deposit()` and swaps.
    *   Uses `transfer()` to send tokens to users during `claimRedeem()` and swaps, and to `feeCollector`.
    *   Uses `balanceOf()` to check token balances of the ARM contract.
    *   Uses `approve()` to allow `activeMarket` to pull `liquidityAsset` during `_allocate()`.
*   **Child Contracts (Specific ARMs like `LidoARM`, `OethARM`):**
    *   Expected to inherit from `AbstractARM`.
    *   **Must implement** the `_externalWithdrawQueue()` internal view function to provide the amount of `baseAsset` that is currently in that asset's specific external withdrawal queue (e.g., ETH being unstaked from Lido). This value is crucial for accurate `totalAssets()` calculation.
    *   May implement other specific logic related to their unique `baseAsset`.

## Mermaid Diagram

```mermaid
graph TD
    subgraph AbstractARM [AbstractARM.sol]
        direction LR

        subgraph StateVars [Key State Variables]
            direction TB
            sv_liquidityAsset["liquidityAsset (IERC20)"]
            sv_baseAsset["baseAsset (IERC20)"]
            sv_token0["token0 (IERC20)"]
            sv_token1["token1 (IERC20)"]
            sv_traderates["traderate0 / traderate1"]
            sv_crossPrice["crossPrice"]
            sv_withdrawalRequests["withdrawalRequests (mapping)"]
            sv_fee["fee / feeCollector"]
            sv_capManagerAddr["capManager (address)"]
            sv_activeMarketAddr["activeMarket (address)"]
            sv_armBuffer["armBuffer"]
        end

        subgraph Functions [Key Functions]
            direction TB
            func_deposit["deposit()"]
            func_requestRedeem["requestRedeem()"]
            func_claimRedeem["claimRedeem()"]
            func_swap["swapExactTokensForTokens()"]
            func_setPrices["setPrices()"]
            func_setCrossPrice["setCrossPrice()"]
            func_allocate["allocate()"]
            func_collectFees["collectFees()"]
            func_totalAssets["totalAssets()"]
            func_externalWithdrawQueue["_externalWithdrawQueue() (virtual)"]
        end

        subgraph ERC20LP [LP Shares (ERC20Upgradeable)]
            erc20_mint["_mint()"]
            erc20_burn["_burn()"]
            erc20_totalSupply["totalSupply()"]
        end

        subgraph OwnableOp [OwnableOperable]
            own_owner["owner()"]
            own_operator["operator()"]
        end

        StateVars --> Functions
        Functions --> ERC20LP
    end

    subgraph ExternalInteractions [External Interactions]
        direction TB
        User["User"]
        CapManager["CapManager (ICapManager)"]
        ActiveMarket["ActiveMarket (IERC4626)"]
        LiquidityAssetToken["liquidityAsset (IERC20)"]
        BaseAssetToken["baseAsset (IERC20)"]
        Token0Instance["token0 (IERC20)"]
        Token1Instance["token1 (IERC20)"]
        Owner["Owner (Role)"]
        Operator["Operator (Role)"]
        ChildContract["Child ARM Contract (e.g. LidoARM)"]
    end

    User -- "deposit, requestRedeem, claimRedeem, swap" --> AbstractARM
    AbstractARM -- "call: postDepositHook" --> CapManager
    AbstractARM -- "call: deposit, withdraw, redeem, etc." --> ActiveMarket
    AbstractARM -- "call: transferFrom, transfer, balanceOf" --> LiquidityAssetToken
    AbstractARM -- "call: balanceOf" --> BaseAssetToken %% baseAsset is mostly held, less direct calls out unless specific ARM logic
    AbstractARM -- "Internal Use" --> Token0Instance
    AbstractARM -- "Internal Use" --> Token1Instance
    Owner -- "call: setCrossPrice, setFee, setCapManager, etc." --> AbstractARM
    Operator -- "call: setPrices, setActiveMarket, allocate, etc." --> AbstractARM
    ChildContract -- "implements: _externalWithdrawQueue()" --> AbstractARM


    %% Link state vars to actual external contracts for clarity
    sv_capManagerAddr -.-> CapManager
    sv_activeMarketAddr -.-> ActiveMarket
    sv_liquidityAsset -.-> LiquidityAssetToken
    sv_baseAsset -.-> BaseAssetToken
    sv_token0 -.-> Token0Instance
    sv_token1 -.-> Token1Instance


    %% Indicate inheritance
    AbstractARM -. "inherits" .-> ERC20LP
    AbstractARM -. "inherits" .-> OwnableOp
    ChildContract -. "inherits" .-> AbstractARM


    classDef armInternals fill:#lightblue,stroke:#333,stroke-width:2px;
    class AbstractARM armInternals;
    classDef external fill:#lightgrey,stroke:#333,stroke-width:2px;
    class User, CapManager, ActiveMarket, LiquidityAssetToken, BaseAssetToken, Token0Instance, Token1Instance, Owner, Operator, ChildContract external;
```

This document provides a comprehensive overview of the `AbstractARM.sol` contract, its functionalities, key components, and interactions within the broader protocol ecosystem.
