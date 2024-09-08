---
title: Multiple Native Assets
description: Introduces Multiple Assets, recognized by the VM alongside ETH
author: Paul Razvan Berg (@PaulRBerg), Iaroslav Mazur (@IaroslavMazur)
discussions-to: https://ethereum-magicians.org[]
status: Draft
type: Standards Track
category: Core
created: 2024-09-09
---

## Abstract

This EIP introduces changes to the EVM, making it recognize Multiple Native Assets (MNAs). ETH becomes one of the Native Assets (NAs), while retaining its unique status in terms of how it is minted/burned, as well as remaining the only Native Asset (NA) that can be used to pay for EVM gas. 
The `MINT`, `BURN` and `BALANCES` opcodes are introduced to control the supply of NAs and query an account's NAs balance. The `CALL` and `CALLCODE`, `CREATE` and `CREATE2`, `BALANCE`, `SELFBALANCE` and `CALLVALUE` opcodes are adapted to support the transfer of NAs, NA-infused contract creation and querying NA-related information, respectively.
The `value` field in transaction structure is adapted to support a collection of (`asset_id`, `asset_amount`) pairs.

## Motivation

The addition of MNAs to the EVM addresses several limitations and inefficiencies present in the current Ethereum architecture, where ETH is the sole native asset. While ETH has historically served as the foundation for gas payments and the primary store of value within the ecosystem, the expanding complexity of decentralized applications, decentralized finance protocols and cross-chain integrations has highlighted the need for a more versatile native asset system. Here are some of the main advantages of implementing MNAs in Ethereum:

  ### Enhanced Flexibility and User Experience

    As Ethereum continues to grow, dApps and protocols increasingly require specialized economic models that leverage native assets. Currently, developers must create  and manage custom tokens via ERC standards, which are less efficient and more complex (read: "unreasonable") to integrate natively into the EVM. MNAs allow for the direct support of diverse native assets within the EVM itself, simplifying development and enabling new types of applications, such as native stablecoins, governance tokens and protocol-specific assets.

  ### Support for DeFi and Financial Innovation

    #### Native Support for Financial Instruments

      DeFi protocols depend heavily on a variety of assets, such as stablecoins and derivative tokens. Natively supporting these assets within the EVM improves their usability, boosts performance and eliminates the need for complex workarounds. This support increases the efficiency of liquidity provision, trading and collateralization mechanisms, all of which are vital for the continued innovation of DeFi.

    #### Facilitating Cross-Chain and Multi-Asset Solutions:

      As the blockchain ecosystem grows increasingly interconnected, the ability to natively support multiple assets within the EVM positions Ethereum as a more versatile and integrative platform. This capability is crucial for enabling seamless cross-chain interactions and creating new financial products that leverage assets from multiple blockchains.

  ### Improved Reliability via Consistent Asset Handling

    By natively recognizing multiple assets within the EVM, Ethereum ensures consistent and reliable asset handling across all transactions and smart contract interactions. This standardization simplifies development, testing and auditing, leading to more secure and predictable outcomes.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

A global `token_id` -> `token_supply` mapping is introduced to keep track of the existing NAs and their circulating supply. This mapping is also being used to validate the supported NAs (a NA is supported iff its id can be found in the mapping). The supply of a NA increases as a result of the `MINT` opcode execution - and decreases during the execution of its `BURN` counterparty. The `token_id` of a NA is the Ethereum address of its associated smart contract.

`ETH` becomes the "Base Asset", with both the ID and supply initialized to 0. `ETH` is the only NA the supply of which is not tracked explicitly (i.e. its supply is being determined just like it currently is).

The addition of new NAs is decided by consensus among the ETH validators. For security and consistency, the smart contracts representing the NAs MUST NOT be mutable/upgradeable. A smart contract MUST control its supply exclusively via the `MINT` and `BURN` opcodes to be considered for being added as a NA.

### New Opcodes

#### `MINT` - `0xb0`
- **Gas**: TBD

- **Stack inputs**:
  - `recipient` (Ethereum address that to which the minted assets are credited)
  - `amount`

- **Stack outputs**:
  - `success` (Boolean indicating success)

Note: the asset that is being minted MUST be a NA.

#### `BURN` - `0xb1`
- **Gas**: TBD
- **Stack inputs**:
  - `burner` (Ethereum address from which the assets burned)
  - `amount`
- **Stack outputs**:
  - `success` (Boolean indicating success)

Note: the burner MUST have a sufficient NA balance to cover the `amount`.

#### `BALANCES` - `0xb2`
- **Gas**: Dynamic, proportional to the number of NAs owned by the queried account
- **Stack inputs**:
  - `account` (Ethereum address to query the NA balance of)
- **Stack outputs**:
  - List of (`token_id`, `amount`) pairs representing the NAs balances of the account

### Existing Opcodes Adaptations

#### `CALL`
- **Gas**: Dynamic, proportional to the number of transferred NAs
- **Stack inputs**:
  - `value` is replaced by `values` which contains a list of (`token_id`, `amount`) pairs

