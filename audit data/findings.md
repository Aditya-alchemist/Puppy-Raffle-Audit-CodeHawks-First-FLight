### [M-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, incrementing gas costs for future entrants

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array is, the more checks a new player will have to make. This means the gas costs for players who enter right when the raffle stats will be dramatically lower than those who enter later. Every additional address in the `players` array, is an additional check the loop will have to make. 



```solidity

//Audit DoS Attack
for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```


**Impact:** The gas costs for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at the start of a raffle to be one of the first entrants in the queue.

An attacker might make the `PuppyRaffle::entrants` array so big, that no one else enters, guarenteeing themselves the win.

**Proof of Concept:**

If we have 2 sets of 100 players enter, the gas costs will be as such:
- 1st 100 players: ~6503272 gas
- 2nd 100 players: ~42160976 gas

This more than 3x more expensive for the second 100 players.


<details>
<summary>PoC</summary>

```solidity

function test_Denial_Of_Service() public {
        vm.txGasPrice(1);
        uint256 numPlayers = 100;
        address[] memory players = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            players[i] = address(i);
        }
        //how much gas it costs to enter the raffle with many players
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(players);
        uint256 gasENd = gasleft();
        uint256 gasUsed = (gasStart - gasENd)*tx.gasprice;
        console.log("Gas used to enter raffle with %s players: %s", numPlayers, gasUsed);


        //for 200 players 
        numPlayers = 200;
        address[] memory players2 = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            players2[i] = address(i+100);
           
        }
        uint256 gasStart2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(players2);
        uint256 gasENd2 = gasleft();
        uint256 gasUsed2 = (gasStart2 - gasENd2)*tx.gasprice;
        console.log("Gas used to enter raffle with %s players: %s", numPlayers, gasUsed2);

        assert(gasUsed2 > gasUsed);


    }

```
</details>


**Recommended Mitigation:** Below are the reccomended mitigations

1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.

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
+           addressToRaffleId[newPlayers[i]] = raffleId;
        }

-       // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+           require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }
-       for (uint256 i = 0; i < players.length; i++) {
-           for (uint256 j = i + 1; j < players.length; j++) {
-               require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-           }
-       }
        emit RaffleEnter(newPlayers);
    }
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    }


```