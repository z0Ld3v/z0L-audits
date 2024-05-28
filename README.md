<div align="center">
  <img src="./logo.png" alt="Logo Alt Text" width="200" />
</div>

# z0L-security-portfolio

Welcome! This repository serves as a comprehensive collection showcasing my expertise in smart contract security audits. I plan to expand this portfolio significantly over time, so stay tuned for more detailed analyses.

## Overview

Smart contract security is crucial in blockchain development. By thoroughly reviewing code and identifying vulnerabilities, I ensure the robustness of decentralized applications and protocols. This repository aims to:

- Document existing audit reports.
- Provide summaries and insights into findings and security best practices.
- Demonstrate my comprehensive approach to smart contract security.

## Current Audit Reports

### 1. [PasswordStore Audit Report](./2024-05-06-password-store-audit.pdf)

<details>
  <summary><strong>[H-1] Storing the password on-chain makes it visible to anyone and no longer private</strong></summary>

- **Description:** All data stored on-chain is visible to anyone and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be private and accessed only through `PasswordStore::getPassword`. However, anyone can read the private password directly from the chain.
- **Impact:** This vulnerability severely breaks the functionality of the protocol.
- **Proof of Concept:** 
  1. Create a locally running chain.
  2. Deploy the contract to the chain.
  3. Run a storage tool to extract data from the contract's storage slot.
- **Recommended Mitigation:** Encrypt the password off-chain before storing it on-chain to keep the actual password secure.

</details>

<details>
  <summary><strong>[H-2] PasswordStore::setPassword has no access controls</strong></summary>

- **Description:** `PasswordStore::setPassword` is accessible to any user, allowing them to change the stored password.
- **Impact:** Any user can call this function and change the stored password, breaking the core functionality of the contract.
- **Proof of Concept:** Add the provided test code to `PasswordStore.t.sol`.
- **Recommended Mitigation:** Implement access control to ensure only the contract owner can modify the password.

</details>

<details>
  <summary><strong>[I-1] PasswordStore::getPassword natspec comment is incorrect</strong></summary>

- **Description:** The function signature differs from what is indicated in the comments, potentially misleading developers.
- **Impact:** This issue may cause confusion for developers.
- **Recommended Mitigation:** Remove the incorrect natspec parameter line.

</details>

### 2. [PuppyRaffle Audit Report](./2024-05-27-puppy-raffle-audit.pdf)

## High

<details>
  <summary><strong>[H-1] Reentrancy attack in `PuppyRaffle::refund` allows entrant to drain raffle balance</strong></summary>

- **Description:** The `PuppyRaffle::refund` function does not follow CEI (Checks, Effects, Interactions) and as a result, enables an attacker to drain the raffle balance.
- **Impact:** All fees paid by raffle entrants could be stolen by the malicious participant.
- **Proof of Concept:** 
  1. User enters the raffle.
  2. Attacker sets up contract with a `fallback` function that calls `PuppyRaffle::refund`.
  3. Attacker enters the raffle.
  4. Attacker calls `PuppyRaffle::refund` from their attack contracts, draining the raffle balance.
- **Recommended Mitigation:** To prevent this, the `PuppyRaffle::refund` function should update the `players` array before making the external call. Additionally, move the event emission up as well.

</details>

<details>
  <summary><strong>[H-2] Weak randomness in `PuppyRaffle::selectWinner` allows users to influence or predict winners</strong></summary>

- **Description:** Hashing `msg.sender`, `block.timestamp`, and `block.difficulty` together generates a predictable number. Malicious users can manipulate the random number generator to predict winners themselves.
- **Impact:** Any user can influence the winner of the raffle, making the entire raffle worthless if it becomes a gas war as to who wins the raffles.
- **Proof of Concept:** 
  1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how to participate.
  2. Users can manipulate their `msg.sender` to result in their address being used to select the winner.
  3. Users can revert their `selectWinner` transaction if they don't like the winner or resulting puppy.
