carrot

high

# Interest earned through lending is forever locked in bank contract

## Summary
BlueBerryBank deposits tokens into SoftVault which in turn deposits them to the Lending protocol which earns interest. The protocol has no code to extract this interest earned and caps the amount of collateral a user can withdraw to their original value. Thus any interest earned will be locked in the BlueBerryBank contract and cannot be taken out to the treasury.
## Vulnerability Detail
The SoftVault collects interest through the Lending protocol, and these interests are then withdrawn by the bank when a user is withdrawing their underlying using the function `withdrawLend` through a spell. This function caps the user withdraws to their original amount.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L682-L701

Thus the interest earned is withdrawn to the bank, but there is no way of taking it out to be used in some productive way and is forever locked in.
## Impact
Loss of interest due to absence of contract functionality
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L682-L701
## Tool used

Manual Review

## Recommendation
During `withdrawLend`, transfer `wAmount - pos.underlyingAmount` amount of tokens to the protocol treasury