#### `CALLCODE`
- **Gas**: Dynamic, proportional to the number of transferred NAs
- **Stack inputs**:
  - `value` is replaced by `values` which contains a list of (`token_id`, `amount`) pairs

#### `CREATE`
- **Gas**: Dynamic, proportional to the number of transferred NAs
- **Stack inputs**:
  - `value` is replaced by `values` which contains a list of (`token_id`, `amount`) pairs

#### `CREATE2`
- **Gas**: Dynamic, proportional to the number of transferred NAs
- **Stack inputs**:
  - `value` is replaced by `values` which contains a list of (`token_id`, `amount`) pairs

#### `BALANCE`
- **Stack inputs**:
  - `token_id` (the id of the NA the balance of which is being queried)

#### `SELFBALANCE`
- **Name**: `SELFBALANCES`
- **Gas**: Dynamic, proportional to the number of NAs owned by the executing account
- **Stack outputs**:
  - List of (token_id, amount) pairs representing the NAs held by the executing account.

#### `CALLVALUE`
- **Name**: `CALLVALUES`
- **Gas**: Dynamic, proportional to the number of NAs transferred by the executing CALL
- **Stack outputs**:
  - List of (token_id, amount) pairs representing the NAs transferred by the executing CALL

### Transaction structure
The `value` field in the transaction structure is renamed to `transferred_assets`, and instead of a single value, now contains a list of (`token_id`, `amount`) pairs.


## Rationale

An alternative to the proposed opcode-based approach was the Precompile-based approach, which presents the following advantages:
  - No new opcodes would need to be introduced.
  - Existing EVM opcodes would remain unchanged.
  - As a result, no modifications to smart contract languages (that compile into EVM opcodes) would be required.

However, the Precompile-based approach also introduces several disadvantages:
  - The minting, burning, transferring and balance checking of NAs would be more complicated for end users, as the complexity could not be abstracted through smart contract languages.
  - Users would need to handle low-level data manipulations to encode inputs for Precompile functions and decode their outputs.
  - Transferring NAs while calling contract functions would require encoding not only the NA data but also the function selector and the arguments for that function.
  - Transferring multiple NAs in a single transaction would further complicate these operations.

Considering the balance of these pros and cons, the opcode-based approach was chosen for its simplicity and efficiency in handling NAs at the EVM level.

## Backwards Compatibility

The changes introduced by this EIP affect existing systems that rely on the `value` field in the transaction structure or any of the modified opcodes.

Front-end Ethereum libraries (e.g. web3js, wagmi) need to adapt to the new transaction structure. At the same time, smart contract languages (e.g. Solidity, Vyper) need to adapt to the changed opcodes - and add support for the newly introduced opcodes. Ethereum wallets, explorers and development tools will require updates to support MNAs.

The authors recommend providing extended development time and offering reference implementations to help ease the transition for developers of these tools and services.

## Test Cases

<!--
  This section is optional for non-Core EIPs.

  The Test Cases section should include expected input/output pairs, but may include a succinct set of executable tests. It should not include project build files. No new requirements may be introduced here (meaning an implementation following only the Specification section should pass all tests here.)
  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`. External links will not be allowed

  TODO: Remove this comment before submitting
-->

## Security Considerations

This EIP introduces several security risks related to gas handling, malicious assets and system integrity. Below are the key considerations and how they are mitigated.

1. Gas Accounting for NAs-related Opcodes
Opcodes dealing with MNAs (e.g. `MINT`, `BURN`, `BALANCES`) must charge gas proportionate to the number of NAs involved. Otherwise, attackers could exploit the system by executing operations on many NAs, leading to potential DDoS attacks.

Mitigation: Gas costs are dynamically adjusted based on the number of processed NAs, ensuring that operations scale in cost alongside the transaction complexity.

2. Malicious or Misbehaving Native Assets
An asset that becomes a Native Asset (NA) may later behave maliciously, causing disruptions in the network.

Mitigation: A governance mechanism allows validators to vote on removing misbehaving NAs. Only immutable, non-upgradeable contracts can become NAs, reducing the risk of post-approval issues.

3. Supply Control Risks
The `MINT` and `BURN` opcodes introduce potential risks of improper asset creation or destruction, which could lead to inflation or deflation of an asset’s supply.

Mitigation: Only the NAs are allowed to execute the `MINT` and `BURN` opcodes - and exclusively to control their own supply.

4. Cross-Contract NA Transfers
Inter-contract NAs transfers could lead to vulnerabilities if contracts are not properly equipped to handle multiple assets.

Mitigation: Contracts must validate asset transfers correctly, with guidance for developers on standard patterns to ensure safe cross-contract interactions.

5. Governance and Validator Consensus
Validator voting on adding or removing NAs introduces potential risks of governance attacks or collusion.

Mitigation: Safeguards such as quorum requirements and transparent voting processes will help prevent governance manipulation. Auditable records of decisions will ensure accountability.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
