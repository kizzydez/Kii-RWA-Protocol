![kii-chain-logo](logo.png)

# CW3643 - T-REX PROTOCOL On Kii-Chain

## Overview

This document outlines the detailed implementation plan for the T-REX (Token for Regulated EXchanges) protocol using CosmWasm smart contracts on Kii-Chain. The T-REX protocol is designed for compliant issuance and management of security tokens on blockchain networks.

## Contracts overview
| Contract                       | Purpose                                                                                             | Connected with                        |
| ------------------------------ | --------------------------------------------------------------------------------------------------- | ------------------------------------- |
| CW_20 Base                     | Holds an asset as a CW_20                                                                           | Owner Roles, Compliance Registry      |
| Owner Roles                    | Registers owner and special permissions to addresses                                                | -                                     |
| Compliance Registry            | Stores compliance modules and verifies compliance on each of them                                   | Compliance whitelist wrapper          |
| Compliance whitelist Wrapper   | A wrapper for compliance modules that whitelists some addresses, ignoring them                      | Compliance country, Compliance Claims |
| Compliance Country Restriction | A sample compliance contract that denies identities from specific countries                         | On Chain ID                           |
| Compliance Claims              | Compliance that denies usage of tokens from users that do not have claims for it                    | Claim topics, On Chain ID             |
| On Chain ID                    | Registers identities for users and allows granting management of certain aspects in their behalf    | -                                     |
| Claim Topics                   | Stores address<>trust level used to block access to tokens, can only be changed by a trusted issuer | Trusted Issuers                       |
| Trusted Issuer                 | Maps special addresses that can change claims                                                       | -                                     |
| Agent Roles                    | -- In the works --                                                                                  | -                                     |

## Action flows
The relation between contracts is not super clear on the first moment, but they create a system where asset usage is limited to only those that were accepted by a trusted issuer. That trusted issuer can only accept users that decided to trust it in the first place.

### New asset
With this flow we create a new asset. The asset will be limited to those that have the required claim trust level, which can only be added by the trusted issuer. The transfer of the asset will have the token pair as intermediate, which will be trusted for compliance on this specific token.

- Create a new `CW20 base`
- Set up a token pair with liquidity of that Asset <> Kii
- Add a link between this token and the wrapped compliance modules into the `compliance registry` (any number of them)
  - The wrap on a compliance module allows us to whitelist the token pair so it can do transfers without having an identity
  - This needs to be done by someone with compliance manager role
- Add a claim need between this token and a level of trust on the `claim topics`.
  - This needs to be done by someone with claim manager role
- Whitelist the token pair address on the `compliance whitelist wrappers` (one for each compliance module)
  - This needs to be done by someone with compliance manager role

### User registering
For the user to be able to interact with assets, it needs to register an identity, state it trusts a common issuer and receive a specific trust level from the trusted issuer.

- User creates it's own identity on `On Chain ID` contract
- User allows the trusted issuer to manage its claims, on `On chaind ID` contract
- Trusted issuer emits a specific trust level for the user, on the `claim topics`

### First buy
The buy will be done with a liquidity pool as a middleman, handling prices. The user needs to be registered beforehand and have a proper trust level issued so they can receive the asset.

- Buy token from liquidity pool
- Do a transfer from the pool to the buyer, directly on the `cw20 base` of the T-REX

# Contracts and Their Methods

### 1. CW20 T-REX Token

Extends the CW20 standard with T-REX functionalities and implements permissioned transfers.

#### Storage
- `token_info`: TokenInfo
  - Name, symbol, decimals, supply and minter(opt)
  - Minter has an addr and can have a cap
- `balances`: Map<Addr, Uint128>
- `allowances`: Map<(Addr, Addr), AllowanceInfo>
  - uint with an expiration date
- `identity_registry`: Addr
- `compliance`: Addr

#### Execute
- `transfer(recipient: Addr, amount: Uint128) -> Result<Response>`
  - Description: Transfers tokens to a recipient if they are verified and compliant.
  - Interaction: Calls `is_verified` on Identity Registry and `can_transfer` on Modular Compliance.

- `transfer_from(owner: Addr, recipient: Addr, amount: Uint128) -> Result<Response>`
  - Description: Transfers tokens on behalf of the owner if the recipient is verified and compliant.
  - Interaction: Similar to `transfer`, but checks allowances first.

- `send(contract: Addr, amount: Uint128, msg: Binary) -> Result<Response>`
  - Msg is weird `Binary::from(r#"{"some":123}"#.as_bytes());`
- `burn(amount: Uint128) -> Result<Response>`
- `mint(contract: Addr, amount: Uint128) -> Result<Response>`
- Increase/Decrease Allowance
  - Can also change expiration

#### Query
- `balance(address: Addr) -> BalanceResponse`
- `token_info() -> TokenInfoResponse`
- `is_verified_transfer(from: Addr, to: Addr, amount: Uint128) -> IsVerifiedTransferResponse`
  - Description: Checks if a transfer would be valid without executing it.
  - Interaction: Calls Identity Registry and Modular Compliance.

### 2. Identity Registry

Manages investor identities and eligibility.

#### Storage
- `identity_storage`: Addr
- `trusted_issuers_registry`: Addr
- `claim_topics_registry`: Addr
- `agents`: Vec<Addr>

