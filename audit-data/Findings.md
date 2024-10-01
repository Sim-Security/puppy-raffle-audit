### [M-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, incrementing gas costs for future entrants

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle:players` array is, the more checks a new player will have to make. This means that the gas costs for players who enter right when the raffle starts will be dramatically lower than those who enter later. Every additional address in the `players` array, is an additional check the loop will have to make. 

**Note to students: This next line would likely be it's own finding itself. However, we haven't taught you about MEV yet, so we are going to ignore it.**
Additionally, this increased gas cost creates front-running opportunities where malicious users can front-run another raffle entrant's transaction, increasing its costs, so their enter transaction fails. 

**Impact:** The impact is two-fold.

1. The gas costs for raffle entrants will greatly increase as more players enter the raffle.
2. Front-running opportunities are created for malicious users to increase the gas costs of other users, so their transaction fails.

**Proof of Concept:** 

If we have 2 sets of 100 players enter, the gas costs will be as such:
- 1st 100 players: 6252039
- 2nd 100 players: 18067741

This is more than 3x as expensive for the second set of 100 players! 

This is due to the for loop in the `PuppyRaffle::enterRaffle` function. 

```javascript
        // Check for duplicates
@>      for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

<details>
<summary>Proof Of Code</summary>
Place the following test into `PuppyRaffleTest.t.sol`.

```javascript
function test_denialOfService() public {
    // The code sets the gas price to 1.
    vm.txGasPrice(1);

    // It creates an array of 100 addresses and assigns them to the 'players' array.
    uint256 playersNum= 100;
    address[] memory players = new address[](100);

    for (uint256 i=0; i < playersNum; i++) {
        players[i] = address(i);
    }

    // It measures the gas cost before and after calling the 'enterRaffle' function with the 'players' array. (first 100 players)
    uint256 gasStartA = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee*players.length}(players);
    uint256 gasCostA = (gasStartA - gasleft()) * tx.gasprice;

    address[] memory playersTwo = new address[](100);

    for (uint256 i=0; i < playersNum; i++) {
        playersTwo[i] = address(i+ playersNum);
    }

    // It creates another array of 100 addresses and assigns them to the 'playersTwo' array.
    // It measures the gas cost before and after calling the 'enterRaffle' function with the 'playersTwo' array. 
    uint256 gasStartB = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee*players.length}(playersTwo);
    uint256 gasCostB = (gasStartB - gasleft()) * tx.gasprice;

    // It logs the gas cost of the first 100 players and the gas cost of the second 100 players.
    // The gas cost of the second 100 players is expected to be higher than the gas cost of the first 100 players.
    console.log("Gas cost of first 100 players: %s", gasCostA);
    console.log("Gas cost of second 100 players: %s", gasCostB);

    // This test demonstrates a potential denial of service vulnerability where the gas cost keeps rising, making it harder for new players to enter the raffle.
    assert(gasCostB > gasCostA);
}

```
</details>

**Recommended Mitigation:** There are a few recommended mitigations.

1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check duplicates. This would allow you to check for duplicates in constant time, rather than linear time. You could have each raffle have a `uint256` id, and the mapping would be a player address mapped to the raffle Id. 

```diff
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;            
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }    
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

Alternatively, you could use [OpenZeppelin's `EnumerableSet` library](https://docs.openzeppelin.com/contracts/4.x/api/utils#EnumerableSet).

### [S-#] TITLE (Root Cause + Impact)

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:**