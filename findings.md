
## M-1: Looping through players to check duplicates in `PuppyRaffle::enterRaffle` is a potential Denial of service (DoS) making it expensive for new enterants as the size of the array grows.

**Description:**
The `PuppyRaffle::enterRaffle` function loops through array of `PuppyRaffle::players` to check for duplicates which make this gas expensive operation over the time as the array size grow and new player comes entering. This make the older player have a advantage to enter the raffle at low cost but the each new `players` enters it will need to check make it more gas expensive to enter the same raffle.


```javascript
// @audit DoS attack
@> for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**impact:**
The gas costs will dramatically increase as the new players enter the raffle which will discourge the later new player to enter raffle causing a rush situation at the start of raffle to at the first position of the queue.

The Hacker might make `PuppyRaffle::entrants` array so big that nobody can enter which guantee his win.

**Proof of Concept**
If we have 2 sets of player enter the raffle the gas cost will as such
 - 1st 100 players ~6252128 gas
 - 2nd 100 players ~18068218 gas
  
This 3x more expensive for the second set of players to enter.

<details>
<summary>Proof of Code</summary>
Place the following <code>PuppyRaffleTest.t.sol</code>.

```javascript
    function testDenialOfService() public {
        vm.txGasPrice(1);

        uint256 playerNum = 100;

        address[] memory players = new address[](playerNum);
        for (uint256 i = 0; i < playerNum; i++) {
            players[i] = address(i);
        }

        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsed = (gasStart - gasEnd) * tx.gasprice;

        console.log("gas cost for first 100 players", gasUsed);

        // Second set of 100 players
        address[] memory secondPlayers = new address[](playerNum);
        for (uint256 i = 0; i < playerNum; i++) {
            secondPlayers[i] = address(i + playerNum);
        }

        uint256 secondGasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * secondPlayers.length}(secondPlayers);
        uint256 secondGasEnd = gasleft();
        uint256 secondGasUsed = (secondGasStart - secondGasEnd) * tx.gasprice;

        console.log("gas cost for first 100 players", secondGasUsed);
        assert(gasUsed < secondGasUsed);
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
