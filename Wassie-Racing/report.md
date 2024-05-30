<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://github.com/Delvir0/files/blob/main/wassieracing.png" width="1000" height="250" /></td>
        <td>
            <h1>Wassie Racing</h1>
            <h2>Betting game</h2>
            <h3>Status: deployed on testnet</h3>
            <p>Author of Smart Contracts: @cuibono @0xOrdo </p>
            <p>Prepared by: Delvir0, Independent Security Researcher</p>
            <p>Date of completion: 21-03-2024</p>
        </td>
    </tr>
</table>

# About **Wassie Racing**
Wassie Racing is a ‘horse racing’ style game launching on Blast. The project contains main betting game alongside two other minigames.
Depositors earn a small rake on bets - plus native yield and gas rebates 

# Summary & Scope

[`Game-contracts`](https://github.com/0xOrdo/Game-Contracts/tree/main/contracts), including contracts inherited, under commit [094042b](https://github.com/0xOrdo/Game-Contracts/commit/094042b67daa083000b291e72c5823432066d496)

The following contracts were in scope:
- [racingGame.sol@094042b](https://github.com/0xOrdo/Game-Contracts/blob/main/contracts/racingGame.sol)
- [Flip.sol@094042b](https://github.com/0xOrdo/Game-Contracts/blob/main/contracts/Flip.sol) 
- [HighLow.sol@094042b](https://github.com/0xOrdo/Game-Contracts/blob/main/contracts/HighLow.sol) 
- [rerouteYield.sol@094042b](https://github.com/0xOrdo/Game-Contracts/blob/main/contracts/rerouteYield.sol)

# Summary of Findings

|  Identifier  | Title                        | Severity      | Status | Commit to fix |
| ------ | ---------------------------- | ------------- | ----- | ----- | 
| [C-01] | Refferal mechanism can be used to drain the contract | Critical | Fixed |[d657332](https://github.com/0xOrdo/Game-Contracts/commit/d6573321b90b24fbf8da41cfbee743f49d877eab)|
| [H-01] | Not accounting for fee tokens will lead to last user(s) not being able to withdraw (all) funds due to imbalanced accounting | High | Acknowledged  | [Acknowledged - not supported](https://github.com/........) |
| [H-02] | Griefing attack could result in failing fulfillUint256() leading to lost funds of bets and temporarily halts contract | High | Fixed | [71ddbcd](https://github.com/0xOrdo/Game-Contracts/commit/71ddbcdc6e753b5c71de49d6fbebfb219f64d9be) |
| [M-01] | A failing API3 callback to fulfillUint256() could result in lost funds of bets and temporarily halts contract | Medium | Fixed |[71ddbcd](https://github.com/0xOrdo/Game-Contracts/commit/71ddbcdc6e753b5c71de49d6fbebfb219f64d9be)|
| [L-01] | Adding approved tokens could lead to DoS which permanently breaks the game | Low | Fixed |[7f08ea2](https://github.com/0xOrdo/Game-Contracts/commit/7f08ea21f0f49498d80f68f52203d830fbcd5efc)|
| [L-02] | IERC20 transfer calls will fail with non-complaint tokens | Low | Acknowledged  |[Acknowledged - not supported](https://github.com/...)|
| [I-01] | Contract addresses are still set to BLAST Testnet addresses | Informational | Fixed |[e9418f1](https://github.com/0xOrdo/Game-Contracts/commit/e9418f1f2bdff81afe4c4538e06392c416196e40)|

# Detailed Findings

## [C-01] Refferal mechanism can be used to drain the contract - racingGame.sol
### Severity
**Impact:** Critical, an attacker could drain the contract
**Likelihood:** High, requires game state to be either closed or paused

### Summary
Placing a bet reserves an amount of the fee, which is based on the bet amount, to the refferal. When state is either closed or paused, this can be called with a high amount (and/ or looped) without placing a bet or transfering funds while increasing refferal amount indefinitely.
The incorrect increased refferal amount can then be claimed.


### Details
`racingGame.placeBet` includes the following invariants:
- if bet amount == 0 -> revert
- if (currentState == State.SETTLED && block.timestamp > (timeSettled + settledPeriod)) -> start new game (set state to open)
- if user had refferal -> add fee/100 amount to refferalBalance
- if not within risk -> revert
- if currentState == open -> place bet (transfer tokens to contract)
- if (block.timestamp > (gameStarted + bettingPeriod)) -> finish game (set state to closed)

This means that only in the cases of `bet amount == 0` and `bet not within risk` the function will revert and that betting is possible in mentioned state.
This leads to a window where the function is vulnerable if the contract state is closed or when State.SETTLED && block.timestamp **<** (timeSettled + settledPeriod). This is due to the fact that the refferal mechanism will always be executed if everything else does not either revert or initiate betting.

The refferal mechanism is located in the function, outside of the checks, and is as follows:
```solidity
if (hasReferral[msg.sender]) { 
    address referralAddress_ = referralAddress[msg.sender];
    uint256 referralAmt = _bet/100; 
    referralBalance[_tokenSelection][referralAddress_] += referralAmt;
    betRake -= referralAmt;
}
```
If we place a bet during mentioned window, while amount returns within risk, an attacker could repeatedly call the placeBet function to increase the refferal amount and drain the contract.
```Solidity
function updateReferral(address referral) external { 
    if (msg.sender == referral || hasReferral[msg.sender]) {
        revert("Cannot refer yourself or change referral address");
    }
    referralAddress[msg.sender] = referral;
    hasReferral[msg.sender] = true;
}

function claimReferrals(address _token) external {
    uint256 balanceToClaim = referralBalance[_token][msg.sender];
    referralBalance[_token][msg.sender] = 0;
    IERC20(_token).transfer(msg.sender, balanceToClaim); 
}
```
### POC
```solidity
function testIncreaseRefferalAmountWithoutDepositOrBet() public {
    MockERC20 token = new MockERC20();
    address tokenAddress = address(token);
    token._mint(alice, 10000 * (10 ** 18)); 
    Game.Option optionA = Game.Option.A;
    Game.State stateNow;
    uint time = 10 minutes;
    uint min5 = 5 minutes + 1;
    uint balanceBeforePause;
    uint balanceAfterPause;
    uint reffAmountBeforePause;
    uint reffAmountAfterPause;

    vm.startPrank(owner);
    game.createPool(tokenAddress);
    vm.stopPrank();

    vm.startPrank(alice);
    token.approve(address(game), 10000 * (10 ** 18));
    game.houseDeposit(tokenAddress, 100 * (10 ** 18));
    vm.warp(time);
    uint timeNow = block.timestamp;
    
    game.updateReferral(bob);
    game.placeBet(tokenAddress, (1 * (10 ** 18)), optionA); // starts new game

    vm.warp(timeNow + min5); //note close time
    game.placeBet(tokenAddress, (1 * (10 ** 18)), optionA); //closes after bet
    balanceBeforePause = token.balanceOf(alice);
    reffAmountBeforePause = game.referralBalance(tokenAddress,bob);
    vm.stopPrank();

    vm.prank(owner);
    game.pauseGames(); // while state is settled, placing bet should be paused

    vm.prank(alice);
    game.placeBet(tokenAddress, (1 * (10 ** 18)), optionA); // Vulnerable
    balanceAfterPause = token.balanceOf(alice);
    reffAmountAfterPause = game.referralBalance(tokenAddress,bob);

    assertEq(balanceBeforePause, balanceAfterPause);
    assertTrue(reffAmountBeforePause < reffAmountAfterPause);
   }
```
### Recommendation
Since placing a bet should only happen when State.SETTLED && block.timestamp > (timeSettled + settledPeriod) or State.OPEN, there is no need to make the function accessible outside of the window. 
When state is either paused or closed, create a modifier for said state and restrict placing bets. 

### Review
Approved. Refferal mechanism as been moved to be included in the if statement. Meaning refferal will only be added when user sends betamount to the contract.

## [H-01] Not accounting for fee tokens will lead to last user(s) not being able to withdraw (all) funds due to imbalanced accounting - racingGame.sol
### Severity
**Impact:** High, depending on the fee/ amount deposited and withdrawn/ time, one or more users would lose their funds
**Likelihood:** Medium, impact starts as soon as first deposit where the impact is small and gradually increases + requires fee on transfer tokens

### Summary
Depositing fee on transfer tokens results in less tokens received than anticipated. This leads to incorrect internal accounting where the balance is lower than actual balance.
This is increased as time passes where one of the last users will pay for the imbalance.
### Details
The imbalance is increased everytime some one deposits, where high amount deposits lead to a higher difference. 
When depositing, amount of shares is calculated and saved according to:
```
uint256 shares = (amount - rake) * pool.totalShares / pool.totalBalance
pool.totalShares += shares;
pool.totalBalance += (amount - rake);
```
Here, `pool.totalBalance` will have a higher value than the actual balance of the contract since fee is deducted before being received. Users that deposit will be able to withdraw according to `uint256 amount`, which is a higher amount than received and used to calculate shares with. Withdrawing results in withdrawing a higher amount than it should be. 
The difference in received what should have been received adds up every time, which increases the impact.

Depending on how long this goes on and with which amounts, the last users could not be able to fully withdraw their funds up untill a x amount of users are not able to withdraw their funds since it will result in an underflow error.

Note: this could also be done via griefing but does cost a lot for the attacker.
### Recommendation
If fee on transfer tokens are going to be supported, implement a before and after transfer balance check to determine the amount which will be used to calculate the amount of shares:
```solidity
uint256 balanceBefore = token.balanceOf(address(this));
//transfer
uint256 balanceAfter = token.balanceOf(address(this));
uint256 betAmount = balanceAfter - balanceBefore;
```

### Review

## [H-02] Griefing attack could result in failing fulfillUint256() leading to lost funds of bets and temporarily halts contract - racingGame.sol/ HighLow.sol/ Flip.sol
### Severity
**Impact:** High, users that bet in minigames during downtime windows will lose their bet amount. Game contract comes to a complete halt until reset.
**Likelihood:** Medium, requires multiple calls in an attack

### Summary
API3 uses funds from a specified sponsorWallet to call `fulfillUint256()`. An attack to initiate this API3 call repeatedly drains the sponsorWallet and leads to API3 not being able to perform the `fulfillUint256()` callback.
This leads to users that bet in the minigame to not getting their requestId processed and 1. lose potential rewards 2. their bet amount.
In adittion, `racingGame.sol` comes to a closed state which needs to be reset to continue after refilling sponsorwallter.

### Details Mini games
API3 uses funds from a specified sponsorWallet to call `fulfillUint256()`. An attack to initiate this API3 calls repeatedly drains the sponsorWallet and leads to API3 not being able to perform the `fulfillUint256()` callback.

API# documentation mentions when calling `makeRequestUint256()`:
```
/// @dev This request will be fulfilled by the contract's sponsor wallet,
/// which means spamming it may drain the sponsor wallet.
```
See: https://docs.api3.org/guides/qrng/qrng-remix/

Also, when a call fails, API3 Airnode will retry within 300 blocks. When exceeding this window, API3 deletes the request and doesn't retry.

Spamming could be done via
1. `racingGame.placeBet()` with a low amount
2. `HighLow.placeBet()`/ `Flip.betFlip()`, which both call `makeRequestUint256()`. Both functions have the following check: 
```solidity
if(hasMinBet[tokenSelection] && _bet < minBet[tokenSelection]) {
    revert("Bet is too low, must cover operational costs");
}
```
If `hasMinBet` is set to false, the attack could be done with 1 wei. If it is set, the attack becomes harder but still possible if `minBet` is low enough.

When placing a bet, the returned `requestId` is captured and used to store the bet:
```solidity
bytes32 requestId = makeRequestUint256();  // Capture the returned requestId

betsByRequestId[requestId] = BetDetails({
    user: msg.sender,
    choice: _choice,
    token: tokenSelection,
    amount: _bet,
    amountAfterFee: _wager
});
```
The radom `requestId` is then used in `fullfillUint256()` to determine potential reward and send it:
```solidity
if (payout > 0) {
    mainContract.subHouseBalance(bet.token, payout, bet.user);
    emit WagerWon(bet.user, bet.token, payout, cardValue);
```
Since it's not possible to re-initialize `fullfillUint256()` with the same `requestId`, result is not calculated and bet amount + potential rewards are lost.

### Details Main game
When above mentioned happens, `racingGame.sol` will be stuck in a closed state until sponsor is refilled and state is reset.

in `racingGame.sol`, the duration of each game duration is saved in `bettingPeriod`. When the duration has passed, requestFinish() is either called when placing a bet or called manually.
```solidity
function requestFinish() public bettingOpenState { 
      if (block.timestamp > (gameStarted + bettingPeriod)) { 
          currentState = State.BETTING_CLOSED;
          makeRequestUint256(); //note API3 trigger to return randomness in callback fulfillUint256()
      } else {
        revert("Please wait at least five minutes after first bet this round");
      }
  }
```
What's notable is that `requestFinish()` sets the state to `BETTING_CLOSED`. `fulfillUint256()` sets the state to `SETTLED` again:
```solidity
function fulfillUint256(bytes32 requestId, bytes calldata data) public {
        require(expectingRequestWithIdToBeFulfilled[requestId], "Request ID not known");
        expectingRequestWithIdToBeFulfilled[requestId] = false;
        uint256 qrngUint256 = abi.decode(data, (uint256));

        _storeVrf(raceId, qrngUint256);

        currentState = State.SETTLED; //note here

        timeSettled = block.timestamp;

        updatePoolBalances();
        resetAccountingForNewRound();
        raceId++;

        emit ReceivedUint256(requestId, qrngUint256);
    }
```
When API3 fails to call the contract, it's stuck in closed state until state is reset via `pauseGames()` -> `startGames()` manually.

### Recommendation
The important part is to increase the difficulty or effort (funds) needed to perform the attack:
- Always have a `minBet` amount, preferably to a high enough value
- Check for sponsor wallet fund before initiating the call
- Reserve x% of bet amount for gas for the sponsor wallet

Do note that next to this the recommendation of M-01 also need to be implemented.

### Review
Approved. The `_checkSponsorWalletCoverageAtFactor` checks have been added to ensure minimum gas is available. The check is performed again after claiming max gas to ensure that the minimum required gas is available. Will revert if this is not the case, meaning user is not able to place a bet as a safeguard.

## [M-01] A failing API3 callback to fulfillUint256() could result in lost funds of bets and temporarily halts contract - racingGame.sol/ HighLow.sol/ Flip.sol
### Severity
**Impact:** High, users that bet in minigames during downtime windows will lose their bet amount. Game contract comes to a complete halt until reset.
**Likelihood:** Low, requires API3 to fail operations

### Summary 
Note, the impact is the same as H-02. H-02 focusses on the griefing attack and to avoid this, while M-01 points down on a other potential issue which could result in the same impact.
This could happen in any case API3 fails to perform the callback fulfillUint256() (e.g. due to being offline).

### Recommendation
There are different ways to remediate this. Though, since the contract is not centralized in a way that funds cannot be withdrawn by a single entity, ideally we would like the solution to respect this.

A good fit would be to implement a function to recall `makeRequestUint256()` when `expectingRequestWithIdToBeFulfilled[requestId] = true` (meaning `fullfillUint256()` has not been called) and x amount time has passed.
An example could be:
```solidity
function recallRequestUint256(uint requestPreviousId) external onlyOwner  {
  require (expectingRequestWithIdToBeFulfilled[requestPreviousId], "request not stuck");
  require (block.timestamp >= timeSettled + bettingPeriod + cooldownBeforeRecall);

  expectingRequestWithIdToBeFulfilled[requestPreviousId] = false;

    bytes32 requestId = airnodeRrp.makeFullRequest(
        airnode,
        endpointIdUint256,
        address(this),
        sponsorWallet,
        address(this),
        this.fulfillUint256.selector,
        ""
    );
    
    expectingRequestWithIdToBeFulfilled[requestId] = true;
    emit reRequestedUint256(requestPreviousId);
    emit RequestedUint256(requestId);
}
```
### Review
Approved. Refund mechanism is added. It's protected by the owner only to be called. expectingRequestWithIdToBeFulfilled[requestId] is set to false so can't be fullfilled after refund is done. minigameBets[requestId] is also deleted in racingGame at refund and user is not able to do anything with that entry between placing bet and refund.

## [L-01] Adding approved tokens could lead to DoS which permanently breaks the game - racingGame.sol
### Severity
**Impact:** Medium, breaks core functionality (betting completely stops until redeployed)
**Likelihood:** low, requires a lot of approved tokens

### Summary
`updatePoolBalances()` iterates through all the approved tokens to get the keys, set balances and delete pools. 
If the iteration exceeds gas limit due to the size of the array, `fulfillUint256()` will always revert which leads to the same issue as mentioned in M-01 and L-01.

### Details
`address[] public approvedTokens` holds approved tokens which are 1. used in `updatePoolBalances()` 2. increased using `createPool()`.
Increasing the approved tokens could happen due to increasing demand in new tokens. The problem is that there is no option to decrease the list. This means that even if certain pools are not used anymore but hold atleast 1 wei, they are included in the loop. This leads to `fulfillUint256()` always reverting.

### Recommendation
Implement a function that sets token choice to not approved and removed it from the array.
While only removing from array, it still keeps the option open to remove from housePool if users still have some funds deposited.

### Review

## [L-02] IERC20 transfer calls will fail with non-complaint tokens
### Severity
**Impact:** Low, would not be able to desposit or place a bet
**Likelihood:** Low

### Summary
High level call `transfer()` and `transferFrom()`, for example, will fail if a return value is not provided. This is the case with USDT on Ethereum network.
This makes it unable to use those tokens.

### Recommendation
Use OpenZeppelin’s SafeERC20

## [I-01] Contract addresses are still set to BLAST Testnet addresses
Contract addresses are still set to BLAST Testnet addresses. 
```
IBlast blast = IBlast(0x4300000000000000000000000000000000000002);
IERC20Rebasing wethRebasing = IERC20Rebasing(0x4200000000000000000000000000000000000023);
IERC20Rebasing usdbRebasing = IERC20Rebasing(0x4200000000000000000000000000000000000022);

```
Leaving this as informational as these contracts were used on testnet.
