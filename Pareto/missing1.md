### Summary
Lack of proportional loss distribution mechanisms will cause a complete loss of funds for late-redeeming USP holders as early redeemers will continue to receive 1:1 redemption even after the protocol becomes undercollateralized.
### Root Cause
The protocol maintains 1:1 redemption until collateral is completely exhausted, without implementing any mechanisms to distribute losses proportionally among all token holders when catastrophic losses exceed the staking pool value.

https://github.com/sherlock-audit/2025-04-pareto-contest/blob/main/USP/src/ParetoDollarQueue.sol#L284-L361

This creates a "last man standing" scenario where token holders who redeem late face 100% loss while early redeemers face no loss at all.

### External Pre-conditions
A yield source experiences a catastrophic loss exceeding the total USP balance in the staking contract
### Attack Path
User 1 deposits 70 USDC and mints 70 USP (does not stake)
User 2 deposits 30 USDC and mints 30 USP (stakes all 30 USP, receiving 30 sUSP)
User 3 deposits 10 USDC and mints 10 USP (does not stake)
At this point:

Total collateral: 110 USDC
Total USP: 110 USP (70 unstaked + 30 staked + 10 unstaked)
Total sUSP: 30 sUSP
The protocol deploys all 110 USDC to yield farming protocols
A 50% slashing event occurs in the yield protocol, resulting in 55 USDC loss
The protocol now has only 55 USDC backing 110 USP tokens
Attempting to distribute the loss:
7. User 2 (staker) can only absorb 30 USDC of loss through their 30 sUSP
8. 25 USDC of loss remains unallocated, yet the protocol still honors the 1:1 peg for redemptions

User 3 redeems their 10 USP first, receiving 10 USDC at full value and no loss for user 3
After User 3's redemption, only 45 USDC remains for 70 USP (70 from User 1)
When User 1 attempts to redeem 70 USP, they can only receive 45 USDC
User 1 suffers a 25 USDC loss despite not participating in yield generation while user 3 did not suffers any loss
### Impact
Loss for the user even though they have not staked.
Loss to the people who redeem the tokens late.
Broke the 1:1 peg and hence the calculation in ParetoDollar.sol

