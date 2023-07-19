Proud Sepia Rattlesnake

high

# Users can redeem setTokens in the middle of a bid to game the system

## Summary
If an auction is not locked users can redeem setTokens in the middle of a bid to game the system. 

## Vulnerability Detail
Some helpful details:
1. `nonReentrant` **does not** protect cross-contract re-entrancy.
2. Index operates with any token/coin, which means either now or in the future it's bound to use ERC777 or any other standart that has beforeTransfer and afterTransfer hooks.
     
Any user who owns any amount of a setToken can use the [bid](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L309-L348) mechanism in [AuctionRebalanceModuleV1](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol) to game the system, by bidding and then reentering to call [redeem](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L129-L163) in [BasicIssuanceModule](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol).This of-sets the math because [redeem](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L129-L163) is using [getDefaultPositionRealUnit](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L149) which in tern return the **current** values of tokens, and in the middle of the bid's transfer we have actually increased the total value of one of the ERC's without minting any new setTokens.

**Example:**
<details><summary> Some prerequisites </summary> 
We are gonna use any ERC777 token that can reenter. No matter which token as long as it has beforeTransfer or afterTransfer hook.

- Price of MKR  : 1000$
- Price of WETH: 2000$
- Set token with MRK : WETH  has 1000 MKR and 100 WETH, attacker owns 10% of it, amount equivalent to 100MKR and 10WETH
- Attacker also holds 50 WETH in his contract, this is needed for the execution.

</details>

1. This setToken is scheduled for auction with quote MKR and increasing in WETH and lowering in MKR from 10:1 to 5:1
2. Attacker's contract [sells](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L309) 50 WETH for  100 MKR and in the middle of the sell triggers a [re-entrancy](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L934-L953)
```jsx
    function _executeBid(BidInfo memory _bidInfo) internal {
        transferFrom(
            _bidInfo.receiveToken,
            msg.sender,
            address(_bidInfo.setToken),
            _bidInfo.quantityReceivedBySet
        );
//@audit re-enter here
        _bidInfo.setToken.strictInvokeTransfer(
            address(_bidInfo.sendToken),
            msg.sender,
            _bidInfo.quantitySentBySet
        );
    }
```
3. In the middle state after  [transferFrom](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L940-L945) and before [strictInvokeTransfer](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L948-L952) the setToken has 1000 MKR and 150 WETH.
4. The attacker's contract calls [redeem](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L129-L163) and trades it's setTokens (10% of them) for 10% of the contracts assets 100MKR and 15WETH
5. [_executeBid](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L934-L953) continues where [strictInvokeTransfer](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L948-L952) sends the rest 100 MKR

Here are the profits and from the flow above we can recall that the attacker stole 5 WETH or 10k in USD value, **which with every example is 10% of the initial bid made by the attacker**. And because all of this can be achieved in one TX (issue, bid, redeem), it means it can be flash-loaned.

|                           |  MKR | WETH | value in USD |
|---------------------------|:----:|:----:|:------------:|
|         **Before**        |      |      |              |
| setToken                  | 1000 |  100 |   1.2m USD   |
| Attacker's funds in setT  |  100 |  10  |  120 000 USD |
| Attacker's contract funds |   0  |  50  |  100 000 USD |
|         **After**         |      |      |              |
| setToken                  |  800 |  135 |     1.07m USD  |
| Attacker's contract       |  200 |  15  |  230 000 USD |

<details><summary> If you need help with the table </summary> 
It might be confusing with all these numbers so let me simplify it.

 1. Before the attack attacker had 100 MKR and 10 WETH **in the setToken contract**  and 50 WETH on it's own contract, if he wanted he could have redeem before and gotten 100MKR : 60 WETH 
 2. SetToken has ownership of the attacker funds (and many other users) so it has in total 1000 MKR and 100 WETH
 3. By the end the attacker gamed the system with 5 WETH  which is 10% of his trade when calling bid.

</details>

 **Final note:**
Because of the `redeem` in the middle of the transaction (after Index has received our funds) we are redeeming a percent of our send funds, equal to the percentage of the pool that we owned before calling `bid`. As in the example above, we owned 10% of the pool before the TX and we "stole" 10% of our bid.

## Impact
Any pool with ERC777 tokens is in danger of this attack.

## Code Snippet

## Tool used

Manual Review

## Recommendation
You can implement a check in [BasicIssuanceModule](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol) to check if the caller is in a bid currently or has bid before (like a cool-down feature).