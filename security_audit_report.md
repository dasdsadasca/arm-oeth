# Security Audit Report

## 1. Introduction

*   **Audit Scope:** This report covers the security analysis of smart contracts within the `src/contracts/` directory, including:
    *   `AbstractARM.sol`
    *   `LidoARM.sol`
    *   `OethARM.sol`
    *   `OriginARM.sol`
    *   `PeggedARM.sol`
    *   `CapManager.sol`
    *   `OethLiquidityManager.sol`
    *   `SonicHarvester.sol`
    *   `ZapperARM.sol`
    *   `ZapperLidoARM.sol`
    *   `Proxy.sol`
    *   `Ownable.sol`
    *   `OwnableOperable.sol`
    *   `OwnerLP.sol` (Note: `OwnerLP.sol` was read but not subjected to a dedicated security analysis doc; its impact was considered within `OethARM.sol`).
    *   `Interfaces.sol` (Reviewed as part of understanding interactions).
    *   (Utility and market subdirectories like `src/contracts/markets/` and `src/contracts/utils/` were not explicitly in scope for individual file analysis reports but considered where relevant to interactions.)

*   **Methodology:** The audit was conducted through manual code review (simulating a white-box approach). The process involved:
    *   Detailed analysis of individual contract logic.
    *   Cross-contract interaction analysis to identify potential integration issues.
    *   Focus on verifying potential exploit scenarios and functional failures based on initial hypotheses.
    *   Compilation of findings from previously generated `security_analysis_*.md` documents.
    *   Prioritization of findings into Critical, High, Medium, and Low/Informational severity categories.

## 2. Executive Summary

This security audit has identified several vulnerabilities and areas of concern across the audited smart contracts. Two critical vulnerabilities require immediate attention:
*   **C-01: `OethARM.sol` - Missing `_initARM()` Call in `initialize()`:** Renders core LP functionality unusable and can lead to loss of funds for depositors.
*   **C-02: `LidoARM.sol` - Multiple Decrement of `lidoWithdrawalQueueAmount` in `claimLidoWithdrawals()`:** Allows for manipulation of internal accounting, potentially leading to unfair LP share pricing or fee exploitation.

Additionally, high-severity issues related to ownership control (`Ownable.sol`) and potential manipulation of asset valuation (`AbstractARM.sol` via `ActiveMarket`) have been found. Several medium and low-severity findings highlight areas for improvement in robustness, UX, and operational security.

**Overall Risk Assessment:** High. The presence of critical vulnerabilities in core ARM components necessitates immediate remediation to prevent potential financial loss and ensure protocol integrity. Other high and medium severity findings also require careful consideration.

## 3. Critical Vulnerabilities

### C-01: `OethARM.sol` - Missing `_initARM()` Call in `initialize()`

*   **Description:** The `OethARM.initialize()` function fails to call `_initARM()` from its parent contract `AbstractARM.sol`. The `_initARM()` function is responsible for essential setup tasks, including initializing the ERC20 LP token (name, symbol, initial total supply by minting to a dead address), setting initial fee parameters, financial rates (`traderate0/1`, `crossPrice`), `lastAvailableAssets`, and configuring the `capManager`.
*   **Impact:**
    *   **Fund Loss:** Users attempting to `deposit()` WETH will have their funds transferred to `OethARM` but will receive `0` LP shares in return. This is because `totalSupply()` of LP tokens remains `0` (as `_mint(DEAD_ACCOUNT, MIN_TOTAL_SUPPLY)` is skipped), causing `convertToShares()` to return `0`. The `capManager` check, which might otherwise revert, is also bypassed as `capManager` is uninitialized (`address(0)`).
    *   **Non-Functional LP Mechanism:** `requestRedeem()` will revert due to division by zero (`totalSupply()` is `0`). The contract cannot function as an LP pool.
    *   **Uninitialized State:** Critical financial parameters, fee structures, and safety mechanisms like `capManager` remain uninitialized, rendering many ARM functionalities inoperable or unsafe.
