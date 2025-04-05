# Unauthorized Token Transfers via Unprotected `payWithERC20`

**Platform:** Sherlock Audit  
**Project:** Crestal Network Audit (April 2025)  
**Severity:** High  
**Status:** Duplicate submission (valid but unpaid)  
**Reported On:** [Insert exact date you reported it]  
**GitHub Issue:** [Direct URL to your Sherlock GitHub issue]

---

## Summary

The `payWithERC20` function in the Payment contract was publicly accessible without proper access controls, allowing any external caller to transfer ERC-20 tokens from users who previously granted token allowances to the contract. This vulnerability allowed an attacker to drain user tokens without explicit authorization.

---

## Impact

- Unauthorized withdrawal and loss of user ERC-20 tokens.
- Loss of funds leading to potential financial damage and loss of trust.

---

## Root Cause

`payWithERC20` function visibility was mistakenly set to `public` without requiring caller authentication.

---

## Recommended Fix

- Change the visibility of `payWithERC20` from `public` to `internal`.
- Implement access control modifiers or signature verification to limit external access.

---

## Notes

- **Duplicate:** Although this was a valid finding, it was not paid out as it was previously submitted by other auditors.
