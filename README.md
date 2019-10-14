# TODO
- Initial state is not so clear. Need to understand how to undestand from where smart contract receive money. Possible solution - we initialize smart contract (creating boc with provided amount a and b and we don't make any transactions on top of initial state before smart contract doesn't get money from both parties). But it works only for onedirectional funding (when a == 0 or b == 0).

To make some atomic swaps to top up channel for B from TON network need to support HTLC.

# Smart contract description

## Persistant state of smart contract:

*persistant state*
(
    `publicKeyA`     uint256, 
    `publicKeyB`     uint256, 
    `state`          uint4, 
    `closingTime`    uint32 (lt of ut?), 
    `closingHeight`  uint64, 
    `closingAmountA` Gram (uint64),
    `closingAmountB` Gram (uint64),
)

`publicKeyA`, `publicKeyB` - public keys of 2 parties that are holding the channel;

`state` - smart contract state:
- *0* - _created_;
// TODO: rewrite so B will it should include coop closing if B posts the same state
- *1* - _unidirect closing by A_ - means that contract got unilateral closing message from A and waiting a timelock so B has time to send revokation message. If such message will not be published to the smart contract during timelock A will get his money (B gets money immediatelly after A posts closing message as provided in it). If closing message from A is invalid or if B posts proof that A post incorrect amounts B will receive all money from the channel. If B provided incorrect proof of A cheating A will receive his money immediatelly;
- *2* - _unidirect closing by B_ - the same as *1* but for B;

`closingTime` - time when unilateral closing message was sent;

`closingHeight` - height of provided offchain state that was provided by closing party;

`closingAmountA`, `closingAmountB` - amounts of parties that was provided in unilateral closing message;

# Scripts description

- send money to counterparty
- sign counterparty message (if it is valid) and save it into a file as a last state
- save counterparty signature of your transaction as a last state (if valid) 
- close channel

# Message structure

First party (A) public transaction for closing the channel:

*closing transaction*
(
    `closingHeight`  uint64, 
    `closingAmountA` Gram,
    `closingAmountB` Gram,
    cellRef (
        `signatureA` uint512,
    )
    cellRef (
        `signatureB` uint512,
    )
)

Second party could public its closing transaction during `timelock` (24 hours) period. If it doesn't do it the correct amount distribution will be as in closing transaction provided by A. In other cases, smart contract will validate closing transaction provided by B and compare its state to one provided by A.