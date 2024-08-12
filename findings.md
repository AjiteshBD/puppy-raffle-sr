
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