# Loan Extension Fails Due to Unused Time Calculation

## Summary

The `extendLoan` function in `DebitaV3Loan::extendLoan` has redundant time calculation logic that causes transaction reversions when borrowers attempt to extend loans near their deadline.

## Root Cause

In the `DebitaV3Loan` contract the function `extendLoan` function has a varaible `extendedTime` that is not used and can cause reverts in some cases which cause some borrowers to not be able to extend their loan.

The exact line of code:

https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/DebitaV3Loan.sol#L590

## Internal pre-conditions

1. Loan must not be already extended `(extended == false)`
2. Borrower must have waited minimum duration `(10% of initial duration)`
3. Loan must not be expired` (nextDeadline() > block.timestamp)`
4. Must have at least one unpaid offer

## External pre-conditions

1. Current time must be close to offer's `maxDeadline`
2. `maxDeadline - (block.timestamp - startedAt)` - `block.timestamp` < 0

## Attack Path

1. Loan starts at timestamp `1704067200` (January 1, 2024)
2. Time advances to `1705190400` (January 14, 2024)
3. Borrower attempts to extend loan
4. For an offer with maxDeadline `1705276800` (January 15, 2024)
5. Transaction reverts due to arithmetic underflow

## Impact

Borrowers cannot extend loans near their deadlines even when they satisfy all other requirements:

1. Forces unnecessary defaults near deadline
2. Wastes gas on failed extension attempts
3. Disrupts normal loan management operationsthe

## PoC

This PoC demonstrates the reversion caused by unused time calculations in `extendLoan` function.

```solidity
contract BugPocTime {

    uint256 loanStartedAt = 1704067200; // 1 January 00:00 time
    uint256 currentTime = 1705190400; // 14 January 00:00 time
    uint256 maxDeadline = 1705276800; // 15 January 00:00 time

    function extendLoan() public view returns(uint256){
        uint256 alreadyUsedTime = currentTime - loanStartedAt;
        uint256 extendedTime = maxDeadline - alreadyUsedTime - currentTime;

        return 10;
    }
}
```

The example uses the following timestamps:

1. loanStartedAt: 1704067200 (Jan 1, 2024 00:00)
2. currentTime: 1705190400 (Jan 14, 2024 00:00)
3. maxDeadline: 1705276800 (Jan 15, 2024 00:00)

The calculation flow:

1. alreadyUsedTime = 1705190400 - 1704067200 = 1,123,200 (≈13 days)
2. extendedTime = 1705276800 - 1,123,200 - 1705190400
   = 1705276800 - 1706313600
   = -1,036,800 (reverts due to underflow)

## Mitigation

Remove the unused `extendedTime` calculation as it serves no purpose and can cause legitimate loan extensions to fail.
