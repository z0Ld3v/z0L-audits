# z0L-security-portfolio

Welcome! This repository serves as a comprehensive collection showcasing my expertise in smart contract security audits. I plan to expand this portfolio significantly over time, so stay tuned for more detailed analyses.

## Overview

Smart contract security is crucial in blockchain development. By thoroughly reviewing code and identifying vulnerabilities, we ensure the robustness of decentralized applications and protocols. This repository aims to:

- Document existing audit reports.
- Provide summaries and insights into findings and security best practices.
- Demonstrate our comprehensive approach to smart contract security.

## Current Audit Reports

### **1. PasswordStore Audit Report**

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

## Upcoming Reports

More audits are on the way! I will continue adding valuable analyses of various smart contracts and protocols.

## Contributing

Contributions are welcome! If you have suggestions or improvements, feel free to open an issue or pull request.
