# Security Analysis of `SonicHarvester.sol`

This document provides a security analysis of `SonicHarvester.sol`, a contract responsible for collecting rewards from multiple strategies, swapping them into a common `liquidityAsset`, and distributing them to a `rewardRecipient`.

## 1. Analysis of `_doSwap` Assembly for Magpie Router

The `_doSwap` function contains an assembly block specifically for parsing the `data` bytes array when `SwapPlatform.Magpie` is used. This `data` is expected to be the `swapWithMagpieSignature` calldata.

*   **Memory Offsets and Bit Shifts:**
    *   The assembly uses hardcoded offsets (36, 56, 76 for addresses; 105, 106 for amount parsing components) and bit shifts (96, 248, 240) to extract:
        *   `parsedRecipient` (expected recipient of swap output)
        *   `parsedFromAsset` (token being swapped)
        *   `parsedToAsset` (token to be received)
        *   `parsedFromAssetAmount` (amount of token being swapped)
    *   These offsets and shifts are highly specific to the structure of Magpie's `swapWithMagpieSignature` calldata.
    *   **Verification of Correctness:** Without Magpie's exact calldata specification for `swapWithMagpieSignature` readily available in this context, a definitive confirmation of these offsets is challenging. However, such parsing is common when dealing with pre-signed data blobs for DEX aggregators. The key is that these values are extracted *for validation purposes* within the harvester.
    *   **Status:** Appears plausible for its purpose. The critical aspect is whether the subsequent validation checks are sufficient.

*   **Validation Checks:**
    *   `if (address(this) != parsedRecipient) revert InvalidSwapRecipient(parsedRecipient);`
    *   `if (fromAsset != parsedFromAsset) revert InvalidFromAsset(parsedFromAsset);`
    *   `if (liquidityAsset != parsedToAsset) revert InvalidToAsset(parsedToAsset);`
    *   `if (fromAssetAmount != parsedFromAssetAmount) revert InvalidFromAssetAmount(parsedFromAssetAmount);`
    *   **Sufficiency:** These checks ensure that the core parameters of the swap encoded within the `data` blob (which Magpie will execute) align with the parameters intended by the `SonicHarvester`'s `swap` function call (i.e., swapping `fromAsset` for `liquidityAsset`, with the output going to the harvester itself, for the specified `fromAssetAmount`).
    *   This is a strong defense against the operator providing a `data` blob that, while valid for Magpie, might target different assets, amounts, or send the proceeds to an unintended recipient *before* the harvester gets control back.
    *   **Status:** These checks are crucial and appear to correctly validate the critical components of the swap data against the harvester's expectations. They significantly mitigate the risk of a malicious operator crafting `data` to steal funds directly during the `IMagpieRouter.swapWithMagpieSignature(data)` call. The ultimate security of the swap still relies on Magpie Router's own security and that the signed message (if any part of `data` is signed) corresponds to these validated parameters.

*   **Malformed `data`:**
    *   If `data` is shorter than the offsets being read by `mload` (e.g., less than ~110 bytes), the `mload` operations could read beyond the actual length of `data` but within the bounds of the `bytes memory data` variable's allocated memory (which might be padded or contain zeros). This could lead to `parsedRecipient`, etc., being zero or garbage values.
    *   If these parsed values are then zero or incorrect, the validation checks above would likely cause a revert (e.g., `InvalidSwapRecipient` if `parsedRecipient` becomes `address(0)`).
    *   **Status:** Low risk of exploitation due to malformed `data` leading to fund loss because of the subsequent validation checks. Malformed `data` would most likely lead to a revert.

## 2. Analysis of Slippage Check Logic in `swap()`

*   Formula: `minExpected = (fromAssetAmount * (1e4 - allowedSlippageBps) * oraclePrice) / 1e4 / 1e18;`
*   **Order of Operations:** Multiplications are performed before divisions, which is good for maintaining precision.
*   **Overflow Potential:**
    *   `fromAssetAmount` can be large (e.g., `10^18` to `10^24` for an 18-decimal token).
    *   `(1e4 - allowedSlippageBps)` is at most `1e4`.
    *   `oraclePrice` is the price of `fromAsset` in terms of `liquidityAsset`. If `liquidityAsset` is WETH (18 decimals) and `fromAsset` is also 18 decimals, `oraclePrice` would be scaled to `1e18` for a 1:1 price. If `fromAsset` is significantly more valuable or has fewer decimals, `oraclePrice` could be larger. The division by `1e18` at the end suggests `oraclePrice` is expected to be scaled by `1e18`.
    *   Worst case for multiplication: `fromAssetAmount * 1e4 * oraclePrice`.
        *   If `fromAssetAmount` is `10^24` (1 million tokens with 18 decimals), `oraclePrice` is `10^18` (1:1 price), then `10^24 * 10^4 * 10^18 = 10^46`. This is well within `uint256` max (approx `1.1579e77`).
        *   Even if `oraclePrice` was as high as `10^36` (extremely unlikely for a standard oracle price feed), the product `10^24 * 10^4 * 10^36 = 10^64`, still within limits.
    *   **Status:** Robust against overflow under realistic conditions for token amounts and oracle price scales.

*   **Skipping Slippage Check (`priceProvider == address(0)`):**
    *   If `priceProvider` is `address(0)`, the entire slippage check logic is bypassed, and the `toAssetAmount` returned by the DEX aggregator is accepted as is.
    *   **Status:** This is a configurable risk. It allows operation without reliance on an oracle (which might be a DoS vector itself or unavailable for certain pairs) but sacrifices slippage protection. The decision to set `priceProvider` to `address(0)` is an administrative one with clear security trade-offs.

## 3. Analysis of `collect()` Function

