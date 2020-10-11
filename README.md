# PlotX smart contracts audit by Mudit Gupta

## Table of contents

- [Objective of the audit](#objective-of-the-audit)
- [Audit specifics](#audit-specifics)
- [Issues discovered](#issues-discovered)
  - [Critical](#critical)
    - [1.1 PlotX tokens that are given out as governance rewards can be used to steal funds from the MarketRegistry](#11-plotx-tokens-that-are-given-out-as-governance-rewards-can-be-used-to-steal-funds-from-the-marketregistry)
    - [1.2 Incorrect `ethAmountToPool` and `tokenAmountToPool` calculations can prevent markets from being closed](#12-incorrect-ethamounttopool-and-tokenamounttopool-calculations-can-prevent-markets-from-being-closed)
  - [Major](#major)
    - [2.1 Staking interest is generated even when there are no tokens at stake](#21-staking-interest-is-generated-even-when-there-are-no-tokens-at-stake)
    - [2.2 Unprotected function in bLOT token](#22-unprotected-function-in-blot-token)
    - [2.3 Advisory board members can unilaterally reject proposals that try to remove them from the Advisory board](#23-advisory-board-members-can-unilaterally-reject-proposals-that-try-to-remove-them-from-the-advisory-board)
  - [Minor](#minor)
    - [3.1 Vesting logic has unnecessary complexity](#31-vesting-logic-has-unnecessary-complexity)
    - [3.2 Unprotected function in MarketRegistry can lead to funds being drained](#32-unprotected-function-in-marketregistry-can-lead-to-funds-being-drained)
    - [3.3 Invalid data check in the vesting contract](#33-invalid-data-check-in-the-vesting-contract)
    - [3.4 Problems when changing contract addresses](#34-problems-when-changing-contract-addresses)
    - [3.5 Extra tokens can be locked during governance than required](#35-extra-tokens-can-be-locked-during-governance-than-required)
    - [3.6 Enums are not used always](#36-enums-are-not-used-always)
    - [3.7 The governance contract ignores some category parameters](#37-the-governance-contract-ignores-some-category-parameters)
    - [3.8 Some functions can run out of gas](#38-some-functions-can-run-out-of-gas)
    - [3.9 Minor rounding issues in the Staking contract](#39-minor-rounding-issues-in-the-staking-contract)
    - [3.10 Option prices have no buffer between them](#310-option-prices-have-no-buffer-between-them)
  - [Informational](#informational)
    - [4.1 There are extra variables and functions defined that are not needed](#41-there-are-extra-variables-and-functions-defined-that-are-not-needed)
    - [4.2 Naming of variables, functions and events can be improved](#42-naming-of-variables-functions-and-events-can-be-improved)
    - [4.3 Additional checks can be added to prevent user errors](#43-additional-checks-can-be-added-to-prevent-user-errors)
    - [4.4 Contracts can be optimized to save gas](#44-contracts-can-be-optimized-to-save-gas)
    - [4.5 Documentation can be improved](#45-documentation-can-be-improved)
    - [4.6 External dependency contracts should be referenced directly](#46-external-dependency-contracts-should-be-referenced-directly)
    - [4.7 Block numbers are used instead of time to calculate interest](#47-block-numbers-are-used-instead-of-time-to-calculate-interest)
    - [4.8 Non standard functionality](#48-non-standard-functionality)
- [Additional risks](#additional-risks)
  - [Oracle risks](#oracle-risks)
  - [Governance risks](#governance-risks)
  - [Economic/Tokenomic risks](#economictokenomic-risks)
- [Test coverage](#test-coverage)
  - [Status Update](#status-update-10)
- [Static analysis](#static-analysis)
- [Disclaimer](#disclaimer)

## Objective of the audit

The audit's focus was to verify that the smart contract system is secure, resilient, and working according to its specifications. The review activities can be grouped in the following three categories:

**Security**: Identifying security related issues within each contract and the system of contracts.

**Sound Architecture**: Evaluation of this system's architecture through the lens of established smart contract best practices and general software best practices.

**Code Correctness and Quality**: A full review of the contract source code.

## Audit specifics

This audit is based on commit hash `57edca4ca8d5cbfcd266de2bf36a65c226c05603` of <https://github.com/plotx/smart-contracts/>

The contracts included in this audit are

- bLOTToken.sol
- Governance.sol
- Market.sol
- MarketRegistry.sol
- MarketUtility.sol
- Master.sol
- MemberRoles.sol
- PlotXToken.sol
- ProposalCategory.sol
- Staking.sol
- TokenController.sol
- Vesting.sol

Additionally, Airdrop.sol was audited from commit hash `91b690c2c71b2c356a451fcbd428e6503ed7f04c`.

The contracts were reviewed over a period of 4 days. This report was compiled after the review.

## Issues discovered

During the review, 2 Critical, 3 Major, 10 Minor, and 8 Informational issues were found.

- A critical issue represents something that can be relatively easily exploited and will likely lead to loss of funds.
- A major issue represents something that can result in an unintended behavior of the smart contracts. These issues can also lead to loss of funds but are typically harder to execute than critical issues.
- A minor issue represents an oddity discovered during the review. These issues are typically situational or hard to exploit.
- An informational issue represents a potential improvement. These issues do not pose any practical security risks.

### Critical

##### 1.1 PlotX tokens that are given out as governance rewards can be used to steal funds from the MarketRegistry

##### Description

The `_closeVote` function of Governance.sol transfers PlotX tokens for incentive distribution from `marketRegistry` to itself. However, there are no guarantees that the `marketRegistry` will have the required funds. The market registry does not reserve any special funds for governance incentives. It will give out the governance incentive from users' market creation rewards if it doesn't have enough extra funds. The `marketRegistry` should only be transferring governance incentives from the leftover commission or from ecosystem reserves, not from market creation rewards. There are multiple ways this can be exploited

- The incentive/reward that is given for voting on a proposal is taken from MarketRegistry. The MarketRegistry can be drained by creating a bunch of bogus proposals and then voting on them. The rewards depend on the category settings, so depending on how the categories are configured, there might be no incentive for the users to create/vote on proposals.
- Even for legitimate proposals, Market registry has no way to ensure it has PlotX tokens for these rewards. If there's not enough user funds in the market registry, the governance proposals will fail to close.

##### Status update

- All token holder proposals except for changing Advisory board members require approval from advisory board members. Additionally, there are no rewards for proposals that change advisory board members. Therefore, this change prevents token holders from creating bogus proposals for rewards.
- Rewards are now taken from the Market Registry at the start of the proposal. Therefore, if there are not enough funds in the market registry, the proposal creation/whitelisting process will fail.

##### 1.2 Incorrect `ethAmountToPool` and `tokenAmountToPool` calculations can prevent markets from being closed

##### Description

- Market's `ethAmountToPool` and `tokenAmountToPool` do not include the commission sent to the market registry. When the market registry resolves any disputes, it does not send back the commission. The market contract tries to send the commission again to the registry in the `_postResult` function, but it will fail since it won't have enough funds left. This prevents markets from being closed unless someone sends them the extra commission.

- Market's `ethAmountToPool` and `tokenAmountToPool` are not reset after a dispute is created, and in the next calculation, the market contract adds the new calculations to these numbers. Therefore, `ethAmountToPool` and `tokenAmountToPool` become more than they should be. This will prevent future disputes from being created since the contract won't have enough funds to send to the market registry due to the buggy calculation of `ethAmountToPool` and `tokenAmountToPool`. Using this exploit, someone can create bogus disputes to prevent real disputes from being created.

##### Status update

This issue has been fixed.

### Major

##### 2.1 Staking interest is generated even when there are no tokens at stake

##### Description

Global interest is accrued from the start time, regardless if anyone has placed any stake or not. If there's no stake, this interest becomes locked and unclaimable by anyone. The interest should start accruing from the time of the first stake and stop accruing if everyone has withdrawn their stake. In other words, interest should be generated only if there are tokens at stake.

##### Status update

The system now transfers the interest to a vault address if there is no stake in the system.

##### 2.2 Unprotected function in bLOT token

##### Description

The `initiatebLOT` function of the bLOT token is unprotected and can be called by anyone at any time. Using this function, anyone can add new minters.

##### Status update

This issue has been fixed.

##### 2.3 Advisory board members can unilaterally reject proposals that try to remove them from the Advisory board

##### Description

The Advisory board members can call `rejectAction` function to reject any proposal, including those that try to remove them from the Advisory board. This will prevent the community from removing a malicious/hacked Advisory board member from the Advisory board.

##### Status update

The advisory board members can no longer unilaterally reject proposals that are for swapping advisory board members.

### Minor

#### 3.1 Vesting logic has unnecessary complexity

##### Description

The vesting logic has support for vesting start date and the cliff date, but the way cliff dates are used, they result in identical behavior to using the start date. Since there are no business requirements for vesting cliffs, it's recommended to eliminate cliff dates and simply use the vesting start date.

##### Status update

This issue has been fixed.

#### 3.2 Unprotected function in MarketRegistry can lead to funds being drained

##### Description

In MarketRegistry, anyone can create new a market using `addInitialMarketTypesAndStart` and then claim the `claimCreationReward`. The attacker can also provide malicious market implementation to steal other funds as well. However, this function can only be called when there is no pre-existing market. Therefore, if anyone tries to exploit this, it will be detected before the contract gets hold of any funds. In any case, it is suggested to add a check to only the owner/governance to call this function.

##### Status update

This issue has been fixed.

#### 3.3 Invalid data check in the vesting contract

##### Description

`noGrantExistsForUser` checks that start time != 0, but this is not a valid check since startTime can be zero in the current implementation. This will lead to the users' vesting schedule getting overwritten and them losing the tokens they were going to get from the first vesting schedule. Please add a check in the `addTokenVesting` fn to prevent 0 start time.

##### Status update

This issue has been fixed.

#### 3.4 Problems when changing contract addresses

##### Description

Some functions facilitate change of address for certain contracts, but they do not do their jobs properly. If the intention is to never change contract addresses, these functions should be removed. Otherwise, these functions should be fixed.

- The `changeOperator` function of token controller changes the operator of PlotX token but not the bLOT token.
- Governance and Token controller contracts have `constructorCheck` in their `setMasterAddress` function that will prevent the master address from changing.

#### 3.5 Extra tokens can be locked during governance than required

##### Description

If a user votes on a governance matter, all of their PlotX tokens are locked for `tokenHoldingTime` (7 days by default but can be changed via governance). The lock also applies to any future tokens they may receive that are not counted in their voting power. Ideally, only the tokens used in voting should be locked.

#### Response from PlotX Team

This is done on purpose to simplify the implementation as PLOT is a freely transferable token. In case, user wishes to use less tokens, they can transfer as many tokens in a separate account and vote.

#### 3.6 Enums are not used always

##### Description

MemberRole.Role enum is used in some places but not in others. For example, it's used in the member roles contract not used in proposal categories and governance contracts. This makes it harder to keep track of things.

#### 3.7 The governance contract ignores some category parameters

##### Description

There are some default categories where token holders are assigned the voter and proposal creator role. However, the governance contract only allows proposals to be created where token holders can vote if the proposal is for changing Advisory board members.

##### Status update

The default categories have been fixed, but the protocol still allows adding such conflicting categories in the future.

#### 3.8 Some functions can run out of gas

##### Description

In `claimReward` function of the governance contract and `claimPendingReturn` of the market registry contract, `i` should be limited to `maxRecords`. Adding a new variable `j` that is only incremented in certain cases does not resolve the problem of a function running out of gas in certain edge cases.

#### 3.9 Minor rounding issues in the Staking contract

##### Description

There can be minor roundoff errors in staking contract in stakeBuyInRate that will prevent the last staker from withdrawing their stake unless they send some insignificant amount of reward tokens to the staking contract to cover the rounding errors. It is recommended to make changes like adding one to the intermediate value before division so that the rounding errors favor the smart contract.

##### Status update

The PlotX team will send an extra token to the staking contract to cover the rounding issues.

#### 3.10 Option prices have no buffer between them

##### Description

On PlotX, 9,999.99 can be a different option than 10,000. In reality, there's no single/exact price of BTC at any given moment. It doesn't matter what oracle is used. This will lead to disputes that will not have a mathematically correct resolution. BTC can trade at different prices at different exchanges. The price is similar thanks to arbitrage, but having a ~0.2% variance is normal among the largest exchanges.

There's no feature to dissolve the market in case of such issues, so the losing party may feel cheated.

#### Response from PlotX Team

Chainlink uses the pricefeed from 7+ exchanges to determine the price. Further, there is no limit of adding deviation and hence the options are mutually exclusive. In case of a scenario where user disagrees with the oracle, they can raise a dispute in the Cooling window of market.

### Informational

#### 4.1 There are extra variables and functions defined that are not needed

##### Description

- bLOT token does not support `transferFrom` and related functions, but `_allowances` is still defined. It is not used anywhere.
- `upFront` in the vesting contract does not need to be stored in storage.
- `claimed` boolean of `LockToken` is not checked in all the functions. It adds no value to the system overall. Just checking if amount > 0 should be sufficient.
- `governance` address is stored in MemmberRoles.sol but it's never used.
- Since `updateProposal`, `addSolution`, `openProposalForVoting`, and some other functions are not used, they should be removed, and the interface should be updated.
- `dAppLocker` in master.sol is set as TC but not used and not updated when TC is changed.
- `OnlyOwner` functionality can be inherited from OpenZeppelin's Ownable contract.

#### 4.2 Naming of variables, functions and events can be improved

##### Description

- Event names are not consistent. For example, Staked, StakeWithdraw, InterestCollected. StakeWithdraw should be renamed to StakeWithdrawn.
- Naming of variables is inconsistent. `_` is prefixed to some variables in random fashion, adding to confusion.
- `burnCommissionTokens` of TC is never used.

#### 4.3 Additional checks can be added to prevent user errors

##### Description

More input data checks can be added to prevent users from shooting themselves in the foot. For example, `require(_vestingDuration > _vestingCliff)` can be added in the `addTokenVesting` function.

#### 4.4 Contracts can be optimized to save gas

##### Description

- A lot of fields in `Allocation` struct can be converted to smaller `uints` to save space and gas cost.
- The condition `require(!token.isLockedForGV(msg.sender),"Locked for GV vote");` can be removed. Even if the user withdraws the vested tokens, they can't transfer them away.
- `ProposalData` and `ProposalVote` are sub-optimal. They have a lot of fields defined as `uint256` that do not need to be `uint256`.
- External contract reads are done in sub-optimal fashion. At a lot of places, getters are used that return multiple values, but only some of those values are used, and rest are discarded. For example, the governance contract has `(, uint256 roleAuthorizedToVote, , , , , ) = proposalCategory.category(_categoryId)`. Since reading values consumes gas, only the required values should be read.
- Since only one Member Role can vote on any given proposal, `abVoteValue` can be skipped from being maintained separately in case of Advisory board voting.
- `calculateInterest` does not update the `GlobalYieldPerToken`. This is not a security issue since the code updates the variable in outer calls, but this means that any UI that uses the `calculateInterest` function to show pending interest, it will show outdated data. If this is meant to be used internally only, it should be marked internal to save some gas.
- It's not required to initialize variables with the default value. For example, `uint totalAmount=0;` can be converted to `uint totalAmount;` to save some gas.
- There's no need to use SafeNath in `remainingbudget = remainingbudget.sub(totalAmount);` in Airdrop.sol since the condition is checked earlier anyway. To further optimize this, amounts can be directly subtracted from a cached copy of `remainingbudget` instead of adding them to `totalAmount` and then subtracting the sum.
- In `Airdrop.sol`, Instead of setting `userClaimed` to true, `userAllocated` can be set to 0 to save gas.

#### 4.5 Documentation can be improved

##### Description

- The codebase has complex economics that can be documented better.
- Natspec comment for `calculateVestingClaim` still refers to "Months" instead of periods.
- Code comments are outdated and invalid throughout the codebase, making it harder to detect the expected behavior. For example, `.convertToPLOT`'s comments say that it is just a burn function.
- Acronyms and abbreviations like SMLP", "DR", "BRLIM", etc. are used throughout the codebase without any documentation. This makes it harder to follow the logic and forces the reader to make some assumptions that may or may not be correct. It is recommended to use full names everywhere. Enums should be used for "code" fields.

#### 4.6 External dependency contracts should be referenced directly

##### Description

External dependency contracts are copy-pasted in "external" folder and out of the scope of this audit. It is recommended to use official npm packages for dependency contracts instead of copy-pasting them.

#### 4.7 Block numbers are used instead of time to calculate interest

##### Description

The block time is not an invariant in Ethereum and might change with a future update. This can lead to fewer/more rewards being paid out than intended. Furthermore, the code uses block numbers to calculate staking interest etc but calls it "time". If the interest is supposed to be calculated based on the time elapsed, then the logic/code is wrong. If the interest is supposed to be calculated based on block number, then the comments and variable naming is wrong. It is recommended to switch to using time for calculating interest.

#### 4.8 Non standard functionality

##### Description

In `increaseLockAmount` function of TokenController, the lock duration is also increased in the case of "SM" locks. I understand why this is being done, but this is against the standard. It also makes the function name a bit misleading. It will be better to call both extend and increase functions individually or have a different wrapper function - IncreaseAndExtendLock.

**NOTE**: Most of the suggestions from the informational issues have been implemented already.

## Additional risks

### Oracle risks

- Uniswap is used to calculate the price of PlotX token in terms of ETH. The price on Uniswap depends significantly on the liquidity of PlotX on Uniswap. Hourly cumulative price is used to reduce the risk of price manipulation but if PlotX isn't traded much on Uniswap or has low liquidity, it will be possible to manipulate the price using flash loans.
- Chainlink is used for BTC/USDT price feed. Chainlink is one of the most reliable oracles, but they've had issues in the past as well. All oracle risks due to things like chain congestion apply to this project as well.

### Governance risks

The governance contract has direct or indirect control over most of the protocol contracts. In the initial governance strategy, the Advisory board is permissioned to take the governance decisions.

### Economic/Tokenomic risks

The economics were not in scope of this audit. However, Some risks are obvious. For example, the risk/reward ratio of predictions can vary a lot based on factors that are not in control of the users.

## Test coverage

The project has Javascript test cases, and from a brief look at them, it was noticed that the staking test cases were not moving blocks forward as intended, and hence they were not verifying the correct results.

The branch coverage of the project was around 85%. It is suggested to aim for 100% branch coverage for the smart contracts. The coverage report is as follows:

File                              |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
----------------------------------|----------|----------|----------|----------|----------------|
 contracts/                       |    94.25 |    84.92 |       92 |    93.98 |                |
  Airdrop.sol                     |      100 |      100 |      100 |      100 |                |
  Governance.sol                  |    94.25 |    83.54 |    93.33 |    93.95 |... 24,946,1093 |
  Market.sol                      |    96.94 |    83.72 |    93.33 |    96.41 |... 285,286,333 |
  MarketRegistry.sol              |     90.7 |    72.06 |    85.71 |     90.8 |... 463,465,494 |
  MarketUtility.sol               |    82.64 |    77.59 |    73.91 |    80.43 |... 349,353,373 |
  Master.sol                      |      100 |      100 |      100 |      100 |                |
  MemberRoles.sol                 |      100 |    89.47 |      100 |      100 |                |
  PlotXToken.sol                  |      100 |       80 |      100 |      100 |                |
  ProposalCategory.sol            |      100 |    93.33 |      100 |      100 |                |
  Staking.sol                     |      100 |      100 |      100 |      100 |                |
  TokenController.sol             |    90.72 |    83.33 |    86.36 |    90.82 |... 303,324,325 |
  Vesting.sol                     |      100 |      100 |      100 |      100 |                |
  bLOTToken.sol                   |    92.68 |    68.18 |    94.44 |    93.02 |       85,86,87 |
 contracts/marketImplementations/ |      100 |      100 |      100 |      100 |                |
  MarketBTC.sol                   |      100 |      100 |      100 |      100 |                |
All files                         |    94.25 |    84.92 |       92 |    93.98 |                |

### Status Update

The test coverage has been improved to around 97% branch coverage.

## Static analysis

Slither was used to do static analysis of the contracts and the output of slither can be found at https://gist.github.com/maxsam4/906a12d5fb3611ea75016be6593cfdde

**NOTE**: Please keep in mind that most of the issues flagged by Slither are false positives.

## Disclaimer

This report is not an endorsement or indictment of any particular project or team, and the report do not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. This report does not provide any warranty or representation to any Third-Party in any respect, including regarding the bugfree nature of code, the business model or proprietors of any such business model, and the legal compliance of any such business. No third party should rely on this report in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project. I owe no duty to any Third-Party by virtue of publishing these Reports.

The scope of my review is limited to a review of Solidity code and only the Solidity code noted as being within the scope of the review within this report. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty.

This audit does not give any warranties on finding all possible security issues of the given smart contracts, i.e., the evaluation result does not guarantee the nonexistence of any further findings of security issues. As one audit-based assessment cannot be considered comprehensive, I always recommend proceeding with several independent audits and a public bug bounty program to ensure the security of smart contracts.
