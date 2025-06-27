The withdrawToNativeChain function of GatewayTransferNative fails to collect platform fees when users use nativeTokens. The fee collection mechanism silently fails due to a low-level call to a non-contract address. This allows users to bridge native asset while completely bypassing the platform's fee collection, resulting in lost revenue for the protocol.

ROOT CAUSE:

User will provide _ETH_ADDRESS_ as zrc20 param in withdrawToNativeChain function when bridging native asset. Then, internal function _handleFeeTransfer is called:

   function withdrawToNativeChain(address zrc20, uint256 amount, bytes calldata message) external payable {
        if (zrc20 != _ETH_ADDRESS_) {
            require(
                IZRC20(zrc20).transferFrom(msg.sender, address(this), amount),
                "INSUFFICIENT ALLOWANCE: TRANSFER FROM FAILED"
            );  @>> it passes as the token address is an ETH address
        }

       // ...

        // Transfer platform fees
        uint256 platformFeesForTx = _handleFeeTransfer(zrc20, amount); // platformFee = 5 <> 0.5%
        amount -= platformFeesForTx;




There, transfer helper is used to collect fees:

    function _handleFeeTransfer(address zrc20, uint256 amount) internal returns (uint256 platformFeesForTx) {  
        platformFeesForTx = (amount * feePercent) / 1000; // platformFee = 5 <> 0.5%  
        TransferHelper.safeTransfer(zrc20, EddyTreasurySafe, platformFeesForTx);  
    }  

That's where the problem lies - since asset is native asset and not erc20 token, direct ETH transfer methods should be used instead of erc20-transfer:

    function safeTransfer(address token, address to, uint value) internal {  
        // bytes4(keccak256(bytes('transfer(address,uint256)')));  
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0xa9059cbb, to, value));  
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'TransferHelper: TRANSFER_FAILED');  
    }  

What happens instead is that low level call 'transfer(address,uint256)' is executed on address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE). Since there's no code at the address, call will be successful even though no funds were moved. TX execution normally proceeds while protocol collected 0 fees.

here is the description that changes it all 
Just a clarification - as stated in the issue description, TXs will not revert for native tokens. That's because low-level call is used to transfer native tokens. And low-level calls return success when called on the non-existing address (ie. account without code). For that reason, platform fee is not collected, even though withdraw is successful.

Solidity docs reference:

Warning - Due to the fact that the EVM considers a call to a non-existing contract to always succeed, Solidity includes an extra check using the extcodesize opcode when performing external calls. This ensures that the contract that is about to be called either actually exists (it contains code) or an exception is raised.

The low-level calls which operate on addresses rather than contract instances (i.e. .call(), .delegatecall(), .staticcall(), .send() and .transfer()) do not include this check, which makes them cheaper in terms of gas but also less safe.

'safeTransfer' uses low-level 'call', thus there is no check that contract instance exists.
