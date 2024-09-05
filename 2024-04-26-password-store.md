---
title: Protocol Audit Report
author: Sami Gabor
date: April 26, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.png} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Sami Gabor\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle


Prepared by: [Sami Gabor](https://github.com/samigabor)

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
    - [\[H-1\] Storing the password on-chain makes it visible to anyone and no longer private](#h-1-storing-the-password-on-chain-makes-it-visible-to-anyone-and-no-longer-private)
    - [\[H-2\] `PasswordStore::setPassword` is missing the access control, meaning non-owner can change the password](#h-2-passwordstoresetpassword-is-missing-the-access-control-meaning-non-owner-can-change-the-password)
- [Informational](#informational)
    - [\[I-1\] Incorrect natspec for `PasswordStore::getPassword`](#i-1-incorrect-natspec-for-passwordstoregetpassword)

# Protocol Summary

PasswordStore is a protocol for storage and retrieval of a user's password. The protocol is designed to be used by a single user. Only the owner should be able to set and access this password.

# Disclaimer

I made all effort to find as many vulnerabilities in the code in the given time period, but hold no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

**The findings in this report correspond to the following commit hash:**
```
2e8f81e263b3a9d18fab4fb5c46805ffc10a9990
```

## Scope 
```
./src/
#-- PasswordStore.sol
```

## Roles
- owner: the user who can set and read the password
- outsiders: no one else should be able to set or read the password

# Executive Summary

*Spent X hours using Y tools etc*

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 1                      |
| Total    | 3                      |

# Findings
# High

### [H-1] Storing the password on-chain makes it visible to anyone and no longer private

**Description:** 
All data stored on-chin is visible to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword` function, which is intended to be only called by the owner of the contract.

Will show one such method of reading any data off chain below.

**Impact:** 
Anyone can read the private password, severely breaking the functionality of the protocol.

**Proof of Concept:** (proof of core)
The below test case shows how you can read the private state variable from the blockchain:
1. run a local chain `anvil`
2. deploy the contract locally `make deploy`
3. read second storage slot `cast storage <contract-address> 1`
4. convert the output from hex to string `cast parse-bytes32-string <output>`

**Recommended Mitigation:** 

The overall architecture of the protocol should be retaught.

One could encrypt the password off-chain, and the store the encrypted password on-chin.

This would require the user to remeber another password off-chain to decrypt the password.

However you'd also likely want to remove the view function as you wouldn't want the user to accidentally sent a tx with the password that decrypts your password.

Or encrypt it with the user's public key and decrypt it with the user's private key.


### [H-2] `PasswordStore::setPassword` is missing the access control, meaning non-owner can change the password

**Description:** 
`PasswordStore::setPassword` external function doesn't restrict access to the owner of the contract. This means that anyone can change the password, which is a critical vulnerability.

```js
    function setPassword(string memory newPassword) external {
@>      // @audit missing access control
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** 
Anyone can change the password, severely breaking the intended functionality of the contract.

**Proof of Concept:** (proof of core)

<details>
<summary>Code</summary>

```js
    function test_anyone_can_change_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        
        string memory expectedPassword = "myChangedPassword";
        vm.prank(randomAddress);
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```

</details>

**Recommended Mitigation:** 
Add access control to `PasswordStore::setPassword`
```js
    if(msg.sender != owner) revert PasswordStore__NotOwner();
```

# Informational
### [I-1] Incorrect natspec for `PasswordStore::getPassword`

**Description:** 
`PasswordStore::getPassword` natspec declares a `newPassword` which doesn't exist.

**Impact:** Incorrect natspec

**Recommended Mitigation:** 
Remove the `newPassword` param from the natspec

```diff
-    * @param newPassword The new password to set.
```
