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