*   **Exploit Scenario:**
    1.  `OethARM` is deployed and `initialize(operator)` is called. `_initARM()` is NOT called.
    2.  User calls `OethARM.deposit(amountWETH, userAddress)`.
    3.  Inside `AbstractARM._deposit`, `convertToShares(amountWETH)` is called.
    4.  `convertToShares` calculates `amountWETH * totalSupply() / totalAssets()`. Since `totalSupply()` is `0`, this results in `0` shares.
    5.  `_mint(userAddress, 0)` occurs.
    6.  The user's `amountWETH` is transferred to the `OethARM` contract.
    7.  The `capManager` check is skipped as `capManager == address(0)`.
    8.  The user loses `amountWETH` and receives 0 LP shares.
*   **Recommendation:** **CRITICAL.** Modify `OethARM.initialize()` to correctly call `_initARM()` with appropriate parameters for name, symbol, operator (for AbstractARM's OwnableOperable), fee settings, and `capManager` address.

### C-02: `LidoARM.sol` - Multiple Decrement of `lidoWithdrawalQueueAmount` in `claimLidoWithdrawals()`

*   **Description:** The `claimLidoWithdrawals()` function in `LidoARM.sol` is publicly callable (`external` without access control). When processing claims, it sums the original stETH amounts for the provided `requestIds` from the `lidoWithdrawalRequests` mapping to calculate `totalAmountRequested`, by which `lidoWithdrawalQueueAmount` is then decremented. However, the entries in `lidoWithdrawalRequests` are not cleared or marked as processed after being used in this calculation.
*   **Impact:** An attacker can repeatedly call `claimLidoWithdrawals` with `requestIds` that have already been accounted for by `LidoARM` (even if already claimed on Lido's side). While Lido's platform prevents actual double withdrawal of ETH, the `LidoARM`'s internal accounting for `lidoWithdrawalQueueAmount` will be incorrectly decremented multiple times for the same logical withdrawal. This artificially deflates `lidoWithdrawalQueueAmount`, causing `_externalWithdrawQueue()` to underreport, which in turn makes `totalAssets()` lower than its true value. This can be exploited:
    *   To allow an attacker to mint LP shares at a cheaper rate.
    *   To cause other LPs redeeming shares to receive fewer assets than they are due.
    *   To potentially manipulate fee calculations if a deflated `totalAssets` is later corrected, showing artificial growth.
*   **Exploit Scenario:**
    1.  `LidoARM` has legitimate withdrawal requests, e.g., ReqA (amount `amtA`). `lidoWithdrawalQueueAmount` includes `amtA`, and `lidoWithdrawalRequests[ReqA_id] = amtA`.
    2.  ReqA is claimed legitimately (either by operator or anyone, as function is public). `lidoWithdrawalQueueAmount` is reduced by `amtA`. `lidoWithdrawalRequests[ReqA_id]` remains `amtA`.
    3.  Attacker calls `claimLidoWithdrawals([ReqA_id], [hintA_id])` again.
    4.  Lido's `claimWithdrawals` call likely does nothing or reverts for ReqA_id (as it's already claimed from Lido), but assume the transaction doesn't fully revert for this example or attacker finds a way to make it not revert (e.g. batching with a valid small one).
    5.  `LidoARM` calculates `totalAmountRequested` by summing `lidoWithdrawalRequests[ReqA_id]`, which is still `amtA`.
    6.  `lidoWithdrawalQueueAmount` is incorrectly reduced by `amtA` *again*.
    7.  `totalAssets()` is now artificially deflated. Attacker can then deposit at a favorable rate.
*   **Recommendations:** **CRITICAL.**
    1.  **Clear Processed Requests:** Modify `claimLidoWithdrawals` to `delete lidoWithdrawalRequests[requestIds[i]];` for each `requestId` after its amount has been summed into `totalAmountRequested` and successfully processed by Lido's `claimWithdrawals`. The `require(requestAmount > 0, "LidoARM: invalid request");` check will then prevent reprocessing.
    2.  **Add Access Control:** Restrict `claimLidoWithdrawals` with `onlyOperatorOrOwner` to prevent public exploitation and ensure calls are made by trusted entities aware of which requests are pending claim at the LidoARM level.

## 4. High-Severity Vulnerabilities

### H-01: `Ownable.sol` - No Zero-Address Check in `setOwner()`

*   **Description:** The `_setOwner(address newOwner)` function in `Ownable.sol` (and thus the public `setOwner`) does not check if `newOwner` is `address(0)`.
*   **Impact:** If the current owner accidentally or intentionally calls `setOwner(address(0))`, ownership is transferred to the zero address. This makes all `onlyOwner` modified functions permanently inaccessible because no one can satisfy `msg.sender == address(0)`. For proxies where `owner` is the admin, this means the proxy becomes immutable (upgrades are impossible). Critical administrative functions in any contract inheriting `Ownable` would be lost.
*   **Exploit Scenario:** Owner mistakenly provides `address(0)` due to UI error, script bug, or incorrect input when calling `setOwner`.
*   **Recommendation:** **HIGH.** Add `require(newOwner != address(0), "Ownable: new owner is the zero address");` to the `_setOwner` function. Consider a two-step ownership transfer for highly critical contracts.

### H-02: `AbstractARM.sol` & `ActiveMarket` - `totalAssets()` Manipulation via `activeMarket.previewRedeem()`

*   **Description:** `AbstractARM.totalAssets()` (via `_availableAssets`) relies on `IERC4626(activeMarket).previewRedeem()` to determine the value of assets held in the `activeMarket`. If the `activeMarket`'s `previewRedeem` function is susceptible to manipulation (e.g., uses a manipulable spot price oracle, or its own share price can be skewed by flash loans affecting its underlying assets) within the same transaction as an ARM deposit or redemption, the reported `totalAssets` can be artificially inflated or deflated.
*   **Impact:** Manipulation of `totalAssets()` directly affects LP share pricing (`convertToShares` and `convertToAssets`). An attacker could inflate `totalAssets` before redeeming their shares (to get more underlying assets) or deflate `totalAssets` before depositing (to get more LP shares for their assets), stealing value from other LPs.
*   **Exploit Scenario:**
    1.  Attacker identifies that `activeMarket.previewRedeem` uses a manipulable spot price from a DEX.
    2.  Attacker takes a flash loan.
    3.  Attacker manipulates the spot price on the DEX.
    4.  Attacker calls `ARM.deposit()` (or induces a victim deposit). `totalAssets()` is now artificially low due to the manipulated `previewRedeem` value. Attacker gets more LP shares.
    5.  Attacker repays flash loan; price on DEX returns to normal.
    6.  Attacker later redeems their inflated LP shares when `totalAssets` is correctly valued.
*   **Recommendation:** **HIGH.** Thoroughly vet any `activeMarket` for the manipulation resistance of its `previewRedeem` function. Prefer `activeMarket`s that use robust, manipulation-resistant price sources (e.g., TWAPs from high-liquidity markets) for share valuation if this value is exposed and used by integrators like `AbstractARM`. Alternatively, `AbstractARM` could use more conservative valuation methods for `activeMarket` assets if high precision is not strictly required and manipulation is a concern.

## 5. Medium-Severity Vulnerabilities

### M-01: `AbstractARM.sol` - Fee Inflation via `totalAssets()` Manipulation

*   **Description:** Fees in `AbstractARM` are calculated based on the increase in `_availableAssets()` (which contributes to `totalAssets()`) between fee collection periods. If `_availableAssets()` can be temporarily and artificially inflated just before `collectFees()` is called (e.g., via flash loan donation of `liquidityAsset` or `baseAsset` if `crossPrice` is favorable, or manipulation of `_externalWithdrawQueue` or `activeMarket.previewRedeem`), the calculated `assetIncrease` and thus the fees can be magnified.
*   **Impact:** The `feeCollector` could receive more fees than legitimately accrued from organic yield.
*   **Exploit Scenario:** An attacker (potentially colluding with `feeCollector` or being the `feeCollector`) flash-loans a large amount of `liquidityAsset`, deposits it into the ARM (if no cap or they can bypass), calls `collectFees()` (if public or they are operator), then withdraws the flash-loaned capital.
*   **Recommendation:** **MEDIUM.** Consider making `collectFees()` `onlyOperatorOrOwner` (it is currently public). For more robust fee calculation, explore mechanisms less sensitive to point-in-time asset value manipulation, such as basing fees on time-averaged asset values, though this adds complexity.

### M-02: `AbstractARM.sol` - DoS in `setActiveMarket()`

*   **Description:** When `setActiveMarket()` is called to switch to a new market, it first attempts to withdraw all assets from the `previousActiveMarket` by calling `redeem()`. If this `redeem()` call fails (e.g., market is paused, illiquid, high utilization, or dust amount issues), the entire `setActiveMarket` transaction reverts.
*   **Impact:** The ARM can be prevented from migrating away from a problematic or deprecated `activeMarket`, potentially trapping funds or impairing yield strategy adjustments.
*   **Recommendation:** **MEDIUM.** This is a significant operational risk. Consider adding a mechanism to allow the owner/operator to proceed with setting a new `activeMarket` even if full redemption from the previous one fails (e.g., an emergency market switch function). This would require careful handling of how remaining assets in the old market are tracked and eventually recovered. Document operational procedures for handling dust share issues.

### M-03: `SonicHarvester.sol` - Reliance on External Systems & Operator Trust

*   **Description:** `SonicHarvester` relies on:
    1.  Correctness and security of whitelisted `IHarvestable` strategies.
    2.  Accuracy and manipulation-resistance of the `IOracle` (`priceProvider`).
    3.  Reliability and security of the `IMagpieRouter`.
    4.  The operator to provide correct and non-malicious `data` blob for swaps in `SonicHarvester.swap()`. While key parameters are validated, the detailed routing within the `data` is opaque.
*   **Impact:** Vulnerabilities or misbehavior in these external systems or by the operator can lead to failed harvests, loss of funds (e.g., bad swaps, slippage), or incorrect fee distribution.
*   **Recommendation:** **MEDIUM.**
    *   Implement strict vetting processes for strategies, oracles, and DEX aggregators.
    *   Consider adding more granular validation of the `data` blob for swaps if possible, or using more abstract swap interfaces if available that don't rely on opaque blobs.
    *   Ensure robust monitoring around oracle price feeds and swap outputs.
    *   For critical operations, the operator role should be managed by a secure process (e.g., multi-sig with defined procedures).

### M-04: `ARM` <> `CapManager` - `totalAssets()` Manipulation Risk for Cap Bypass

*   **Description:** `CapManager.postDepositHook()` calls `ARM.totalAssets()` to check `totalAssetsCap`. If `ARM.totalAssets()` can be temporarily manipulated downwards within the same transaction as an ARM deposit *before* the hook is called (e.g., via flash loan exploiting an oracle used by `ARM.activeMarket.previewRedeem()`), the cap check might use an artificially low value.
*   **Impact:** A user could deposit assets exceeding the intended `totalAssetsCap` or their `liquidityProviderCap` if it's derived from a manipulated share price.
*   **Recommendation:** **MEDIUM.** This is an integration risk. ARMs should use manipulation-resistant oracles for any components of `totalAssets()` if possible (e.g., for `activeMarket` valuation). `CapManager` itself is behaving correctly based on the data it gets.

## 6. Low-Severity / Informational Findings

*   **L-01: `AbstractARM.sol` - Minor DoS in `setCrossPrice`:** Donating dust amounts of `baseAsset` (if balance check uses `< MIN_TOTAL_SUPPLY`) can block legitimate `crossPrice` decreases. Impact is limited but can be a nuisance for admins.
*   **L-02: `AbstractARM.sol` - Swap `+3` Wei Adjustment:** The `+3` wei in `_swapTokensForExactTokens` is a slight, systematic overcharge to users (gain for ARM) if the `baseAsset` is a standard ERC20 (not stETH-like with transfer shortfalls). This is very minor.
*   **L-03: `CapManager.sol` - Consumable Individual Caps UX:** Individual caps in `CapManager` are reduced on deposit but not replenished on ARM withdrawal, requiring manual admin updates for users to deposit again after withdrawing. This is a UX/operational issue.
*   **L-04: Zapper Contracts (`ZapperARM`, `ZapperLidoARM`) - ETH Bundling Edge Case:** If ETH is pre-sent/stuck in a Zapper, it's bundled into the next user's deposit, benefiting that user. Minor fairness issue, not a direct exploit of the current user.
*   **L-05: General - Lack of Global Reentrancy Guards:** Several contracts (notably `AbstractARM` and `SonicHarvester`) perform external calls without global `nonReentrant` modifiers. While specific reentrancy exploits were not always obvious for the functions themselves, this increases risk if external contracts are malicious or if complex call chains lead to unexpected state interactions.

## 7. Conclusion & General Recommendations

The audited contracts present a complex system with significant external dependencies and privileged roles. While many components demonstrate robust logic for their specific tasks, critical vulnerabilities have been identified in `OethARM.sol` (initialization failure leading to fund loss) and `LidoARM.sol` (accounting error allowing `totalAssets` deflation). High-severity issues also exist concerning ownership transfer in `Ownable.sol` and potential manipulation of `AbstractARM.totalAssets()` via external market integrations.

**Priority Actions:**
1.  **Fix C-01 (`OethARM.initialize`)**: Ensure `_initARM()` is called.
2.  **Fix C-02 (`LidoARM.claimLidoWithdrawals`)**: Clear processed request IDs and add access control.
3.  **Address H-01 (`Ownable.setOwner`)**: Add a zero-address check for the new owner.
4.  **Review H-02 (`AbstractARM` `totalAssets` manipulation via `ActiveMarket`)**: Assess and mitigate risks associated with `activeMarket.previewRedeem()` manipulation for all integrated vaults.

**General Recommendations:**
*   **Adopt `nonReentrant` Modifiers:** Implement OpenZeppelin's `ReentrancyGuard` (or a similar pattern) on key state-changing functions that have complex external interactions (e.g., `AbstractARM.allocate`, `AbstractARM.setActiveMarket`, `SonicHarvester.swap`).
*   **Strengthen Admin Procedures:** For functions involving critical changes or external data (like `SonicHarvester.swap` data blob), consider multi-sig approvals or more stringent off-chain validation before on-chain execution by an operator.
*   **Thorough Vetting of External Integrations:** All external contracts (ARMs' `activeMarket`, `SonicHarvester`'s strategies, oracles, DEX aggregators) must be thoroughly vetted for security, reliability, and compliance with expected interfaces and behaviors.
*   **Improve User Experience for Consumable Caps:** If individual caps in `CapManager` are a long-term feature, consider adding a mechanism to replenish them upon user withdrawal from the ARM to avoid unnecessary admin burden and user friction.
*   **Comprehensive Testing:** Ensure comprehensive test coverage, including scenarios for identified vulnerabilities, edge cases, and interactions with various types of external dependencies (e.g., mock contracts for oracles, markets, strategies with varying behaviors).

Addressing these findings, particularly the critical and high-severity issues, is essential for the security and reliability of the protocol.
