In `Erc20TokenEmitter`, treasury and creators can bypass the preventing them from buying tokens by including themselves in addresses param.

https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L157-L158

According to the invariants of revolution protocol, treasury and creatorsAddress of ERC20TokenEmitter should not be able to buy tokens. But they can bypass this restriction by including themselves in `addresses` param of `buyToken` function.

## Impact
Treasury and creators of `ERC20TokenEmitter` can bypass the restriction of preventing them from buying tokens.

## Proof of Concept
In `buyToken` function, it reverts if `msg.sender` is address of treasury or creators to prevent them from buying tokens.
But if `addresses` param includes address of treasury or creators, minted tokens for buyers will be distributed to them and it can break the invariant of preventing buying tokens.

```solidity
        for (uint256 i = 0; i < addresses.length; i++) {
            if (totalTokensForBuyers > 0) {
                // transfer tokens to address
                _mint(addresses[i], uint256((totalTokensForBuyers * int(basisPointSplits[i])) / 10_000));
            }
            bpsSum += basisPointSplits[i];
        }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
```solidity
		for (uint256 i = 0; i < addresses.length; i++) {
+			require(msg.sender != treasury && msg.sender != creatorsAddress, "Funds recipient cannot buy tokens");
            if (totalTokensForBuyers > 0) {
                // transfer tokens to address
                _mint(addresses[i], uint256((totalTokensForBuyers * int(basisPointSplits[i])) / 10_000));
            }
            bpsSum += basisPointSplits[i];
        }
```
Additionally, you should add logic that check if treasury or creators of emiiter is included in ArtPiece creators when start auction or creat new piece. Without it, the auction can fall into DoS due to above restriction code.


Since `buyToken` function has no slippage checking, users can get less tokens than expected when they buy tokens directly

https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L152-L230

## Impact
Users can buy NontransferableERC20Token by calling `buyToken` function directly. At that time, the expected amount of tokens they will receive is determined based on current supply and their paying ether amount. But, due to some transactions(such as settleAuction or another user's buyToken) which is running in front of caller's transaction, they can get less token than they expected.

## Proof of Concept
The VRGDAC always exponentially increase the price of tokens if the supply is ahead of schedule. Therefore, if another transaction of buying token is fronrun against a user's buying token transaction, the token price can arise than expected.

For instance, let's assume that ERC20TokenEmitter is initialized with following params:
 - target price: 1 ether
 - decay percent: 10 %
 - per time unit: 10 ether
To avoid complexity, we will assume that the supply of token so far is consistent with the schedule. When Alice tries to buy token with 5 ether, expected amount is calculated by `getTokenQuoteForEther(5 ether)` and the value is about 4.87 ether.
However, if Bob's transaction to buy tokens with 10 ether is executed before Alice, the real amount which Alice will receive is about 4.43 ether.

You can check result through following test:
```solidity
	function testBuyTokenWithoutSlippageCheck() public {
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");

        vm.deal(address(alice), 100000 ether);
        vm.deal(address(bob), 100000 ether);

        address[] memory recipients = new address[](1);
        recipients[0] = address(1);
        uint256[] memory bps = new uint256[](1);
        bps[0] = 10_000;

        // expected amount of minting token when alice calls buyToken
        int256 expectedAmount = erc20TokenEmitter.getTokenQuoteForEther(5 ether);

        vm.startPrank(bob);
        // assume that bob calls buy token with 10 ether
        erc20TokenEmitter.buyToken{ value: 10 ether }(
            recipients,
            bps,
            IERC20TokenEmitter.ProtocolRewardAddresses({
                builder: address(0),
                purchaseReferral: address(0),
                deployer: address(0)
            })
        );

        vm.stopPrank();

        vm.startPrank(alice);
        // calculate the amount of tokens which alice will actually receive
        int256 realAmount = erc20TokenEmitter.getTokenQuoteForEther(5 ether);

        vm.stopPrank();

        emit log_string("Expected Amount: ");
        emit log_int(expectedAmount);
        emit log_string("Real Amount: ");
        emit log_int(realAmount);

        assertLt(realAmount, expectedAmount, "Alice should receive less than expected if Bob frontrun buyToken");
    }
```
Therefore, Alice will get about 0.44 ether less tokens than expected since there is no any checking of slippage in `buyToken` function.

## Tools Used
VS Code

## Recommended Mitigation Steps
Add slippage checking to `buyToken` function. This slippage checking should be executed only when the user calls `buyToken` function directly. In other words, it should not be executed when settleAuction calls `buyToken` function.


The amounts of token for buyers and creators isn't calculated correctly based on received due to incorrect updating total emitted amount.

https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L179-L188

## Impact
The amounts of token for buyers and creators isn't accurately calculated.

## Proof of Concept
ERC20TokenEmitter enables anyone to purchase the ERC20 governance token at any time through `buyToken` function. The `buyToken` function sends a portion of the paid ETH to protocolRewards. And then calculates the amount of tokens to be distributed to buyers and creators at a certain ratio based on the remaining ETH. The `buyToken` function calculates the amount of ETH according to the specified ratio and then calls the `getTokenQuoteForEther` function to calculate the each amount of tokens to be distributed to buyers and creators, respectively.
```solidity
        //Tokens to emit to creators
        int totalTokensForCreators = ((msgValueRemaining - toPayTreasury) - creatorDirectPayment) > 0
            ? getTokenQuoteForEther((msgValueRemaining - toPayTreasury) - creatorDirectPayment)
            : int(0);