- **Recommended Mitigation:** Use a cryptographically proven random number generator like [Chainlink VRF](https://docs.chain.link/docs/vrf-contracts/).

</details>

<details>
  <summary><strong>[H-3] Integer overflow of `PuppyRaffle::totalFees` loses fees</strong></summary>

- **Description:** In solidity versions prior to `0.8.0`, integers were subject to overflow. In `PuppyRaffle::selectWinner`, `totalFees` are accumulated for the `feeAddress` to collect later in `PuppyRaffle::withdrawFees`. However, if the `totalFees` variable overflows, the `feeAddress` will not receive any fees, leaving fees stuck in the contract permanently.
- **Impact:** The `feeAddress` will not receive any fees, leaving fees stuck in the contract permanently.
- **Proof of Concept:** 
  1. Conclude a raffle of 4 players.
  2. Have 89 players enter a new raffle and conclude the raffle.
  3. `totalFees` will overflow, preventing fee withdrawal.
- **Recommended Mitigation:** Use a newer version of solidity and a `uint256` instead of `uint64` for `PuppyRaffle::totalFees`. Use the `SafeMath` library of OpenZeppelin to mitigate this issue in version 0.7.6 of solidity. Remove the balance check from `PuppyRaffle::withdrawFees`.

</details>

## Medium

<details>
  <summary><strong>[M-1] Looking through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack</strong></summary>

- **Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. The longer the `PuppyRaffle::players` array grows, the more gas it will cost for new players to enter the raffle. This can be exploited by an attacker to block new entries indefinitely.
- **Impact:** The gas costs for raffle entrants can be increased to the point where it becomes prohibitively expensive to join the raffle, effectively blocking new entries.
- **Proof of Concept:** 
  1. Enter 100 players.
  2. Enter another 100 players. The gas cost for the second set of 100 players is nearly 3 times higher than the first set.
- **Recommended Mitigation:** Consider allowing duplicates or using a mapping to check for duplicates.

</details>

<details>
  <summary><strong>[M-2] Unsafe cast of `PuppyRaffle::fee` loses fees</strong></summary>

- **Description:** In `PuppyRaffle::selectWinner`, there is a type cast of a `uint256` to a `uint64`. This is an unsafe cast, and if the `uint256` is larger than `type(uint64).max`, the value will be truncated.
- **Impact:** The `feeAddress` will not collect the correct amount of fees, leaving fees permanently stuck in the contract.
- **Proof of Concept:** 
  1. A raffle proceeds with a little more than 18 ETH worth of fees collected.
  2. The line that casts the `fee` as a `uint64` hits.
  3. `totalFees` is incorrectly updated with a lower amount.
- **Recommended Mitigation:** Set `PuppyRaffle::totalFees` to a `uint256` instead of a `uint64`, and remove the casting.

</details>

<details>
  <summary><strong>[M-3] Smart contract wallets raffle winners without a `receive` or a `fallback` function will block the start of a new contest</strong></summary>

- **Description:** The `PuppyRaffle::selectWinner` function is responsible for resetting the lottery. If the winner is a smart contract wallet that rejects payment, the lottery would not be able to restart.
- **Impact:** The `PuppyRaffle::selectWinner` function could revert many times, making a lottery reset impossible.
- **Proof of Concept:** 
  1. 10 smart contract wallets enter the lottery without a fallback or receive function.
  2. The lottery ends.
  3. The `selectWinner` function wouldn't work, even though the lottery is over!
- **Recommended Mitigation:** Create a mapping of addresses -> payout so winners can pull their funds out themselves.

</details>

## Low

<details>
  <summary><strong>[L-1] `PuppyRaffle:getActivePlayerIndex` returns 0 for non-existent players and for players at index 0, causing a player at index 0 to incorrectly think they have not entered the raffle</strong></summary>

- **Description:** If a player is in the `PuppyRaffle::players` array at index 0, this will return 0. But according to the natspec, it will also return 0 if the player is not in the array.
- **Impact:** A player at index 0 may incorrectly think they have not entered the raffle, and attempt to enter the raffle again, wasting gas.
- **Proof of Concept:** 
  1. User enters the raffle, they are the first entrant.
  2. `PuppyRaffle::getActivePlayerIndex` returns 0.
  3. User thinks they have not entered the raffle due to the function documentation.
- **Recommended Mitigation:** Revert if the player is not in the array rather than returning 0. Alternatively, return an `int256` where the function returns -1 if the player is not active.

</details>

### Gas Optimizations

<details>
  <summary><strong>[G-1] Unchanged state variables should be declared constant or immutable</strong></summary>

- **Description:** Reading from storage is much more expensive in gas than reading from a constant or immutable variable.
- **Instances:** 
  - `PuppyRaffle::raffleDuration` should be `immutable`
  - `PuppyRaffle::commonImageUri` should be `constant`
  - `PuppyRaffle::rareImageUri` should be `constant`
  - `PuppyRaffle::legendaryImageUri` should be `constant`

</details>

<details>
  <summary><strong>[G-2] Storage variables in a loop should be cached</strong></summary>

- **Description:** Every time `players.length` is called, it reads from storage, which is more expensive than reading from memory.
- **Proof of Concept:**
  ```diff
  +        uint256 playersLength = players.length;
  -        for (uint256 i = 0; i < players.length - 1; i++) {
  +        for (uint256 i = 0; i < playersLength - 1; i++) {
  -            for (uint256 j = i + 1; j < players.length; j++) {
  +            for (uint256 j = i + 1; j < playersLength; j++) {
                  require(players[i] != players[j], "PuppyRaffle: Duplicate player");
              }
          }
  ```
</details>

### Informational / Non-Critical

<details>
  <summary><strong>[I-1] Solidity pragma should be specific, not wide</strong></summary>
Description: Consider using a specific version of Solidity in your contracts instead of a wide version.
Instance:
solidity
Copy code
pragma solidity 0.8.18;
</details>
<details>
  <summary><strong>[I-2] Using an outdated version of Solidity is not recommended</strong></summary>
Description: Using an old version of Solidity prevents access to new Solidity security checks.
Recommendation: Deploy with a recent version of Solidity (at least 0.8.18) with no known severe issues.
</details>
<details>
  <summary><strong>[I-3] Missing checks for `address(0)` when assigning values to address state variables</strong></summary>
Description: Check for address(0) when assigning values to address state variables.
Instances:
```solidity
feeAddress = _feeAddress; // src/PuppyRaffle.sol [Line: 69]
feeAddress = newFeeAddress; // src/PuppyRaffle.sol [Line: 217]
```
</details>
<details>
  <summary><strong>[I-4] `PuppyRaffle::selectWinner` does not follow CEI, which is not a good practice</strong></summary>
Description: It's best to follow CEI (Checks, Effects, Interactions) to keep your code clean.
Proof of Concept:
```diff
-    (bool success,) = winner.call{value: prizePool}("");
-    require(success, "PuppyRaffle: Failed to send prize pool to winner");
    _safeMint(winner, tokenId);
+    (bool success,) = winner.call{value: prizePool}("");
+    require(success, "PuppyRaffle: Failed to send prize pool to winner");
```
</details>
<details>
  <summary><strong>[I-5] Use of "magic" numbers is not recommended</strong></summary>
Description: It can be confusing to see number literals in a codebase, and it's much more readable if the numbers are given a name.
Proof of Concept:
```solidity
uint256 prizePool = (totalAmountCollected * 80) / 100;
uint256 fee = (totalAmountCollected * 20) / 100;
```
Instead, use:
```solidity
uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
uint256 public constant FEE_PERCENTAGE = 20;
uint256 public constant POOL_PRECISION = 100;
```
</details>
<details>
  <summary><strong>[I-6] State changes are missing events</strong></summary>
Description: Emit events for state changes for better traceability.
</details>
<details>
  <summary><strong>[I-7] `PuppyRaffle::_isActivePlayer` is never used and should be removed</strong></summary>
Description: The function PuppyRaffle::_isActivePlayer is never used and should be removed.
Proof of Concept:
```diff
-    function _isActivePlayer() internal view returns (bool) {
-        for (uint256 i = 0; i < players.length; i++) {
-            if (players[i] == msg.sender) {
-                return true;
-            }
-        }
-        return false;
-    }
```
</details>
Upcoming Reports
More audits are on the way! I will continue adding valuable analyses of various smart contracts and protocols.

## Contributing
Contributions are welcome! If you have suggestions or improvements, feel free to open an issue or pull request.
