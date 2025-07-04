Issue #928
Submitted on June 9, 2025 at 3:59:33 PM GMT+1

Boolean Return Assumption in transferFrom() Causes Token Compatibility Issues
## Summary
The GatewaySend contract incorrectly uses raw IERC20.transferFrom() calls wrapped in require() statements, expecting a boolean return value from all ERC20 tokens. However, tokens like USDT, which have a void return type, cause these calls to fail even when the transfer succeeds, as Solidity interprets the missing return as false. This makes the contract incompatible with major non-standard tokens, despite importing TransferHelper for safe transfers, which is not consistently applied.

## Root Cause
https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/GatewaySend.sol#L232C1-L242C10

Internal Pre-conditions
N/A

External Pre-conditions
N/A

Attack Path
A user, Charlie, tries to bridge 5,000 USDT from Ethereum to Polygon using depositAndCall(). After approving the GatewaySend contract to spend 5,000 USDT, Charlie initiates the transfer. The contract calls IERC20(USDT).transferFrom(charlie, contract, 5000), which successfully transfers the tokens, reducing Charlie’s balance and increasing the contract’s balance. However, since USDT’s transferFrom returns void instead of true, Solidity interprets this as false, triggering the require() failure with the error "INSUFFICIENT AMOUNT: ERC20 TRANSFER FROM FAILED." The transaction reverts, wasting Charlie’s gas fees and preventing the bridge operation.

Impact
Transaction Failures: Legitimate transfers revert, rendering the contract unusable for major tokens like USDT.

User Cost: Users lose gas fees due to failed transactions, degrading the bridging experience.

PoC
No response

Mitigation
safe transferrom shoyuld be used