*   **Reentrancy from `IHarvestable(_strategies[i]).collectRewards()`:**
    *   The `collect()` function iterates through `_strategies` and calls `strategy.collectRewards()`. This is an external call.
    *   `collect()` itself does not modify any critical state during the loop that a re-entrant call could easily exploit to corrupt the overall collection process (e.g., it's populating local arrays `rewardTokens` and `amounts`).
    *   If a malicious strategy re-entered `collect()`, it would start a new, independent collection, likely just increasing gas costs or failing due to call depth.
    *   It cannot re-enter `swap()` because `swap()` is `onlyOperatorOrOwner` and `collect()` is public.
    *   If it re-entered a setter function (e.g., `setSupportedStrategy`), it would require the strategy to also be the owner, which is an external security issue (compromised owner adding malicious strategy).
    *   The primary risks from a malicious strategy in `collectRewards()` are:
        1.  Reverting, causing the entire `collect()` transaction to fail (griefing).
        2.  Not transferring any tokens or transferring fewer tokens than expected.
        3.  Transferring worthless or malicious ERC20 tokens (which would then proceed to the swap stage).
    *   **Status:** Low direct reentrancy risk to `SonicHarvester`'s state integrity from within `collect()`. The main trust assumption is on the whitelisted strategies.

*   **Trust in `supportedStrategies`:**
    *   The security of the funds (reward tokens) collected depends on the legitimacy and security of the contracts added via `setSupportedStrategy()`.
    *   If a malicious contract is added as a strategy, it could, during its `collectRewards()` call (which is called by the harvester), potentially perform harmful actions if it has any special approvals or if other contracts are vulnerable to it. However, its direct ability to harm `SonicHarvester` via this call is limited to griefing or returning unexpected tokens.
    *   **Status:** This is an administrative trust issue. The owner must ensure only valid and secure strategy contracts are whitelisted.

## 4. Interaction with `rewardRecipient` in `swap()`

*   The final action in a successful `swap()` operation is `IERC20(liquidityAsset).safeTransfer(rewardRecipient, toAssetAmount);`.
*   **Reentrancy:** This is an external call. If `rewardRecipient` is a contract, it can execute code.
    *   All state changes related to the *current swap instance* (e.g., balance checks, event emission, slippage calculation) within `SonicHarvester.swap()` are completed *before* this `safeTransfer`.
    *   If `rewardRecipient` re-entered `swap()`, it would initiate a completely new swap operation (if it had funds and valid data, and was also an operator/owner). It would not be able to interfere with the already concluded state of the outer `swap()` call that triggered the transfer.
    *   **Status:** Robust against reentrancy from `rewardRecipient` affecting the integrity of the swap that initiated the transfer, due to adherence to the Checks-Effects-Interactions pattern for this final step.

## 5. Access Control and Configuration Setters

*   `setPriceProvider(address _priceProvider)`: `onlyOwner`. Allows `address(0)`. Correct.
*   `setAllowedSlippage(uint256 _allowedSlippageBps)`: `onlyOwner`. Validates `_allowedSlippageBps <= 1000` (10%). Correct.
*   `setRewardRecipient(address _rewardRecipient)`: `onlyOwner`. Validates `_rewardRecipient != address(0)`. Correct.
*   `setSupportedStrategy(address _strategyAddress, bool _isSupported)`: `onlyOwner`. Correct.
*   `setMagpieRouter(address _router)`: `onlyOwner`. Validates `_router != address(0)`. Correct.
*   `collect(address[] calldata _strategies)`: Public. This is acceptable as it only calls out to whitelisted strategies to pull funds *into* the harvester. It doesn't authorize spending or change critical config.
*   `swap(...)`: `onlyOperatorOrOwner`. Appropriate, as this function executes trades and transfers out the final `liquidityAsset`.
*   **Status:** Access controls and validations on setter functions appear appropriate and correctly implemented.

## 6. Constructor `liquidityAsset` Decimal Check

*   `if (IERC20Metadata(_liquidityAsset).decimals() != 18) revert InvalidDecimals();`
*   **Observation:** This hardcodes the expectation that the `liquidityAsset` (the asset rewards are converted *into*) must have 18 decimals. This is likely because the slippage calculation `... / 1e18;` assumes the `oraclePrice` is normalized to an 18-decimal `liquidityAsset`. If a different decimal `liquidityAsset` were to be used, this slippage math would be incorrect.
*   **Status:** This is a design choice that simplifies the oracle price and slippage logic by standardizing on 18 decimals for the `liquidityAsset`. It's a valid constraint.

## 7. Conclusion

`SonicHarvester.sol` is a complex contract with significant external interactions.
*   The Magpie-specific assembly in `_doSwap`, while intricate, is protected by strong validation checks that ensure the swap parameters align with the harvester's intent, mitigating risks from a malicious `data` blob if the operator were compromised.
*   Slippage check logic is sound, with a configurable escape hatch if an oracle is not used. Overflow risk in the calculation is low.
*   The `collect` function's security relies on the trustworthiness of whitelisted strategies. Reentrancy risks from strategy calls to `collect` itself seem minimal.
*   The final transfer to `rewardRecipient` follows the Checks-Effects-Interactions pattern.
*   Access controls for configuration are appropriate.
*   The hardcoded 18-decimal requirement for `liquidityAsset` is a notable design constraint linked to the slippage math.

Primary risks are related to:
1.  **Trust in Admin/Operator Roles:** For correct configuration and honest execution of swaps.
2.  **Security of External Contracts:** Bugs or vulnerabilities in `supportedStrategies`, `priceProvider` (oracle), or `magpieRouter` could impact the harvester.
3.  **Correctness of Magpie Swap Data:** The harvester validates key parts of the Magpie data but relies on Magpie's system and any associated signatures for the actual swap execution integrity.
