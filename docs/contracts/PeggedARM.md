# PeggedARM.sol Detailed Breakdown

## Purpose and Function

`PeggedARM.sol` is an **abstract contract** designed as an extension of `AbstractARM.sol`. It is not intended for direct deployment but serves as a foundational layer for concrete Automated Redemption Module (ARM) implementations that require a strict 1:1 pegged swap functionality between two tokens. These tokens are referred to as `token0` and `token1` as defined in the `AbstractARM` contract (where one of them is typically the `liquidityAsset` and the other is the `baseAsset`).

The primary purpose of `PeggedARM` is to override the standard, potentially variable-rate swapping logic found in `AbstractARM` and replace it with a straightforward 1:1 exchange mechanism. When a child contract inherits from `PeggedARM`, any swap operations between its designated `token0` and `token1` will result in an equal amount of the output token being exchanged for the input token. This effectively bypasses the `traderate0`, `traderate1`, and `crossPrice` mechanisms of `AbstractARM` for these specific pegged swaps, simplifying operations for assets expected to maintain a hard peg.

The directionality of the 1:1 swap can be configured at the deployment time of the child contract:
*   **One-way Pegged Swap:** Swaps are permitted only from `token0` to `token1`.
*   **Two-way Pegged Swap (Both Directions):** Swaps are permitted from `token0` to `token1` and also from `token1` to `token0`.

This contract is particularly useful for creating ARMs that manage pairs of assets like stablecoin-to-stablecoin or a liquid staking derivative to its underlying asset where a 1:1 exchange rate is the desired behavior (e.g., OETH for WETH in some contexts).

## Key State Variables & Functions

### Immutable State Variables
*   `bothDirections (bool)`: A boolean flag that is set immutably during the construction of a child contract that inherits from `PeggedARM`.
    *   If `true`, the 1:1 pegged swap functionality is enabled for both directions: `token0` can be swapped for `token1`, and `token1` can be swapped for `token0`.
    *   If `false`, the 1:1 pegged swap is restricted to one direction only: `token0` can be swapped for `token1`.

### Constructor
*   `constructor(bool _bothDirections)`: This constructor is called by the child ARM contract during its own deployment. It initializes the `bothDirections` immutable variable based on the boolean argument passed, thereby configuring the swap directionality for that specific ARM instance.

### Overridden Internal Functions (from `AbstractARM.sol`)
These functions are part of `AbstractARM`'s internal swap execution flow. `PeggedARM` overrides them to implement its 1:1 logic.
*   `_swapExactTokensForTokens(IERC20 inToken, IERC20 outToken, uint256 amountIn, address to) internal override returns (uint256 amountOut)`:
    *   This function overrides the virtual function in `AbstractARM.sol`.
    *   Instead of calculating `amountOut` based on `traderate0` or `traderate1`, it delegates the operation to the internal `_swap` function of `PeggedARM`.
    *   In a 1:1 swap, `amountOut` will always be equal to `amountIn`.
*   `_swapTokensForExactTokens(IERC20 inToken, IERC20 outToken, uint256 amountOut, address to) internal override returns (uint256 amountIn)`:
    *   Similar to the above, this function overrides its counterpart in `AbstractARM.sol`.
    *   It also delegates to the internal `_swap` function.
    *   In a 1:1 swap, `amountIn` will always be equal to `amountOut`.

### New Internal Function
*   `_swap(IERC20 inToken, IERC20 outToken, uint256 amount, address to) internal returns (uint256)`:
    *   This function contains the core logic for executing the 1:1 pegged swap.
    *   It first performs validation checks:
        *   If `bothDirections` is `false`, it ensures the swap is only from `token0` to `token1`.
        *   If `bothDirections` is `true`, it ensures the swap is between `token0` and `token1` in either direction.
        *   It reverts if the `inToken` and `outToken` do not match the configured `token0` and `token1` based on the allowed direction(s).
    *   It then facilitates the token exchange:
        *   Transfers `amount` of `inToken` from the `msg.sender` (the user initiating the swap) to the ARM contract itself.
        *   Transfers an identical `amount` of `outToken` from the ARM contract to the specified recipient address `to`.
    *   It returns `amount`, confirming that the quantity of tokens exchanged is equal for both input and output.

## Interactions

