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

This EIP introduces changes to the EVM, making it recognize Multiple Native Assets (MNAs). ETH is made one of the Native Assets (NAs), while retaining its unique status in terms of how it is minted/burned, as well as remaining the only Native Asset (NA) that can be used to pay for EVM gas. 
The `MINT`, `BURN` and `BALANCES` opcodes are introduced to control the supply of NAs and query one's NAs balance. The `CALL` and `CALLCODE`, `CREATE` and `CREATE2`, `BALANCE`, `SELFBALANCE` and `CALLVALUE` opcodes are adapted to support NAs transfer, the NA-infused contract creation and the querying of the NAs-related information, respectively.
The `data` field of the transaction structure is adapted to support a collection of (`asset_id`, `asset_amount`) pairs.

## Motivation

The introduction of MNAs into the EVM addresses several limitations and inefficiencies present in the current Ethereum architecture, where ETH is the sole native asset. While ETH has served as the foundation for gas payments and the primary store of value within the ecosystem, the growing complexity of decentralized applications, decentralized finance protocols and cross-chain integrations has highlighted the need for a more versatile native asset system. Here are some of the main advantages of implementing MNAs in Ethereum:

  ### Enhanced Flexibility and User Experience:

    As the Ethereum ecosystem continues to expand, dApps and protocols increasingly require more specialized economic models that rely on native assets. Currently, developers must create  and manage custom tokens via ERC standards, which are less efficient and more complex (read: "unreasonable") to integrate natively into the EVM. MNAs allow for the direct implementation of diverse native assets within the EVM, simplifying development and enabling new types of applications, such as native stablecoins, governance tokens or protocol-specific assets.

  ### Support for DeFi and Financial Innovation:

    #### Native Support for Financial Instruments:

      DeFi protocols heavily rely on various assets, including stablecoins and derivative tokens. Natively supporting these assets within the EVM enhances their integration, improves performance, and reduces the need for complex workarounds. This support enables more efficient liquidity provision, trading, and collateralization mechanisms, which are critical for the continued growth and innovation of DeFi.

    #### Facilitating Cross-Chain and Multi-Asset Solutions:

      As the blockchain ecosystem becomes increasingly interconnected, the ability to natively support multiple assets within the EVM positions Ethereum as a more versatile and integrative platform. This capability is crucial for enabling seamless cross-chain interactions and creating new financial products that leverage assets from multiple blockchains.

  ### Improved Reliability via Consistent Asset Handling:

    Natively recognizing multiple assets within the EVM ensures consistent and reliable asset handling across all transactions and smart contracts. This standardization simplifies development, testing, and auditing processes, leading to more secure and predictable interactions.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

A global `token_id` -> `token_supply` mapping is introduced, to keep track of the existing NAs and their circulating supply. The supply of a NA is increased during the execution of the `MINT` opcode - and decreased during the execution of its `BURN` counterparty. The `token_id` of a NA is the Ethereum address of its associated smart contract. This mapping is also being used to validate the supported NAs (a NA is supported iff its id can be found in the mapping).

`ETH` becomes the "Base Asset" with the id and supply both equal to 0. `ETH` is the only NA with its supply not being tracked explicitly (i.e. the `ETH` supply is being determined just like it currently is).

The addition of new NAs is being decided via a consensus expressed by the ETH validators. For security and consistency reasons, the smart contracts representing the NAs MUST NOT be mutable/upgradeable. A smart contract MUST contain the "supply logic" **complete** and based **solely** on the `MINT` and `BURN` opcodes in order to be considered for being added as a NA.

### New Opcodes

#### `MINT` - `0xb0`
##### Gas: TBD
##### Stack inputs:
        - `recipient` (the address that the minted assets will be credited to)
        - `amount`
##### Stack outputs:
        - `success`


#### `BURN` - `0xb1`
##### Gas: TBD
##### Stack inputs:
        - `burner` (the address that the minted assets will be debited/burned from)
        - `amount`
##### Stack outputs:
        - `success`

#### `BALANCES` - `0xb2`
##### Gas: Dynamic, as a factor of the number of NAs owned by the queried account
##### Stack inputs:
        - `account` (the address the NA holdings of which are being queried)
##### Stack outputs:
        - the list of (`token_id`, `amount`) pairs, representing the NAs balances of the queried account

### Changes to The Existing Opcodes

#### `CALL`
##### Gas: Dynamic, as a factor of the transferred NAs number
##### Stack inputs:
        - `value` is replaced by `values`, containing a list of (`token_id`, `amount`) pairs

#### `CALLCODE`
##### Gas: Dynamic, as a factor of the transferred NAs number
##### Stack inputs:
        - `value` is replaced by `values`, containing a list of (`token_id`, `amount`) pairs

#### `CREATE`
##### Gas: Dynamic, as a factor of the transferred NAs number
##### Stack inputs:
        - `value` is replaced by `values`, containing a list of (`token_id`, `amount`) pairs

#### `CREATE2`
##### Gas: Dynamic, as a factor of the transferred NAs number
##### Stack inputs:
        - `value` is replaced by `values`, containing a list of (`token_id`, `amount`) pairs

#### `BALANCE`
##### Stack inputs:
        - `token_id` (the id of the NA the balance of which is being queried)

#### `SELFBALANCE`
##### Name: `SELFBALANCES`
##### Gas: Dynamic, as a factor of the number of NAs owned by the executing account
##### Stack outputs:
        - the list of (`token_id`, `amount`) pairs, representing the NAs balances of the account

#### `CALLVALUE`
##### Name: `CALLVALUES`
##### Gas: Dynamic, as a factor of the number of NAs transferred by the executing CALL
##### Stack outputs:
        - the list of (`token_id`, `amount`) pairs, representing the NAs transferred by the executing CALL

### Transaction structure



## Rationale

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

TBD

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