```
```solidity
        // Tokens to emit to buyers
        int totalTokensForBuyers = toPayTreasury > 0 ? getTokenQuoteForEther(toPayTreasury) : int(0);
```
However, since the `buyToken` function doesn't accurately update `emittedTokenWad` state(total amount of emitted token), which is used for calculating the token issuance amount in `getTokenQuoteForEther` function, the calculated value is different from the amount that should actually be minted.
```solidity
        //Transfer ETH to treasury and update emitted
        emittedTokenWad += totalTokensForBuyers;
        if (totalTokensForCreators > 0) emittedTokenWad += totalTokensForCreators;
```
As you can see above code, it updates `emittedTokenWad` state after calculating token amounts for both of buyers and creators. This cause incorrect calculation of minting token. Let me explain with example.
let's assume that ERC20TokenEmitter is initialized with following params:
 - creatorrateBps: 2_000
 - entropyRateBps: 5_000
 - target price: 1 ether
 - decay percent: 10 %
 - per time unit: 10 ether

Alice is trying to call `buyToken` function with over `5 ether` by passing her address as `addresses` param, and 10_000(100%) as `basisPointSplits` param. Assume that remaining ETH value is `5 ether` after sending a portion of `msg.value` to protocolRewards.
Then, the amount of ETH which will be used to mint for buyers is `4 ether`:
5 * (10000 - 2000) / 10000 = 4 (ether)
```solidity
	uint256 toPayTreasury = (msgValueRemaining * (10_000 - creatorRateBps)) / 10_000;
```

And, the amount of ETH which will be used to mint for creators is `0.5 ether`:
(5 - 4) * 5000 / 10000 = 0.5 (ether)
(5 - 4) - 0.5 = 0.5 (ether)
```solidity
    //Ether directly sent to creators
    uint256 creatorDirectPayment = ((msgValueRemaining - toPayTreasury) * entropyRateBps) / 10_000;
    //Tokens to emit to creators
    int totalTokensForCreators = ((msgValueRemaining - toPayTreasury) - creatorDirectPayment) > 0
        ? getTokenQuoteForEther((msgValueRemaining - toPayTreasury) - creatorDirectPayment)
        : int(0);
```
And total amount of minting token is `getTokenQuoteForEther(4 ether) + getTokenQuoteForEther(0.5 ether)`. But `buyTokenAmount` isn't update the `emittedTokenWad` between calculating token amount for crators and buyers. It causes wrong calculating price of token based on paid ETH for buyers because VRGDA's calculation is based on total supply and paid ETH amount. As a result, more ERC20 token is minted than expected.
You can check result through the following test:
```solidity
	function testComparingTokenAmountWithUpdatingEmittedAmount() public {
        address alice = makeAddr("Alice");
        vm.deal(address(alice), 100000 ether);

        vm.startPrank(alice);

        address[] memory recipients = new address[](1);
        recipients[0] = address(1);
        uint256[] memory bps = new uint256[](1);
        bps[0] = 10_000;

        // ensure that enough volume was bought for the day, so purchase expectedVolume amount first
        erc20TokenEmitter.buyToken{ value: expectedVolume }(
            recipients,
            bps,
            IERC20TokenEmitter.ProtocolRewardAddresses({
                builder: address(0),
                purchaseReferral: address(0),
                deployer: address(0)
            })
        );

        // first, calculate the actual amount of tokens for buyers and creators.
        int256 for_buyer_1 = erc20TokenEmitter.getTokenQuoteForEther(4 ether);
        int256 for_creators_1 = erc20TokenEmitter.getTokenQuoteForEther(0.5 ether);

        // first, calculate the amount of tokens for buyers and creators with updating emittedTokenWad btw them.
        int256 for_buyer_2 = erc20TokenEmitter.getTokenQuoteForEther(4 ether);
        erc20TokenEmitter.setEmittedTokenWad(erc20TokenEmitter.emittedTokenWad() + for_buyer_2);
        int256 for_creators_2 = erc20TokenEmitter.getTokenQuoteForEther(0.5 ether);

        emit log_string("Current Calc: ");
        emit log_int(for_buyer_1 + for_creators_1);
        emit log_string("Calc with updating emittedTokenWad btw them: ");
        emit log_int(for_buyer_2 + for_creators_2);

        // Assert that the new token amount is less than the previous tokenAmount
        assertGt(for_buyer_1 + for_creators_1, for_buyer_2 + for_creators_2, "");
        vm.stopPrank();
    }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
Move line 188 to line 182:
```solidity
        uint256 creatorDirectPayment = ((msgValueRemaining - toPayTreasury) * entropyRateBps) / 10_000;
        //Tokens to emit to creators
        int totalTokensForCreators = ((msgValueRemaining - toPayTreasury) - creatorDirectPayment) > 0
            ? getTokenQuoteForEther((msgValueRemaining - toPayTreasury) - creatorDirectPayment)
            : int(0);

+		if (totalTokensForCreators > 0) emittedTokenWad += totalTokensForCreators;

        // Tokens to emit to buyers
        int totalTokensForBuyers = toPayTreasury > 0 ? getTokenQuoteForEther(toPayTreasury) : int(0);   // @audit-issue execute `getTokenQuoteForEther` function only once based on total ether with subtracting `creatorDirectorPayment`.

        //Transfer ETH to treasury and update emitted
        emittedTokenWad += totalTokensForBuyers;
-       if (totalTokensForCreators > 0) emittedTokenWad += totalTokensForCreators;
```