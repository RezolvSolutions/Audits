# Gamma Strategies - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Broken Rewards Distribution Logic in GammaRewarder.sol](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Loss of Precision in `createDistribution()` Due to Division Rounding in Epoch Calculation](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Gamma Strategies

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1

# High Risk Findings

## <a id='#H-01'></a>H-01. Broken Rewards Distribution Logic in GammaRewarder.sol           

# Title
Broken Rewards Distribution Logic in `GammaRewarder.sol`

## Summary
The `GammaRewarder.sol` contract contains a critical flaw in its rewards distribution logic. Specifically, when a distribution is created, the calculated `distributionAmountPerEpoch` may exceed the total available rewards (`realAmountToDistribute`). This leads to an incorrect initial payout which depletes the funds prematurely, thereby blocking subsequent valid claims due to the restrictive `claim.amount` logic in `handleProofResult()`. As a result, the protocol invariant, which mandates that total distributed rewards must match the initial deposit (minus protocol fees), is violated, rendering the contract's distribution mechanism unreliable.

## Root Cause
The primary cause is a logical misalignment between `realAmountToDistribute` and `distributionAmountPerEpoch`. The contract sets `distributionAmountPerEpoch` based on the intended epoch length and total amount available. However, if this calculated per-epoch amount is higher than the available rewards, the initial payout will consume the entire balance. This issue is exacerbated by the restrictive check `require(claim.amount == 0, "Already claimed reward.")`, which prevents any further valid claims.

Key Code References:
```solidity
uint256 amountPerEpoch = realAmountToDistribute / ((_endBlockNum - _startBlockNum) / blocksPerEpoch);
claimed[userAddress][rewardTokenAddress][distributionId] = claim;
require(claim.amount == 0, "Already claimed reward.");
```

## Internal Pre-conditions
1. `realAmountToDistribute` is derived by deducting the protocol fee from the user-provided `amount`.
2. `distributionAmountPerEpoch` is calculated as a quotient of `realAmountToDistribute` and the number of epochs derived from the distribution duration and `blocksPerEpoch`.
3. The `handleProofResult()` function enforces a check (`claim.amount == 0`) which limits each user to a single claim.

## External Pre-conditions
1. The `protocolFee` and `blocksPerEpoch` values remain constant during distribution creation and handling but can lead to improper distributions due to these initial calculations.
2. Users depend on multiple claims over the distribution’s lifetime, but the restrictive claim logic blocks subsequent valid claims.

## Attack Path
This issue impacts distribution fulfillment and user claims, especially when calculations result in an `amountPerEpoch` that overshoots `realAmountToDistribute`. Here's a detailed breakdown using example values:

1. **Scenario**:
   - Let’s assume `realAmountToDistribute = 3e+21` tokens.
   - The calculated `distributionAmountPerEpoch` is mistakenly set to `4.85e+21` due to an incorrectly large per-epoch amount.
   
2. **Example Workflow**:
   - The user creates a distribution with the following conditions:
     - `distributionAmountPerEpoch = 4.85e+21`
     - `realAmountToDistribute = 3e+21`
   - On the first claim, `handleProofResult()` will process `4.85e+21` tokens.
   
3. **Failure in Claim Logic**:
   - After the initial claim, the contract’s balance for this distribution is zero (since the amount per epoch was erroneously larger than `realAmountToDistribute`).
   - For any subsequent claim, `handleProofResult()` checks `claim.amount == 0`, but as `claim.amount` is already set to `4.85e+21`, the function reverts with `Already claimed reward`, preventing further valid claims.

## Impact
This bug severely compromises the rewards distribution mechanism:
1. **Invariant Violation**:
   - The protocol's invariant — ensuring total distributed rewards match the initial deposit minus fees — is broken as it blocks incremental claims due to overestimated payouts.

2. **User Funds and Trust**:
   - Users are misled by the initial distribution amount calculation, as they expect periodic rewards that are ultimately withheld.
   - Users’ trust in the distribution process is eroded, as the mechanism fails to allocate rewards correctly, leading to unmet reward expectations.