*   **Inheritance from `AbstractARM.sol`:** `PeggedARM` inherits all other functionalities, state variables, and external interactions from `AbstractARM`. This includes:
    *   LP share tokenization (the ARM contract itself is an ERC20 LP token).
    *   Deposit and redemption mechanisms for the `liquidityAsset`.
    *   Fee collection logic.
    *   Integration with an `activeMarket` for yield generation on the `liquidityAsset`.
    *   `OwnableOperable` for access control.
    *   The public interface for swaps (`swapExactTokensForTokens` and `swapTokensForExactTokens` as defined in `AbstractARM`) remains the same for users.
*   **Child Contracts (e.g., `OethARM.sol`):**
    *   `PeggedARM` is an abstract contract and must be inherited by a concrete ARM implementation.
    *   The child contract is responsible for:
        *   Calling the `PeggedARM` constructor with the desired `_bothDirections` setting.
        *   Defining the specific ERC20 token addresses for `token0` and `token1` by passing them to the `AbstractARM` constructor.
    *   When users interact with the public swap functions of such a child contract, the overridden internal functions in `PeggedARM` ensure that the swap executes with 1:1 pegged logic.
*   **Users (Interacting via Child Contracts):**
    *   Users call the standard public swap functions (`swapExactTokensForTokens`, `swapTokensForExactTokens`) available on the child ARM contract.
    *   They do not directly interact with `PeggedARM`'s internal functions. The 1:1 pegged behavior is applied transparently if the swap involves `token0` and `token1` of the child ARM.

`PeggedARM` does not introduce new external contracts or actors beyond those already managed by `AbstractARM`. Its primary impact is on the internal execution path of swap operations for its derived contracts.

## Mermaid Diagram

```mermaid
graph TD
    subgraph ContractHierarchy [Contract Inheritance Structure]
        direction TB
        AbstractARM_Base["AbstractARM.sol\n(Abstract, ERC20Upgradeable, OwnableOperable)"]
        PeggedARM_Contract["PeggedARM.sol\n(Abstract)"]
        ConcreteARM_Child["Concrete ARM (e.g., OethARM.sol)\n(Initializable)"]

        PeggedARM_Contract -- "inherits from" --> AbstractARM_Base
        ConcreteARM_Child -- "inherits from" --> PeggedARM_Contract
    end

    subgraph PeggedARM_Details [PeggedARM.sol Specifics]
        direction TB
        state_bothDirections["bothDirections (bool, immutable)"]
        constructor_pegged["constructor(_bothDirections)"]

        func_override_swapExact["_swapExactTokensForTokens()\n(internal override)"]
        func_override_swapForExact["_swapTokensForExactTokens()\n(internal override)"]
        func_internal_swap["_swap(in, out, amount, to)\n(internal)"]

        constructor_pegged --> state_bothDirections
        func_override_swapExact --> func_internal_swap
        func_override_swapForExact --> func_internal_swap
    end

    subgraph AbstractARM_Interactions [AbstractARM Public Swap Interface]
        direction TB
        User["User"]
        public_swapExact["swapExactTokensForTokens()\n(external, from AbstractARM)"]
        public_swapForExact["swapTokensForExactTokens()\n(external, from AbstractARM)"]

        User -- "Calls" --> public_swapExact
        User -- "Calls" --> public_swapForExact
        public_swapExact -. "Internally uses PeggedARM's override" .-> func_override_swapExact
        public_swapForExact -. "Internally uses PeggedARM's override" .-> func_override_swapForExact
    end

    %% Styling
    classDef abstract fill:#e6ccff,stroke:#333,stroke-width:2px;
    class AbstractARM_Base abstract;
    class PeggedARM_Contract abstract;
    classDef concrete fill:#cceeff,stroke:#333,stroke-width:2px;
    class ConcreteARM_Child concrete;
    classDef actor fill:#ffffcc,stroke:#333,stroke-width:2px;
    class User actor;
    classDef functionInternal fill:#lightgrey,stroke:#333,stroke-width:1px;
    class func_override_swapExact, func_override_swapForExact, func_internal_swap functionInternal;
```

This document outlines `PeggedARM.sol`, an abstract contract that provides configurable 1:1 pegged swap functionality for its child ARM implementations by overriding key internal swap methods of `AbstractARM.sol`.
