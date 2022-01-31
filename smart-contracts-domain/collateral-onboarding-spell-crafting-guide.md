# Collateral Onboarding Spell Crafting Guide

Spells are essentially contracts that do something (or some things) once. They are generally used to change parameters inside the Maker Protocol like adding new collateral types, changing existing vault variables (such as debt ceilings, liquidation amounts, and stability fees), and also managing MKR and DAI distributions.

The engineering process is quite straightforward, but at the same time is specific to each case and also requires comprehensive testing.

Currently, the main spell repositories are located at [spells-goerli ](https://github.com/mardao/spells-goerli)and[ spells-mainnet](https://github.com/mardao/spells-mainnet). Working development branch follows a _exec-YY-MM-DD_ branch naming according to the sprint end date, so the Goerli repo should be forked, thoroughly tested, and then handed over to the Protocol Engineering Core Unit for Mainnet execution.

Past implementations of spells can be found in the archive/ folder of each repository alongside test files.

### Useful links

* dss-exec-lib:[https://github.com/makerdao/dss-exec-lib/](https://github.com/makerdao/dss-exec-lib/)
* ilk-registry: [https://github.com/makerdao/ilk-registry](https://github.com/makerdao/ilk-registry)
* dss-gem-joins: [https://github.com/makerdao/dss-gem-joins](https://github.com/makerdao/dss-gem-joins)
* Maker Glossary: [https://docs.makerdao.com/other-documentation/system-glossary](https://docs.makerdao.com/other-documentation/system-glossary)
* Maker Change Log for contract addresses: [https://changelog.makerdao.com/](https://changelog.makerdao.com)
* Bytes32 converter: [https://web3-type-converter.onbrn.com/](https://web3-type-converter.onbrn.com)
* Auditing Executive Spells MD by Derek (PE): [https://hackmd.io/ELaxHNjBRn-cVMF6JcML4w?view](https://hackmd.io/ELaxHNjBRn-cVMF6JcML4w?view)

## Intro

Spells are essentially `DssSpellAction/SpellAction` contracts extended from the `DssAction` abstract contract from `dss-exec-lib`. Alongside `DssExecLib`, which contains the high-level functions used to interact with the Maker Protocol, and `DssExec`, the actual spell deployer, this lib bootstraps a good part of the on-chain interactions required to achieve whatever collateral engineering demands coming from the community/CUs.

Taken from `dss-exec-lib` itself:

_The developer must override the actions() function and place all spell actions within. This is called by the execute() function in the pause, which is subject to an optional limiter for office hours._

Overall, an executive contract can only be executed once it has more votes than the previously-executed executive proposal. After a certain proposal has been voted on, governance actually gas a delay of 48 hours in place before any spell is cast. This grace period is in place for a final analysis before on-chain execution.

Polling is used for off-chain voting (Snapshot.org-like) on deciding how to aggregate multiple collateral onboarding proposals, as there's only one proposal up for voting at a time. This also optimizes the timeline for collateral engineering versus the risk factor of changing too many parameters of the protocol at once.

Different assets can have multiple maker vaults/collateral types (ilks) to deposit collateral with varying risk factors. That's why you'll observe cases like ETH-A/B/C.



### DssSpellAction

The `dss-exec-lib` README is pretty comprehensive and goes over every available function exposed, so refer to it whenever necessary to get more granular into specific requirements. The scope of this doc is to provide a more collateral-engineering-focused view of the spell crafting process and embed any notes that might be helpful.

### Constants

Due to the fact that the delegate proxy actually casts the spells in practice, all variables need to be defined as constants in the contract. Spells should not rely on or interact with the contract storage for this very reason.

#### Rates

* `uint256 ZERO_PCT_RATE` = Null multiplier (1).
* `uint256 {ADJUSTED}_PCT_RATE` = Per-second-adjusted stability fee for 1.5% cases. If multiplied by the number of seconds in a year (60_60_24\*365), should return the `uint256` multiplier for 1.5%. Naming convention follows 1.75% = `ONE_SEVEN_FIVE_PCT_RATE`. Common rates can be found in [rates.sol](https://github.com/ClioFinance/spells-goerli/blob/master/src/test/rates.sol).

If the rate is not already declared in the file above, it can be calculated using `bc`:

```bash
bc -l <<< 'scale=27; e( l(1.08) / (60 * 60 * 24 *365) )' # 8%
```

Which will result `1.000000002440418608258400030`.

Optionally the `rates.sol` file can be updated to include it:

```javascript
// The key is the pct in basis points 8% = 800 b.p.
rates[800] = 1000000002440418608258400030; // Notice we rejamove de dot!
```

#### Precision-related variables for mathematical operations

`uint256 MILLION` = 10 to the power of 6.

`uint256 WAD` = 10 to the power of 18.

`uint256 RAY` = 10 to the power of 27.

`uint256 RAD` = 10 to the power of 45 (`RAY * WAD`).

#### Contract addresses

These contracts below need to be deployed before the creation of the Spell on a per-case basis. Deployed factories to create them can be found on the [Maker Changelog](https://changelog.makerdao.com) after navigating to → chosen network → latest release → Contract addresses → _VAR\_KEY_

Contracts owner needs to be the _MCD\_PAUSE\_PROXY_ address proxy, which ends up being authorized as the highest-level admin to cast the spells.

* `address MCD_JOIN_{TOKEN-L}` = Contract where users will deposit their assets so the Maker Protocol can internally abstract away the actual tokens used as unlocked collateral (gem). This means that users are actually credited an internal balance once this Join adapter receives deposited tokens so the system is more standardized.

_VAR\_KEY = **JOIN\_FAB**_

You can refer to the [dss-gem-joins](https://github.com/makerdao/dss-gem-joins) repo for alternate gem join cases according to different token paradigms/functionalities (such as upgradeable contract cases and decimal places) for quick redeploys of similar assets. Be wary of the initial comment in each join for more detailed explanations.

* `address MCD_CLIP_{TOKEN-L}` = Collateral action house instance for this particular ilk. Can be deployed using a more constant factory (without many possibilities like join gems).

_VAR\_KEY = **CLIP\_FAB**_

Factory Parameters:

`owner` = _MCD\_PAUSE\_PROXY._

`vat` = MCD\_VAT, Maker's main lending engine.

`spotter` = MCD\_SPOT, Maker's oracle price feed interface.

`dog` = MCD\_DOG, Maker's liquidation contract.

`ilk` = Bytes32 representation of collateral case.

* `address MCD_CLIP_CURVE_{TOKEN-L}` = Collateral auctions follows a dutch auction format with a price curve set by this specific contract, feeding the clip contract itself. There are three different curve options - Exponential, Linear & Stairstep - one of which will be outlined in the governance proposal.

_VAR\_KEY = **CALC\_FAB**_

* `address {GEM}` = Token address of the underlying asset to be onboarded as collateral.
* `address PIP_{GEM}` = Oracle price feed address. Both gems most up-to-date parameters can be fetched from change log by leveraging the _DssExecLib.getChangelogAddress("GEM//PIP\_GEM")_ helper function.

### Functions

**officeHours()** = Public function the contract overrides from DssAction that returns a bool related to casting spells in a specific weekday timeframe.

**action()** = Executive function that changes the maker protocol. Should finish by first calling DssExecLib.setChangelogAddress("_VAR\_ADD_",_VAR\_ADD_) for each new contract deployed then calling the DssExecLib.setChangelogVersion("1.X.YY") function to increase versioning.

Can include multiple actions of different types (e.g. multiple collateral onboarding, multiple stability fees, or debt ceiling changes).

#### Common Actions

*   `setIlkStabilityFee` = Public function used to increase or decrease vault stability fee.

    Parameters:

    `bytes32 _ilk`

    `uint256 _rat`

    `bool _doDrip` = relates to the accumulation stability fees for the collateral. Usually is true.
*   `setIlkAutoLineDebtCeiling` = Public function used to increase or decrease vault debt ceiling.

    Parameters:

    `bytes32 _ilk`

    `uint256 _amount`
*   `setIlkAutoLineParameters` = More general version of `setIlkAutoLineDebtCeiling` that also includes the gradual gap increase parameters to gradually manage debt ceiling risk.

    Parameters:

    `bytes32 _ilk`

    `uint256 _amount`

    `uint256 _gap` = can be fetched by inputting the bytes32 of the ilk (that can be obtained using a tool such as [Web3 Type Converter](https://web3-type-converter.onbrn.com)) in the ilks method of the[ DssAutoLine contract](https://etherscan.io/address/0xC7Bdd1F2B16447dcf3dE045C4a039A60EC2f0ba3#readContrac).

    `uint256 _ttl` = amount in seconds of each gap interval
*   `setIlkMinVaultAmount` = Public function to increase or decrease the dust parameter. Parameters:

    `bytes32 _ilk`

    `uint256 _amount`

#### Collateral Onboarding

Adding new collateral types to the Maker Protocol is first of all guided by an on-chain governance proposal such as [this one](https://vote.makerdao.com/polling/QmdVYMRo?network=mainnet). There, we can find many of the necessary parameters for the spell engineering process, leveraging some DssExecLib helpers to kickstart the process. The main one is:

* `addNewCollateral` = Public function to add a new collateral type to the Maker Protocol. Takes in a struct param in the form of ColleteralOpts.

### CollateralOpts

`bytes32 ilk` = Bytes32 result of the hyphenated, all caps token vault name (e.g. WBTC-C). Can be passed down directly as a string.

`address gem` = Address of the gem of the particular underlying token ({GEM} variable), found already deployed in the Maker Changelog with {_TOKEN\_TICKER_} (e.g. _WBTC_) as its key.

`address join` = Address of join gem (MCD\_JOIN\_{TOKEN-L}).

`address clip` = Address of auction clip (MCD\_CLIP\_{TOKEN-L}).

`address calc` = Address of auction price discovery curve (MCD\_CLIP\_CALC\_{TOKEN-L}). Type of contract to be deployed is decided by the [_**Auction Price Function (****`calc`****)**_](https://community-development.makerdao.com/en/learn/governance/param-auction-price-function) gov proposal parameter.

`address pip` = Address of the price feed oracle to be used **(PIP\_{GEM}**).

`bool isLiquidatable` = Usually true, the collateral can indeed be liquidated.

`bool isOSM` = True if there's an OSM contract being used for the price feed.

`bool whitelistOSM` = True if there's the need to whitelist an OSM contract being onboarded to the Maker Protocol.

`uint256 ilkDebtCeiling` = [_**Debt Ceiling (****`line`****)**_](https://community-development.makerdao.com/en/learn/governance/param-debt-ceiling) gov proposal parameter. Set in DAI. Debt ceiling is the maximum amount of DAI able to be minted from a given vault collectively.

`uint256 minVaultAmount` = [_**Debt Floor (****`dust`****)**_](https://community-development.makerdao.com/en/learn/governance/param-debt-floor) gov proposal parameter. Set in DAI. Dust parameter is the minimum amount of DAI possible to generate in a given vault.

`uint256 maxLiquidationAmount` = [_**Local Liquidation Limit (****`ilk.hole`****)**_](https://community-development.makerdao.com/en/learn/governance/param-local-liquidation-limit) gov proposal parameter. Set in DAI.

`uint256 liquidationPenalty` = Basis point percentage of penalty fee (e.g. 13% = 1300). Usually 1300 for most cases.

`uint256 ilkStabilityFee` = Per-second-rate obtained from the [_**Stability Fee**_](https://community-development.makerdao.com/en/learn/governance/param-stability-fee) gov proposal parameter ({_ADJUSTED_}\_PCT\_RATE).

`uint256 startingPriceFactor` = [_**Auction Price Multiplier (****`buf`****)**_](https://community-development.makerdao.com/en/learn/governance/param-auction-price-multiplier) gov proposal parameter. Set in basis points (1.2 = 12000). Starting price multiplier is how much higher than the oracle price auction starts at.

`uint256 breakerTolerance` = [_**Breaker Price Tolerance (****`tolerance`****)**_](https://community-development.makerdao.com/en/learn/governance/param-breaker-price-tolerance) gov proposal parameter. Set in basis points (0.5 = 5000). How large of a price drop is tolerated before liquidations are paused.

`uint256 auctionDuration` = [_**Maximum Auction Duration (****`tail`****)**_](https://community-development.makerdao.com/en/learn/governance/param-max-auction-duration) gov proposal parameter. Given in minutes (Solidity time unit).

`uint256 permittedDrop` = [_**Maximum Auction Drawdown (****`cusp`****)**_](https://community-development.makerdao.com/en/learn/governance/param-max-auction-drawdown) gov proposal parameter. Set in basis points (0.4 = 4000). Maximum percentage drop in collateral price during a collateral auction before the auction is reset.

`uint256 liquidationRatio` = [_**Liquidation Ratio**_](https://community-development.makerdao.com/en/learn/governance/param-liquidation-ratio) gov proposal parameter. Set in basis points (175% = 17500). Maximum amount of DAI debt that a vault user can draw from their vault given the value of their collateral locked in that vault.

`uint256 kprFlatReward` = [_**Flat Kick Incentive (****`tip`****)**_](https://community-development.makerdao.com/en/learn/governance/param-flat-kick-incentive) gov proposal parameter. Set in DAI. Flat DAI reward a keeper receives for triggering liquidations (to compensate for gas costs).

`uint256 kprPctReward` = [\*\*\*Proportional Kick Incentive (`chip`)](https://community-development.makerdao.com/en/learn/governance/param-proportional-kick-incentive)\*\*\* gov proposal parameter. Set in basis points (0.1% = 10). Percentual reward keeper receives from liquidations.

To finish onboarding a new collateral, a couple of additional configs are needed:

* DssExecLib.setStarstepExponentialDecrease(CLIP\_CALC, [_**Price Change Interval (****`step`****)**_](https://community-development.makerdao.com/en/learn/governance/param-auction-price-function) in seconds, [_**Price Change Multiplier (****`cut`****)**_](https://community-development.makerdao.com/en/learn/governance/param-auction-price-function) in basis points).
* DssExecLib.setIlkAutoLineParameter(ILK\_STR, ilkDebtCeiling\*\*, [_Target Available Debt (`gap`)_](https://makerdao.world/en/learn/governance/module-dciam)_,_ [_Ceiling Increase Cooldown (`ttl`)_](https://makerdao.world/en/learn/governance/module-dciam)\*\* in Solidity time units).

__