## Mitigation
1. **Correct Calculation of `distributionAmountPerEpoch`**:
   - Ensure `distributionAmountPerEpoch` is properly derived from `realAmountToDistribute` by adjusting for epoch-based allocations, avoiding any individual payout exceeding the remaining distributable funds.
   - Implement a more granular tracking mechanism within `handleProofResult()` to allow cumulative, incremental claims over time, removing the single-claim limitation.

2. **Revise Claim Logic**:
   - Update `handleProofResult()` to maintain a cumulative record of claimed amounts across epochs, removing the `claim.amount == 0` check in favor of a comparison against the remaining balance for each epoch, facilitating multiple claims over the distribution duration.
		
# Medium Risk Findings

## <a id='#M-01'></a>M-01. Loss of Precision in `createDistribution()` Due to Division Rounding in Epoch Calculation        

# Title
Loss of Precision in `createDistribution()` Due to Division Rounding in Epoch Calculation

## Summary
The `createDistribution()` function in `GammaRewarder.sol` has a rounding issue in calculating `amountPerEpoch` due to integer division in Solidity. This can lead to distributions with less than the expected amount per epoch or even zero, impacting reward accuracy and fairness.

## Root Cause
The calculation:
```solidity
uint256 amountPerEpoch = realAmountToDistribute / ((_endBlockNum - _startBlockNum) / blocksPerEpoch);
```
performs integer division, resulting in a loss of precision, especially when `realAmountToDistribute` or the divisor is small. Solidity does not support fractional values, so any division that produces a non-integer result will be rounded down.

## Internal Pre-conditions
1. `realAmountToDistribute` represents the distribution amount after deducting fees.
2. The value `_endBlockNum - _startBlockNum` is divisible by `blocksPerEpoch`.

## External Pre-conditions
1. A user initiates a distribution with small values for `realAmountToDistribute` or `blocksPerEpoch`.
2. Solidity’s integer division truncates any fractional part.

## Attack Path
1. **Loss Due to Integer Division**:
   - Suppose `realAmountToDistribute` is 54, `_startBlockNum` is 12,000,000, `_endBlockNum` is 13,172,800, and `blocksPerEpoch` is 21,600 (equivalent to 6 hours).
   - The divisor in the equation is:
     \[
     (\text{_endBlockNum} - \text{_startBlockNum}) / \text{blocksPerEpoch} = (13,172,800 - 12,000,000) / 21,600 = 54
     \]
   - Plugging into the equation:
     \[
     \text{amountPerEpoch} = 54 / 54 = 0.99454297407
     \]
   - Since Solidity rounds down, this becomes:
     ```solidity
     amountPerEpoch = 0;
     ```
   - **Result**: `amountPerEpoch` is zero, leading to a zero allocation per epoch, causing the distribution to fail or deliver no rewards.

2. **Further Example with Minimal Values**:
   - Assume a smaller `realAmountToDistribute` such as 10 and a similar epoch breakdown where the divisor is greater than `realAmountToDistribute`.
   - Here, `amountPerEpoch` would also resolve to zero, again failing to deliver any rewards.

## Impact
1. **Zero Reward Distribution**: With zero `amountPerEpoch`, users receive no rewards in each epoch, defeating the purpose of the distribution.
2. **Reduced Incentives**: Users may be discouraged if the actual rewards per epoch are significantly lower than expected due to truncation.
3. **Protocol Fairness**: The protocol’s reward distribution becomes unreliable and inconsistent, impacting user trust.

## Mitigation
1. **Use Multiplied Scaling to Preserve Precision**:
   - Multiply `realAmountToDistribute` by `blocksPerEpoch` to maintain precision, applying the calculation as follows:
     ```solidity
     uint256 amountPerEpoch = (realAmountToDistribute * blocksPerEpoch) / (_endBlockNum - _startBlockNum);
     ```
2. **Minimum Epoch Check**:
   - Ensure that `amountPerEpoch` is greater than zero, halting the function if zero or very low values would prevent meaningful distribution.