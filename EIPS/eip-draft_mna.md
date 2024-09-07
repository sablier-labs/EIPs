---
title: Multiple Native Assets
description: Introduces Multiple Assets, recognized by the VM alongside ETH
author: Paul Razvan Berg (@PaulRBerg), Iaroslav Mazur (@IaroslavMazur)
discussions-to: https://ethereum-magicians.org[]
status: Draft
type: Standards Track
category: Core
created: 2024-09-04
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

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.
-->

- the global mapping
- opcodes vs precompile
- the `BALANCES` opcode vs w/o it

## Backwards Compatibility

<!--

  This section is optional.

  All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

No backward compatibility issues found.

## Test Cases

<!--
  This section is optional for non-Core EIPs.

  The Test Cases section should include expected input/output pairs, but may include a succinct set of executable tests. It should not include project build files. No new requirements may be introduced here (meaning an implementation following only the Specification section should pass all tests here.)
  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`. External links will not be allowed

  TODO: Remove this comment before submitting
-->

## Security Considerations

<!--
  All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. For example, include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
