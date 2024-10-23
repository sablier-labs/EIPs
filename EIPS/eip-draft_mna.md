---
title: Multiple Native Tokens
description:
  Introduces Multiple Native Tokens, with balances stored in the global VM state and support for transfers in EVM
  opcodes
author: Paul Razvan Berg (@PaulRBerg), Iaroslav Mazur (@IaroslavMazur)
discussions-to: https://ethereum-magicians.org[]
status: Draft
type: Standards Track
category: Core
created: TBD
---

## Abstract

TODO: rewrite this at the end

This EIP introduces significant changes to the EVM in order to make it recognize Multiple Native Tokens (MNTs, or just
NTs) and foster innovation in L2s. The balances of these NTs are stored in the global VM state, and ETH becomes one of
the NTs while retaining its unique status of the only NT that can be used to pay for EVM gas. The `MINT`, `BURN`,
`BALANCEOF`, and `CALLVALUES` opcodes are introduced to control the supply of NTs and query an account's NT balances.
The `CALL2`, `DELEGATECALL2`, `CALLCODE2`, `NTCREATE`, `NTCREATE2` opcodes are introduced to handle the transfer of NTs
and the NT-infused contract creation, respectively. Existing opcodes and transactions are adapted to refer to the
default NT, which is `ETH`. A new transaction type is introduced in which the `value` field is replaced with a
collection of (`token_id`, `token_amount`) pairs.

## Motivation

Implementing Multiple Native Tokens in the EVM offers several compelling advantages over traditional ERC-20 smart
contracts, fostering innovation and improving user experience.

### Native Support for Financial Instruments

Storing token balances in the VM state unlocks the potential for sophisticated financial instruments to be implemented
at the protocol level. This native integration facilitates features such as recurring payments and on-chain incentives
without the need for complex smart contract interactions. For instance, platforms could natively provide yield to token
holders or execute airdrops natively, similar to how rollups like Blast offer yield for ETH holders. Extending this
capability to any token enhances utility and encourages users to engage more deeply with the network.

### Elimination of Two-Step "Approve" and "Transfer"

By embedding token balances into the VM state, the cumbersome process of approving tokens before transferring them is
eliminated. Token transfers can be seamlessly included into smart contract calls, simplifying transaction flows and
reducing the number of steps users must take. This streamlined process not only enhances the user experience but also
reduces gas costs associated with multiple contract calls, making interactions more efficient and cost-effective.

### Encouraging Experimentation on Layer 2 Solutions

The proposed model aims to encourage innovation on Ethereum L2s by providing a flexible framework for token management.
EVM rollups can experiment with this design to develop new paradigms in decentralized finance (DeFi), gaming, and
beyond. By enabling tokens to have native properties and interactions, developers are empowered to explore features that
could lead to more robust and versatile applications. This experimentation is vital for the evolution of the Ethereum
ecosystem, as it fosters advancements that can benefit the broader community.

## Prior Art

