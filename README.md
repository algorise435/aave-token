# Aave Token Design

AAVE is an ERC-20–compatible token. It implements governance-inspired features and allows Aave to bootstrap the rewards program for safety and ecosystem growth.  
This document explains the main features of AAVE, its monetary policy, and the redemption process from LEND.

## Roles

The initial AAVE token implementation does not have any admin roles configured. The contract will be proxied using OpenZeppelin’s implementation of the EIP-1967 Transparent Proxy pattern. The proxy has an Admin role, and the Admin of the proxy contract will be set upon deployment to the Aave governance contracts.

## ERC-20

The AAVE token implements the standard methods of the ERC-20 interface. A balance-snapshot feature has been added to keep track of user balances at specific block heights. This helps with the Aave governance integration of AAVE.  
AAVE also integrates the EIP-2612 `permit` function, which allows gasless transactions and one-tx approval/transfer.

# LendToAaveMigrator

A smart contract for LEND token holders to execute the migration to the AAVE token, using part of the initial emission of AAVE for it.

The contract is covered by a proxy whose owner will be AAVE governance. Once governance passes the corresponding proposal, the proxy will be connected to the implementation and LEND holders will be able to call the `migrateFromLend()` function, which—after LEND approval—will pull LEND from the holder’s wallet and transfer back an equivalent AAVE amount defined by the `LEND_AAVE_RATIO` constant.

One trade-off of `migrateFromLend()` is that, as the AAVE total supply will be lower than LEND, the `LEND_AAVE_RATIO` will always be > 1, causing a loss of precision for amounts of LEND that are not multiples of `LEND_AAVE_RATIO`.  
For example, a person sending 1.000000000000000022 LEND with `LEND_AAVE_RATIO == 100` will receive 0.01 AAVE, losing the value of the last 22 small units of LEND.

Given the current value of LEND and the expected value of AAVE, a lack of precision for fewer than `LEND_AAVE_RATIO` small units represents a value several orders of magnitude smaller than $0.01. We evaluated potential solutions:

1. Rounding half up the amount of AAVE returned from migration — opens attacks where users purposely migrate less than `LEND_AAVE_RATIO` to gain from rounding.
2. Returning back the excess LEND — leaves LEND in circulation forever, which is not the intended final state.
3. Requiring users to migrate only amounts that are multiples of `LEND_AAVE_RATIO` — creates considerable UX friction.

None presents a better outcome than the implemented solution.

## The Redemption Process

The first step to bootstrap AAVE emission is to deploy the AAVE token contract and the LendToAaveMigrator contract. This task will be performed by the Aave team. Upon deployment, the ownership of the AAVE Proxy and the LendToAaveMigrator will be set to Aave Governance.

To start the LEND redemption process, the Aave team will create an AIP (Aave Improvement Proposal) and submit it to governance. Once approved, the proposal will activate the LEND/AAVE redemption process and the ecosystem incentives, marking the initial emission of AAVE on the market.

As migration proceeds, the supply of LEND will be progressively locked within the new AAVE smart contract, while an equivalent amount of AAVE is issued. The amount of AAVE equivalent to the LEND tokens burned in the initial phase of the AAVE protocol will remain locked in the LendToAaveMigrator contract.

## Technical Implementation

### Changes to OpenZeppelin Contracts

In this implementation, we applied the following changes to OpenZeppelin:

- In `/contracts/open-zeppelin/ERC20.sol`, lines 44–45, `_name` and `_symbol` were changed from `private` to `internal`.
- We extended `Initializable` and created `VersionedInitializable`. Differences:
  1. The boolean `initialized` is replaced with `uint256 latestInitializedRevision`.
  2. The `initializer()` modifier fetches the implementation’s revision via `getRevision()` defined in the implementation contract.
  3. The `initializer()` modifier enforces that only an implementation with a higher revision number than the current one can be initialized.

These changes allow calling `initialize()` on multiple implementations, which is not possible with the original OpenZeppelin `Initializable`.

### `_beforeTokenTransfer` Hook

We override `_beforeTokenTransfer` in the base ERC-20 to include:

1. Snapshotting balances every time an action involving a transfer occurs (mint, burn, `transfer`, or `transferFrom`). If the account transfers to itself, no snapshot is created.
2. A call to the Aave governance contract forwarding the same input parameters as the `_beforeTokenTransfer` hook. It’s assumed the Aave governance contract is a trusted party responsible for controlling any potential reentrancy if it calls back into `AaveToken`. If the account transfers to itself, no governance interaction occurs.

## Development Deployment

For development, deploy `AaveToken` and `LendToAaveMigrator` to a local network:

```bash
npm run dev:deployment


For any other network, you can run the deployment in the following way

```
npm run ropsten:deployment
```

You can also set an optional `$AAVE_ADMIN` enviroment variable to set an ETH address as the admin of the AaveToken and LendToAaveMigrator proxies. If not set, the deployment will set the second account of the `accounts` network configuration at `buidler.config.ts`.

## Mainnet deployment

You can deploy AaveToken and LendToAaveMigrator to the mainnet network via the following command:

```
AAVE_ADMIN=governance_or_ETH_address
LEND_TOKEN=lend_token_address
npm run main:deployment
```

The `$AAVE_ADMIN` enviroment variable is required to run, set an ETH address as the admin of the AaveToken and LendToAaveMigrator proxies. Check `buidler.config.ts` for more required enviroment variables for Mainnet deployment.

The proxies will be initialized during the deployment with the `$AAVE_ADMIN` address, but the smart contracts implementations will not be initialized.

## Enviroment Variables

| Variable                | Description                                                                         |
| ----------------------- | ----------------------------------------------------------------------------------- |
| \$AAVE_ADMIN            | ETH Address of the admin of Proxy contracts. Optional for development.              |
| \$LEND_TOKEN            | ETH Address of the LEND token. Optional for development.                            |
| \$INFURA_KEY            | Infura key, only required when using a network different than local network.        |
| \$MNEMONIC\_\<NETWORK\> | Mnemonic phrase, only required when using a network different than local network.   |
| \$ETHERESCAN_KEY        | Etherscan key, not currently used, but will be required for contracts verification. |

## Audits

The Solidity code in this repository has undergone 2 traditional smart contracts' audits by Consensys Diligence and Certik, and properties' verification process by Certora. The reports are:
- [Consensys Diligence](https://diligence.consensys.net/audits/2020/07/aave-token/)
- [Certik](audits/AaveTokenReport_CertiK.pdf)
- [Certora](audits/AaveTokenVerification_by_Certora.pdf)

## Current Mainnet contracts (25/09/2020)

- **AaveToken proxy**: [0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9](https://etherscan.io/address/0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9)
- **AaveToken implementation**: [0xea86074fdac85e6a605cd418668c63d2716cdfbc](https://etherscan.io/address/0xea86074fdac85e6a605cd418668c63d2716cdfbc)
- **LendToAaveMigrator proxy**: [0x317625234562b1526ea2fac4030ea499c5291de4](https://etherscan.io/address/0x317625234562b1526ea2fac4030ea499c5291de4)
- **LendToAaveMigrator implementation**: [0x86241b6c526998582556f7c0342d8863b604b17b](https://etherscan.io/address/0x86241b6c526998582556f7c0342d8863b604b17b)

## Credits

For the proxy-related contracts, we have used the implementation of our friend from [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-sdk/).

## License

The contents of this repository are under the AGPLv3 license.