#### Execute
- `register_identity(address: Addr, identity: Addr) -> Result<Response>`
  - Description: Registers a new identity for an address.
  - Interaction: Calls Identity Registry Storage to store the mapping.

- `update_identity(address: Addr, identity: Addr) -> Result<Response>`
  - Description: Updates an existing identity for an address.

- `remove_identity(address: Addr) -> Result<Response>`
  - Description: Removes an identity for an address.

#### Query
- `is_verified(address: Addr) -> IsVerifiedResponse`
  - Description: Checks if an address is verified (has required claims).
  - Interaction: Queries Identity Registry Storage, Trusted Issuers Registry, and Claim Topics Registry.

### 3. Identity Registry Storage

Stores the mapping of wallet addresses to identity contracts.

#### Storage
- `identities`: Map<Addr, Addr>

#### Execute
- `add_identity(address: Addr, identity: Addr) -> Result<Response>`
- `remove_identity(address: Addr) -> Result<Response>`

#### Query
- `get_identity(address: Addr) -> GetIdentityResponse`

### 4. Trusted Issuers Registry

Manages the list of trusted claim issuers. OwnerRole::IssuersRegistryManager can change trusted issuers.

#### Storage
- `trusted_issuers`: Map<Addr, TrustedIssuer>

#### Execute
- `add_trusted_issuer(issuer: Addr, claim_topics: Vec<u32>) -> Result<Response>`
- `remove_trusted_issuer(issuer: Addr) -> Result<Response>`

#### Query
- `is_trusted_issuer(issuer: Addr) -> IsTrustedIssuerResponse`
- `get_trusted_issuers() -> GetTrustedIssuersResponse`

### 5. Claim Topics Registry

Stores the required claim topics for token eligibility. It is used for compliance. Only those with role OwnerRole::ClaimRegistryManager can change this.

#### Storage
- `token_claim_topics`: Vec<u32>
  - Token addresses are mapped to claims
  - Claim is defined by a topic uint128 and an active bool

#### Execute
- `add_claim_topic(topic: u32) -> Result<Response>`
- `remove_claim_topic(topic: u32) -> Result<Response>`

#### Query
- `get_claims_for_token() -> StdResult<Vec<Uint128>>`

### 6. Modular Compliance

Implements transfer restriction rules. Only OwnerRole::ComplianceManager can execute changes. For now country compliance and claims compliance are the ones in use.

#### Storage
- `modules`: Vec<Addr>
- `bound_tokens`: Vec<Addr>

#### Execute
- `add_module(module: Addr) -> Result<Response>`
- `remove_module(module: Addr) -> Result<Response>`
- `bind_token(token: Addr) -> Result<Response>`

#### Query
- `can_transfer(from: Addr, to: Addr, amount: Uint128) -> CanTransferResponse`
  - Description: Checks if a transfer complies with all modules.
  - Interaction: Calls `check_transfer` on each compliance module.

### 7. ONCHAINID

Represents user identities (similar to ERC-734/735).

#### Storage
- `keys`: Map<bytes32, Key>
- `claims`: Map<bytes32, Claim>

#### Execute
- `add_key(key: bytes32, purpose: u32, key_type: u32) -> Result<Response>`
- `remove_key(key: bytes32) -> Result<Response>`
- `add_claim(topic: u32, scheme: u32, issuer: Addr, signature: Vec<u8>, data: Vec<u8>, uri: String) -> Result<Response>`
- `remove_claim(topic: u32) -> Result<Response>`

#### Query
- `get_key(key: bytes32) -> GetKeyResponse`
- `get_claim(topic: u32) -> GetClaimResponse`

### 8. T-REX Factory

Deploys and configures T-REX token suites.

#### Storage
- `implementation_authority`: Addr
- `deployed_tokens`: Vec<Addr>

#### Execute
- `deploy_token_suite(token_info: TokenInfo, compliance_info: ComplianceInfo, identity_info: IdentityInfo) -> Result<Response>`
  - Description: Deploys a new T-REX token suite.
  - Interaction: Instantiates all necessary contracts and configures them.

#### Query
- `get_deployed_tokens() -> GetDeployedTokensResponse`

### 9. Implementation Authority

Manages contract upgrades and versioning.

#### Storage
- `versions`: Map<String, ContractVersion>
- `current_version`: String

#### Execute
- `add_version(name: String, contracts: Vec<(String, Addr)>) -> Result<Response>`
- `set_current_version(name: String) -> Result<Response>`

#### Query
- `get_current_version() -> GetCurrentVersionResponse`
- `get_version(name: String) -> GetVersionResponse`

## Implementation Roadmap

1. Implement CW20 T-REX Token
2. Implement Identity Registry and Identity Registry Storage
3. Implement Trusted Issuers Registry and Claim Topics Registry
4. Implement Modular Compliance
5. Implement ONCHAINID
6. Implement T-REX Factory
7. Implement Implementation Authority
8. Integrate all contracts and perform thorough testing
9. Conduct security audit
10. Deploy to Kii-Chain testnet
11. Final testing and adjustments
12. Deploy to Kii-Chain mainnet

## Testing Strategy

- Implement unit tests for each contract function
- Develop integration tests for contract interactions
- Create end-to-end scenarios to test the entire T-REX suite
- Perform fuzz testing to uncover edge cases
- Conduct governance testing to ensure proper access control

