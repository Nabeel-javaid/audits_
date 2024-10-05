# MProfy-Dao-Web3 Security Review

A security review of the MProfy-Dao-Web3 smart contract protocol was done by [0xepley](https://twitter.com/0xepley). \
This audit report includes all the vulnerabilities, issues and code improvements found during the security review.

# About Legacy

Legacy is a decentralized smart contract protocol on Ethereum designed to be a digital store of value with Bitcoin-like tokenomics, offering a finite supply, self-minting rewards mechanism, and the potential to outperform Bitcoin, all while providing an inflation-resistant asset that empowers users to preserve and grow their wealth in a secure, decentralized, and energy-efficient manner.

## Disclaimer

"Audits are a time, resource and expertise bound effort where trained experts evaluate smart
contracts using a combination of automated and manual techniques to find as many vulnerabilities
as possible. Audits can show the presence of vulnerabilities **but not their absence**."

\- Secureum

## Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
| :----------------- | :----------: | :------------: | :---------: |
| Likelihood: High   |   Critical   |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

### Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- **Low** - can lead to any kind of unexpected behaviour with some of the protocol's functionalities that's not so critical.

### Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no incentive.

### Actions required by severity level

- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

## Executive summary

### Overview

|               |                                                                                              |
| :------------ | :------------------------------------------------------------------------------------------- |
| Project Name  | MProfy-Dao-Web3                                                                                       |
| Repository    | https://github.com/DhruvGodambe/mprofy-dao-web3  
| Methods       | Static + Manual review                                                                                |
|               |


### Issues found

| Severity      |                                                     Count |
| :------------ | --------------------------------------------------------: |
| High risk     |       0 |
| Medium risk   |     1 |
| Low risk      |       N/A |
| Informational | N/A |

### Scope

| File                                                                                                    | 
| :------------------------------------------------------------------------------------------------------ | 
| _Contracts (8)_                                                  |
| /Deed.sol |
| /FounderNFT.sol |
| /Governance.sol |
| /PriceConvertor.sol |
| /Treasury.sol |

# Findings

## High severity

### [H-01] Excessive Proposal Fee sent by user is lost

#### Description:
In the `createProposal()` function, the requirement is that users send a minimum fee of `CREATE_PROPOSAL_COST` to create a proposal. However, if a user sends more than the required fee there is no advantage of it and, the excess amount is not refunded. This could lead to user may unintentionally overpay without being refunded.

```solidity
require(
    msg.value >= CREATE_PROPOSAL_COST,
    "Amount too low for creating proposal"
);
```

There is no logic to handle cases where the user sends more than `CREATE_PROPOSAL_COST`. As a result, any excess ETH sent is effectively "lost" to the contract.

#### Recommendation:
Introduce a logic to refund the excess ETH amount back to the user if they send more than the required proposal creation fee.


```solidity
require(
    msg.value >= CREATE_PROPOSAL_COST,
    "Amount too low for creating proposal"
);

// If user sends more than the required fee, refund the excess
uint excessAmount = msg.value - CREATE_PROPOSAL_COST;
if (excessAmount > 0) {
    payable(msg.sender).call{value: excessAmount}(""); // Refund excess

}
```

This way, users will only be charged the exact amount needed to create the proposal, and any extra ETH sent will be safely refunded to their wallet.


### [H-02] Unrestricted Access to Change Swap Router Address


#### Description:
The `setSwapRouter` function allows **anyone** to change the swap router address (`SWAP_ROUTER_02`) without any restrictions. This is a critical issue, as a malicious actor could change the router to their own contract, enabling them to steal all user's funds.


```solidity
function setSwapRouter(address _router) public { 
    SWAP_ROUTER_02 = _router; // Unrestricted access to modify critical contract address
}
```

#### Recommendations:

It is recommended to restrict this function explicitly to the owner that only authorized person can change the router address.

```solidity
   modifier onlyOwner() {
       require(msg.sender == owner, "Caller is not the owner");
       _;
   }

   function setSwapRouter(address _router) public onlyOwner {
       SWAP_ROUTER_02 = _router;
   }
 ```




### [H-02] Treasury can not received the Locked NFT

#### Description:
In the `lockDeedNFT()` function, The NFT is being sent to `address(this)` which is treasury contract. The Transfer is being made using `transferFrom` 

```solidity
  IERC721(deedNFT).transferFrom(msg.sender, address(this), tokenId); 
```
but since the Treasury Contract do not have `OnERC721Received` function it will not received the NFT and the NFT will be lost in limbo.

According to [openzeppelin docs](https://docs.openzeppelin.com/contracts/2.x/api/token/erc721#ERC721-transferFrom-address-address-uint256-):

`transferFrom()`

`Transfers the ownership of a given token ID to another address. Usage of this method is discouraged, use safeTransferFrom whenever possible. Requires the msg.sender to be the owner, approved, or operator.`

As per the documentation of `EIP-721`:

`A wallet/broker/auction application MUST implement the wallet interface if it will accept safe transfers.`

Ref: https://eips.ethereum.org/EIPS/eip-721



#### Recommendation:
Call the safeTransferFrom() method instead of transferFrom() for NFT transfers. 
```solidity
        IERC721(deedNFT).safeTransferFrom(msg.sender, address(this), tokenId); 
```



## Medium severity

### [M-01] Excessive Centralization Risk by Owner

#### Description:
The function `drainTreasury()` allows the contract owner to drain all ERC20 tokens and NFTs from the treasury to a company wallet. This introduces significant centralization risks as the owner has unrestricted control over the assets. If the private key of the owner or admin is compromised, or if there is any malicious intent, the entire treasury can be drained. This creates a single point of failure and is devastating for the platform's ecosystem.

```solidity
    function drainTreasury() external onlyOwner {
        // Drain all ERC20 tokens to the admin wallet
        for (uint256 i = 0; i < assets.length; i++) {
            IERC20 token = IERC20(assets[i]);
            uint256 balance = token.balanceOf(address(this));
            if (balance > 0) {
                token.transfer(companyWallet, balance); // Risky centralization
            }
        }
        // Transfer all NFTs
        _transferAllNFTs(deedNFT);
        _transferAllNFTs(founderNFT);
        _transferAllNFTs(genesisNFT);
    }
```


#### **Recommended Mitigation Steps**
It is recommended to use Multi-Sig wallet


### [M-02] Chainlink's `latestRoundData` might return stale or incorrect results

#### **Description**
Chainlink's latestRoundData is used here to retrieve price feed data; however, there is insufficient protection against price staleness.

Return arguments other than `int256 answer` are necessary to determine the validity of the returned price, as it is possible for an outdated price to be received. [See here](https://ethereum.stackexchange.com/questions/133242/how-future-resilient-is-a-chainlink-price-feed/133843#133843) for reasons why a price feed might stop updating.

The return value `updatedAt` contains the timestamp at which the received price was last updated, and can be used to ensure that the price is not outdated. See more information about `latestRoundID` in the [Chainlink docs](https://docs.chain.link/data-feeds/api-reference#latestrounddata). Inaccurate price data can lead to functions not working as expected and/or loss of funds.

```solidity
function getPrice(AggregatorV3Interface priceFeed) public view returns (uint256) {
        (, int256 answer,,,) = priceFeed.latestRoundData(); //@audit 
        // Chainlink returns price with 8 decimal places
        return uint256(answer);
    }

```

Chainlink [recommends](https://docs.chain.link/data-feeds/api-reference#latestrounddata) using their data feeds along with some controls to prevent mismatches with the retrieved data
```solidity
function latestRoundData() external view
    returns (
        uint80 roundId,             //  The round ID.
        int256 answer,              //  The price.
        uint256 startedAt,          //  Timestamp of when the round started.
        uint256 updatedAt,          //  Timestamp of when the round was updated.
        uint80 answeredInRound      //  The round ID of the round in which the answer was computed.
    )

```
only the `answer` is used in the `getPrice()` implementation. The retrieved price of the priceFeed can be outdated and used anyways as a valid data because no timestamp tolerance of the update source time is checked while using the return parameters of feed.latestRoundData().


#### **Recommended Mitigation Steps**
It is recommended both to add also a tolerance that compares the updatedAt return timestamp from latestRoundData() with the current block timestamp and ensure that the priceFeed is being updated with the required frequency.



### [M-03] Missing deadline checks allow pending transactions to be maliciously executed

#### Description:
AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example see [Uniswap V2](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L229) and [Uniswap V3](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L119)). If such an option is not present, users can unknowingly perform bad trades:

1): Alice wants to swap `100 tokens` for `1 ETH` and later sell the `1 ETH` for `1000 DAI`.

2): The transaction is submitted to the mempool, however, Alice chose a transaction fee that is `too` low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.

3): When the average gas fee dropped far enough for Alice's transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of ETH could have drastically changed. She will still get 1 ETH but the DAI value of that output might be significantly lower. She has unknowingly performed a bad trade due to the pending transaction she forgot about.


### Recommended Mitigation Steps

Introduce a deadline parameter to all functions which potentially perform a swap on the user's behalf.



### [M-04] Missing deadline checks allow pending transactions to be maliciously executed

#### Description:

```solidity
IV3SwapRouter.ExactInputSingleParams memory params = IV3SwapRouter
            .ExactInputSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                fee: feeTier,
                recipient: address(this), 
                amountIn: amountIn,
                amountOutMinimum: 0, 
                sqrtPriceLimitX96: 0
            });
```
The `swapExactInputSingle()` function in `treasury.sol` is performing the swap from uniswap pool. In this function `0` value is assigned to `amountOutMinimum` argument which is used for slippage tolerance. 0 value here essentially means 100% slippage tolerance. This is a very easy target for MEV and bots to do a flash loan sandwich attack. 

These occur when MEV algorithms purchase significant amounts of the asset involved in the swap. This action drives the price of the asset up before the swap occurs. These algorithms then sell the asset at this higher price, making a profit. As a result, the user conducting the swap might end up paying more for the asset than they initially expected

### Recommended Mitigation Steps

It is recommended to consider slippage when calculating the minimum amount of tokens that should receive.


## Low severity

### [L-01] Missing Events for Critical Functions

#### Description:
The contract lacks event emissions for critical functions, such as minting tokens and burning tokens during transfers. Events are crucial for providing transparency and facilitating off-chain monitoring of contract activities. Without events, it becomes challenging to track important state changes, like minting and burning, which could lead to a lack of accountability and hinder debugging efforts.
  
#### Recommendation:
Emit events in critical functions to log important actions:

###  [L-02] Misleading Function Name


#### Description:
The function `getLockedNFTBalances()` returns the count of locked NFTs of different types (Deed, Founder, Genesis) for a user. However, the function name suggests that it returns "balances," which typically refers to token quantities or amounts rather than counts of NFTs. 

```solidity
function getLockedNFTBalances()
```

The function is actually returning the count of locked NFTs, not balances in the traditional sense of tokens.

#### Recommendation:
Rename the function to `getLockedNFTCount()` 
