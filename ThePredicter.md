### [H-01]: Lack of Player's Addresses Authentication in `ThePredicter::makePrediction` Function, Permitting Unauthorized Addresses to Submit Predictions.

**Summary:** The `ThePredicter::makePrediction` function is designed to be called by addresses within the player's array. However, any address can currently call this function and submit a prediction.

**Vulnerability Details:**

```solidity
function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }
        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

Place the following test case in `ThePredicter.test.sol`

```javascript
address public anyone = makeAddr("anyone");

function test_anyoneCanMakePrediction() public {
        vm.startPrank(anyone);
        vm.deal(anyone, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.First
        );
        vm.stopPrank();
    }
```

I created a random address "anyone," and called the `ThePredicter::makePrediction` function directly without approving the address "anyone" as a player through the `ThePredicter::approvePlayer` function.

```javascript

Ran 1 test for test/ThePredicter.test.sol:ThePredicterTest
[PASS] test_anyoneCanMakePrediction() (gas: 133671)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.34ms (481.50µs CPU time)

Ran 1 test suite in 11.34ms (2.34ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

The test case is passing meaning any random address can make a prediction that goes against the intended functionality of the protocol

To verify that the address "anyone" was not included in the `players` array, I added a getter function to the contract to retrieve all the addresses in the `players` array.

```diff
+     function getPlayers() public view returns (address[] memory) {
+        return players;
+    }
```

As expected, the verification was successful because the function returned an empty array.

```javascript
 address public anyone = makeAddr("anyone");
    function test_anyoneCanMakePrediction() public {
        vm.startPrank(anyone);
        vm.deal(anyone, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.First
        );

        vm.stopPrank();


   @>      thePredicter.getPlayers();
    }
===============================================================================

├─ [2695] ThePredicter::getPlayers() [staticcall]
 @>     │   └─ ← [Return] []
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.47ms (164.50µs CPU time)
```

**Impact:**

1. Unauthorized users can submit predictions, undermining the fairness of the competition. This disrupts the intended structure where only registered and approved players can participate.

2. The system is designed to have a maximum of 30 players. Allowing anyone to call the function can easily surpass this limit, leading to potential logistical and operational issues within the competition.

3. The inclusion of unauthorized predictions can skew the results, causing critical issues during the prize distribution phase. Legitimate players who followed the registration process might be unfairly disadvantaged or deprived of their rightful rewards.

1) User enters the raffle
2) Attackers sets up a contract with a `fallback` function that calls `PuppyRaffle::refund` balance.

**Tools Used:**

Manual Review

**Recommendations:**

Add a specific error, `error ThePredicter__NotAPlayer()`, within an `if` condition in your `ThePredicter::makePrediction`  function to validate that the caller is a player.

```diff

+     error ThePredicter__NotAPlayer();


 function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {

+ if (playersStatus[msg.sender] != Status.Approved) {
+            revert ThePredicter__NotAPlayer();
+        }

        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```
