# Collateral Technology

## Overview

In order to onboard assets as collateral into the Maker protocol, the DAO has developed a set of smart contracts which provide a secure way to lock collateral and launch vaults for users wanting to mint Dai. These smart contracts are use case specific, through iterations, abstractions have managed to standardize the technical integration which provides a reusable, secure and efficient process to onboard new collateral into the protocol. A set of [technical MIPs](https://mips.makerdao.com/mips/list?search=$%20%23technical) representing such abstractions have been ratified for their use in the protocol.

Smart contract Domain Teams within the protocol are the ones deciding on the final technical implementation for onboarding new collateral into the protocol. The following collateral onboarding technology provides an overview to teams wanting to propose collateral onboarding for the protocol.

![Crypto native and Real World Assets technology overview](<../.gitbook/assets/CES Collateral Tech - MakerDAO Collateral Tech (3).jpg>)

### Products

Before we dive deep into the collateral onboarding technology, we must step back and define the products that this technology serves.

MakerDAO's main product is Dai. Dai is a stable, decentralized, overcollateralized currency that is soft-pegged to the USD, available to anyone, anywhere.&#x20;

In order to mint Dai a set of methods (products) are made available.

* **Permissionless Vaults** which onboard crypto native collateral are the most widely used  method to generate Dai in the protocol. These vaults use collateral technology such as: GemJoins and CropJoins providing the ability to the protocol to onboard a wide rage of ERC20-like tokens. PSM technology also allows for onboarding stablecoin collateral and launching permissionless vault types which anyone can interact with, even though in the case of the USDC PSM, small trades are gas intensive and only bigger ones make sense economically, hence, these vault types are used by DEX router contracts mainly.&#x20;
* **Permissioned Vaults** are focused on onboarding collateral which requires a specific set of vault parameters due to either the collateral nature (i.e.: Real World Assets) or the size of the collateral and the interest of the DAO to capture such market share of a specific collateral type (i.e.: Institutional Vaults)
* **Protocol-to-Protocol Modules** are focused on providing Dai liquidity to other DeFi protocols by integrating directly with their smart contracts and targeting a case specific criteria. This criteria is the benefit the end-user of the DeFi protocol gets.

### Collateral Technology

#### GemJoins

These are adapter smart contracts which decouple ERC20-like tokens from the Maker protocol.

There is a base [gemjoin.sol](https://github.com/makerdao/dss/blob/master/src/join.sol) implementation which is modified on a per case basis as new needs come up when onboarding new tokens into the protocol. A list of current GemJoin versions and their uses cases is available [here](https://github.com/makerdao/dss-gem-joins).

#### MIP30: Farmable cUSDC Adapter (`CropJoin`)

These adapters are an evolution of the basic GemJoins. CropJoin adapters allow vault users to withdraw token rewards which are distributed by protocols such as cTokens from Compound and LP tokens from DEXs. These adapters were ratified in the [MIP30](https://mips.makerdao.com/mips/details/MIP30#compound-technical-risk) and the code base is available [here](https://github.com/makerdao/dss-crop-join).

#### MIP29: Peg Stability Module

This is a special vault type within the Maker protocol. Its purpose is to provide a 1:1 swap for a stablecoin to DAI and vice versa, effectively pegging DAI to the stablecoin. PSMs create arbitrage opportunities which help DAI price react to changing market demand. One of the main differences from other permissionless vaults is that PSMs do not keep track of user deposits; the stablecoin deposited in exchange of DAI becomes one single pool of assets that back DAI. These vaults lack an oracle and don't need a liquidation mechanism. The biggest PSM available at the time of writing is the USDC PSM which pegs DAI to USDC. PSMs provide a set of benefits as detailed in the [MIP29](https://mips.makerdao.com/mips/details/MIP29#creating-more-dai-utility-through-a-stable-peg). Code is available [here](https://github.com/makerdao/dss-psm).

#### MIP50: Dai Direct Deposit Module (D3M) - Stable Rate and Credit Line

The Dai Direct Deposit Module is the most recent technological innovation to mint Dai. It is a new vault type which is intended to be a protocol-to-protocol integration, making it a permissioned vault type. D3Ms are automated through a permissionless function which can be called at a regular basis to check a certain market condition. From the market condition we derive to main use cases:

**Stable Rate use case**

The first D3M iteration is the Aave D3M which deposits Dai into the Aave protocol and takes aDai as collateral. The market condition to target is a specific aDai borrow rate on Aave, by automatically managing the amount of Dai deposited up to a Maker Governance specified limit. Users from lending protocols such as Aave benefit from having a stable borrow rate for Dai.

**Debt ceiling use case**

Instead of targeting a borrow rate on a lending protocol, a D3M can be used to target an amount of liquidity to a pool of Dai which is used as capital to be borrowed by entities in the real world. This is a new use case which Maker is currently exploring with a set of partners that are innovating in the intersection of on-chain liquidity and real world lending.

More information is available in the [MIP50](https://mips.makerdao.com/mips/details/MIP50) and in this [repository](https://github.com/makerdao/dss-direct-deposit).

#### MIP59: DssCharter

This technology operates under two modes: permissionless and permissioned. It features an origination fee and specific negotiated parameters which together align the borrower and the protocol into a long-term relationship.

More information is available in the [MIP59](https://mips.makerdao.com/mips/details/MIP59#component-summary) and in this [repository](https://github.com/makerdao/dss-charter).

#### MIP21: Real World Assets - Off-Chain Asset Backed Lender

This is a permissioned vault technology which uses a set of smart contracts to mint Dai for RWA backed borrowers, who engage with the DAO in a real world deal enforced by local jurisdiction regulation. In this setup, an asset (or pool of assets) is used as collateral to back the Dai loan the DAO enters with the real world partner. This collateral is represented by an ERC20 token minted specifically for each permissioned vault. An optional actor in this setup, a Security Agent, is authorized on the input and output conduits with the task to verify a set of transaction criteria which must be followed during the deal life cycle.

More information is available in the [MIP21 ](https://mips.makerdao.com/mips/details/MIP21#sentence-summary)and in this [repository](https://github.com/makerdao/MIP21-RWA-Example).



The above technology and product definitions are in constant evolution as the protocol and the market growth. New [MIPs](https://mips.makerdao.com/mips/details/MIP0) are constantly proposed, discussed in the forum and ratified or not through governance [voting](https://vote.makerdao.com/). We encourage the community to propose new collateral, suggest use cases and tech to support it by following the MIP processes defined.