This EIP has been inspired by FuelVM's
[Native Assets](https://docs.fuel.network/docs/sway/blockchain-development/native_assets/) design, as well as its
[SRC-20: Native Asset](https://docs.fuel.network/docs/sway-standards/src-20-native-asset/) standard.

The key distinction between Fuel's Native Assets and this EIP is that, in our proposal, each contract is limited to a
single native token (NT). A contract can mint only one NT, and the contract's address itself serves as the NT's ID.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT
RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

For increased security and consistency, the token contracts representing the NTs SHOULD NOT use an upgradeability
pattern.

Note: Since the EVM stack can support only up to 1024 elements, there is a natural limit to the number of tokens that
can be transferred during the execution of a single opcode. Given that a token pair takes 2 stack elements and the
number of transferred tokens need to be stored as a stack element, the maximum number of tokens that can be transferred
can be calculated as follows:

$$
(1024 - 1 - [NON_NT_ARGS]) / 2
$$

For example, a single `NTCALL` opcode can transfer a total of (1024 - 1 - 6) / 2 = 508 tokens.

### State Changes

A global `token_id` -> `token_supply` mapping is introduced to keep track of the existing NTs and their circulating
supply. This mapping is also used to validate the supported NTs. An NT exists if and only if its ID can be found in the
mapping. The supply of an NT increases as a result of executing the `MINT` opcode, and decreases as a result of
executing the `BURN` opcode. The `token_id` of an NT is the Ethereum address of its associated smart contract.

`ETH` becomes the 'Base Token', with its ID and supply initialized to zero. `ETH` is the only NT whose supply is not
tracked explicitly, i.e., its supply is determined just like it currently is.

### New Opcodes

#### `MINT` - `0xb0`

- **Gas**: TBD
- **Stack inputs**:
  - `recipient`: the address to which the minted tokens are credited
  - `amount`
- **Stack outputs**:
  - `success`: a Boolean indicating success

#### `BURN` - `0xb1`

- **Gas**: TBD
- **Stack inputs**:
  - `burner`: the address from which the tokens are burned
  - `amount`
- **Stack outputs**:
  - `success`: a Boolean indicating success

Note: the burner MUST have an NT balance that is at least equal to `amount`.

#### `BALANCEOF` - `0xb2`

- **Gas**: TBD
- **Stack inputs**:
  - `token_id`: the ID of the NT to query the balance of
  - `address`: the address to query the balance of
- **Stack outputs**:
  - `balance`: the NT balance of the given address

#### `CALLVALUES` - `0xb3`

- **Gas**: Dynamic, proportional to the number of NTs transferred by the executing call
- **Stack inputs**: None
- **Stack outputs**:
  - `transferred_tokens_length`: the number of transferred tokens
  - The list of `transferred_tokens_length` (`token_id`, `token_amount`) pairs

#### `NTCALL` - `0xb4`

- **Gas**: Dynamic, proportional to the number of transferred NTs
- **Stack inputs**:

  - `gas`: amount of gas to send to the sub context to execute. The gas that is not used by the sub context is returned
    to this one
  - `address`: the account which context to execute
  - `transferred_tokens_length`: the number of transferred tokens
  - The list of `transferred_tokens_length` (`token_id`, `token_amount`) pairs
  - `argsOffset`: byte offset in the memory in bytes, the calldata of the sub context
  - `argsSize`: byte size to copy (size of the calldata)
  - `retOffset`: byte offset in the memory in bytes, where to store the return data of the sub context
  - `retSize`: byte size to copy (size of the return data)

- **Stack outputs**:
  - `success`: return 0 if the sub context reverted, 1 otherwise

#### `NTCALLCODE` - `0xb6`

- **Gas**: Dynamic, proportional to the number of transferred NTs
- **Stack inputs**:

  - `gas`: amount of gas to send to the sub context to execute. The gas that is not used by the sub context is returned
    to this one
  - `address`: the account which code to execute
  - `transferred_tokens_length`: the number of transferred tokens
  - The list of `transferred_tokens_length` (`token_id`, `token_amount`) pairs
  - `argsOffset`: byte offset in the memory in bytes, the calldata of the sub context
  - `argsSize`: byte size to copy (size of the calldata)
  - `retOffset`: byte offset in the memory in bytes, where to store the return data of the sub context
  - `retSize`: byte size to copy (size of the return data)

- **Stack outputs**:
  - `success`: return 0 if the sub context reverted, 1 otherwise

#### `NTCREATE` - `0xb7`

- **Gas**: Dynamic, proportional to the number of transferred NTs
- **Stack inputs**:

  - `transferred_tokens_length`: the number of transferred tokens
  - The list of `transferred_tokens_length` (`token_id`, `token_amount`) pairs
  - `offset`: byte offset in the memory in bytes, the initialization code for the new account
  - `size`: byte size to copy (size of the initialization code)

- **Stack outputs**:
  - `address`: the address of the deployed contract, 0 if the deployment failed.

#### `NTCREATE2` - `0xb8`

- **Gas**: Dynamic, proportional to the number of transferred NTs
- **Stack inputs**:

  - `transferred_tokens_length`: the number of transferred tokens
  - The list of `transferred_tokens_length` (`token_id`, `token_amount`) pairs
  - `offset`: byte offset in the memory in bytes, the initialization code of the new account
  - `size`: byte size to copy (size of the initialization code)
  - `salt`: 32-byte value used to create the new account at a deterministic address

- **Stack outputs**:
  - `address`: the address of the deployed contract, 0 if the deployment failed

### Existing Opcodes Adaptations

#### Balance Query

The following opcodes are adapted to query the balance of the default NT, which is `ETH`:

- `BALANCE`
- `SELFBALANCE`
- `CALLVALUE`

#### Contract Creation

The `value` field in the following opcodes will refer to the default NT, which is `ETH`:

- `CREATE`
- `CREATE2`

#### Calling Contracts

The `value` field in the following opcodes will refer to the default NT, which is `ETH`:

- `CALL`
- `CALLCODE`

### Transaction structure

#### New Transaction Types

A new EIP-1559 transaction type is introduced to support the transfer of NTs.

The `value` field in the transaction structure is replaced by the number of tokens transferred
(`transferred_tokens_length`), followed by the (`token_id`, `token_amount`) pairs.

#### EVM Transactions

All existing EVM transactions are still valid.

- A zero `value` is equivalent to an empty `transferred_tokens` list.
- A non-zero `value` is equivalent to a list containing a single pair with `ETH`'s `token_id` (which is zero) and the
  `value` as `token_amount`.

## Rationale

An alternative to the proposed opcode-based approach was to use precompiles, which would have worked as follows:

- No new opcodes.
- Existing EVM opcodes would remain unchanged.
- As a result, no modifications to smart contract languages would be required.

However, the precompile-based approach also has disadvantages:

- Users would be required to handle low-level data manipulations to encode inputs to precompile functions and decode
  their outputs. This would lead to a subpar user experience.
- It would require major architectural changes to the EVM implementation, as precompiles are not designed to be
  stateful.

Considering this, the opcode-based approach was chosen for its simplicity and efficiency in handling NTs at the EVM
level.

## Backwards Compatibility

This EIP does not introduce any breaking changes to the existing Ethereum protocol. However, it adds substantial new
functionality that requires consideration across various layers of the ecosystem.

Front-end Ethereum libraries, such as web3.js and wagmi, will need to adapt to the new transaction structures introduced
by Multiple Native Tokens (MNTs). These libraries must update their interfaces and transaction handling mechanisms to
accommodate the inclusion of token transfers within smart contract calls and the absence of traditional "approve" and
"transfer" functions.

Smart contract languages like Solidity will need to incorporate support for the newly introduced opcodes associated with
MNTs. This includes adapting compilers and development environments to recognize and compile contracts that interact
with tokens stored in the VM state.

Additionally, Ethereum wallets, block explorers, and development tools will require updates to fully support MNTs.
Wallets must be capable of managing multiple native token balances, signing new types of transactions, and displaying
token information accurately. Explorers need to parse and present the new transaction formats and token states, while
development tools should facilitate debugging and deployment in this enhanced environment.

To ensure a smooth transition, the authors recommend a gradual deployment process. This phased approach allows
developers, users, and infrastructure providers to adapt incrementally. By introducing MNTs in stages, the ecosystem can
adjust to the new functionalities, verify compatibility, and address any issues that arise, ensuring that every
component behaves correctly throughout the integration period.

## Reference Implementation

TBD

## Security Considerations

This EIP introduces a few security risks related to malicious tokens and system integrity. Below are the key
considerations and how they are mitigated.

1. **Malicious or Misbehaving Native Tokens**: a token that becomes a Native Token (NT) may later behave maliciously,
   causing disruptions in the network.

Mitigation: encourage users to prefer interacting with immutable, non-upgradeable NTs. Mitigation: A governance
mechanism allows validators to vote on removing misbehaving NTs. Only immutable, non-upgradeable contracts can become
NTs, reducing the risk of post-approval issues.

2. **Cross-Contract NT Transfers**: inter-contract NTs transfers could lead to lost tokens if contracts are not properly
   equipped to handle multiple tokens.

Mitigation: Contracts must validate token transfers correctly, with guidance for developers on standard patterns to
ensure safe cross-contract interactions. Existing EVM contracts should be audited and updated to handle NTs.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
