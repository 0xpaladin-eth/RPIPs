---
rpip: 72
title: rETH Withdrawal Liquidity via EIP-7002 and EIP-7251
description: Enable rETH stakers to access protocol liquidity for unstaking from Rocket Pool via partial withdrawals enabled by Pectra
author: Paladin (@0xpaladin.eth)
discussions-to: <URL> TODO
status: Draft
type: Protocol
category: Core
created: 2025-05-20
requires (*optional): 
---

# WITHDRAWN

Parital withdrawals aren't significantly faster than exits per https://piertwo.com/insights/pectra-withdrawals-explained, so it's not a big improvement over RPIP-71

## Abstract
rETH does not currently have a mechanism for stakers to burn (aka redeem) rETH for the full fair value of the underlying protocol ETH, even if they were prepared to wait for a validator exit.

RocketTokenRETH.sol
```
function burn(uint256 _rethAmount) override external {
    ...
    uint256 ethBalance = getTotalCollateral();
    require(ethBalance >= ethAmount, "Insufficient ETH balance for exchange");
    ...
}
```

There's an implicit assumption when you stake ETH with a liquid staking protocol, that you can get back the underlying assets. The inability of rETH to provide this guarantee is weakness of the protocol and is evidenced by stakers regularly appearing in Discord asking how they can redeem for full fair value and being both surprised and disappointed (or worse), when they learn it's not possible.

In the current state, when an rETH staker is unable to burn their rETH directly, the solution is to trade on a secondary market, which may be at a discount to the true value of the protocol's underlying assets (sellers earn a liquidity premium). As the discount widens, node operators are incentived to exit to capture the discount, which does eventually closes the gap.

The reason for this state is that prior to the Pectra hard fork, there was no way for the protocol to "force exit" validators to make liquidity available to burn. When the protocol was growing and especially when deposits were growing faster than node operator supply, this wasn't an issue as there was always 'idle ETH' in the deposit tool to meet redemptions. When the protocol is shrinking, flat, or simply has a surplus of node operators, that liquidity is not available.

This RPIP proposes using EIP-7002 Execution Triggerable Withdrawals + EIP-7251 Max EB to give the protocol a mechanism to:

- Provide liquid ETH to meet burns in a reasonable timeframe (usually next block)
- That is not a drag on protocol APR
- That does not require validators to be force exited

At a high level, the proposal is to maintain a target liquidity level, e.g. 2% of protocol ETH in MaxEB validators (validators with balance >32 ETH). When below that target, deposits are added to existing validators up to some max balance, e.g. 128 ETH, instead of adding new minipools. When the target is met, new minipools are created as normal. 

If a staker wants redeem (burn) rETH for ETH and there insufficent ETH, they can trigger a partial withdrawal from a MaxEB validator, which typically completes in the next block, giving them "guaranteed liquidity at the protocol fair value"

## Motivation

- Lack of guaranteed burn is a bad staker experience and often a shock
- The protocol leaks value to bots that capture the discount when node operators exit without "running the magic script" to capture it themselves
- It makes rETH a riskier asset in "looping strategies" whereby rETH is deposited as collateral to borrow ETH, to then mint more rETH to increase returns, because the potential for a widening gap to fair value means increased liquidation risk
- In finance, the best collateral is one that has deep liquidity even in times of market stress. rETH's discount during fairly normal market conditions hurts its reputation as collateral.

### Relationship to RPIP-71

This RPIP is an alternative to the soluton proposed in RPIP-71. Specifically, it seeks to use partial withdrawals to avoid forced exits. This is because I believe:

- It's a superior node operator experience (no one joins in the hope of being force exited)
- Will be more responsive than forced exits (the partial withdrawal queue processes faster than the exit queue)
- (Hopefully) For a similar level of complexity

## Specification

### Goal
- Add new deposits to MaxEB validators until the target protocol liquidity is met, otherwise process as normal
- Add a new method that allows a staker to request a partial withdrawal from those validators.

### Pseudo code

I offer the below to show the core logic is not complex, even if some of the implementation details may indeed be complex, ssuch as tracking MaxEB validator indices and having the notion of a new 'class' of validators, the MaxEBs.

Introduce 2 new constants (potentially upgraded to UARS variables in the future for fine tuning)

```
//The target amount of ETH to be stored on MaxEB validators and be made available for partial withdrawals.
PROTOCOL_TARGET_WITHDRAWABLE_LIQUIDITY_PERCENT=2%`

//The target size of such validators
MAX_WITHDRAWABLE_BALANCE_ETH=128
```

#### Processing Deposits
```
//Calculate what's amount of ETH that would meet the target liquidity percentage
protocol_target_withdrawable_liquidity_in_eth = PROTOCOL_TARGET_WITHDRAWABLE_LIQUIDITY_PERCENT * total protocol assets;

//Calculate how far away we are
protocol_withdrawable_liquidity_current = sum of all (balance-32ETH) of MaxEB validators

if protocol_withdrawable_liquidity_current < protocol_target_withdrawable_liquidity_in_eth
    //If we're below, route ETH to MaxEBs
    target_maxEB_validator = select the target validator()
    //depositing  up to MAX_WITHDRAWABLE_BALANCE_ETH
    deposit (MAX_WITHDRAWABLE_BALANCE_ETH - target_maxEB_validator.balance)
    repeat until the deposit pool is empty or protocol_target_withdrawable_liquidity = true
else
    //otherwise
    process as normal and create minipools
```

#### Processing Withdrawals

Existing burn mechanism is unchanged.

Add a new method, which requests an EIP-7251 partial withdrawal;
```
function requestBurn(uint256 _rethAmount) override external {
    //otherwise, request a partial withdrawal (reuquires a fee in ETH, UI should calculate)
    target_maxEB_validator = select the target validator();
    (bool ret, ) = eip7002WithdrawalContract.call{value: msg.value}(abi.encodePacked(target_maxEB_validator.pubkey, _rethAmount));
    //if it worked, send the ETH on to msg.sender
    //Will arrive "soon", probably next block
}
```

_Copied from RPIP-71_
- The priority selection of which validators to exit to provide protocol liquidity for pool stakers MAY take the following factors into consideration:
    - RPL stake of the node that owns the validator.
    - Performance of the validator, as determined permissionlessly via a beacon chain balance proof.
    - Size of the node that owns the validator, in terms of total node operator-provided ETH.
    - Commission percentage for the validator.

## Open questions
- Is there anything about the protocol or Megapools that precludes MaxEB?
- How gas efficient is it to calculate `protocol_withdrawable_liquidity_current`? The higher the MAX_WITHDRAWABLE_BALANCE, the fewer balances to add together. OR, just keep a running total on deposits and withdrawals.
- MEV theft? At what balance does MEV theft become a problem?
- What's the selection heuristic on withdrawal? Who gets drawn down first?
- Partial withdraws require 0x1 addresses... do we have 0x0 addresses? Can we just exclude them?

## Rationale
The rationale for partial withdrawals over validator exits (besides node operators not wanting to get force exited), is that the partial withdrawal queue is very efficient compared to validator exits.
Parital withdrawal queue = 16 per block = 115,200 per day
Exit queue = 16 per epoch = 3,600 per day (with a large backlog)

## Security Considerations

Higher balances = risk of MEV theft. What's the number where that becomes a problem?

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
