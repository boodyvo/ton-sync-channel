# Smart contract description

## Persistant state of smart contract

**Structure**:
    
    publicKeyA     uint256, 
    publicKeyB     uint256, 
    state          uint4, 
    closingTime    uint32 (lt of ut?), 
    closingHeight  uint64, 
    closingAmountA Grams (uint64),
    closingAmountB Grams (uint64),
    initAmountA    Grams (uint64),
    initAmountB    Grams (uint64),

- `publicKeyA`, `publicKeyB` - public keys of 2 parties that are holding the channel;

- `state` - smart contract state:
    - **0** - _created_;
    - **1** - _start closing by A_ - means that contract got closing message from A and waiting a timelock so B has time to send its closing message. If such message will not be published to the smart contract during timelock A and B will get their money with distribution in transaction provided by A. If closing message from A is invalid or if B posts proof that A post incorrect amounts B will receive all money from the channel. If B provided incorrect proof of A cheating A will receive his money immediatelly;
    - **2** - _start closing by B_ - the same as *1* but vise versa for A and B;
    - **3** - _possible to close_ - balances could be sent to parties;

- `closingTime` - time when unilateral closing message was sent;

- `closingHeight` - height of provided offchain state that was provided by closing party;

- `closingAmountA`, `closingAmountB` - amounts of parties that was provided in unilateral closing message;

# Scripts description

- **new-channel-a.fif** - generate new channel address and closing transaction for the channel for A;
- **new-channel-b.fif** - generate new channel address and closing transaction for the channel for B;
- **create-initial-state-a.fif** - create initial state and sign it for B;
- **create-initial-state-b.fif** - create initial state and sign it for A;
- **create-transaction-a.fif** - send money (if possible) offchain to B;
- **create-transaction-b.fif** - send money (if possible) offchain to A;
- **validate-transaction-a.fif** - validate transaction from create-transaction-b script (or validate-transaction-b if it was own transaction) and rewrite last-confirmed-state-a.boc;
- **validate-transaction-b.fif** - validate transaction from create-transaction-a script (or validate-transaction-a if it was own transaction) and rewrite last-confirmed-state-b.boc;
- **parse-state.fif** - parse last-confirmed-state for both participants (for debug purposes);

## Flow of file exchange

All files for now in one repository. Channel will be unidirectional (5000000000, 0). Private and public keys for debug purposes are pregenerated and saved in repository (own.pk, own.pub - for a; party.pk, party.pub - for b).

**Steps:**
- generate initial state with `create-initial-state-a.fif` and `create-initial-state-b.fif`:
```bash
fift -s create-initial-state-a.fif 5000000000
fift -s create-initial-state-b.fif 5000000000
```
- compile channel smart-contract code:
```bash
func -P -o channel-code.fif stdlib.fc channel.fc
```
- generate smart-contract address with `new-channel.fif`:
```bash
fift -s new-channel.fif -1 5 own.pub party.pub own.pk
```
- send money from a to b with `create-transaction-a.fif`:
```bash
fift -s create-transaction-a.fif 5000000000 own.pk
```
- validate transaction by B with `validate-transaction-b.fif`:
```bash
fift -s validate-transaction-b.fif party.pk own.pub 0
```
- validate generated in response transaction by A with `validate-transaction-a.fif`:
```bash
fift -s validate-transaction-a.fif own.pk party.pub -1
```

To check current confirmed state check 

# Messages structure

**Closing transaction structure:**

    closingHeight  uint64, 
    closingAmountA Gram,
    closingAmountB Gram,
    cellRef (
        signatureA uint512,
    )
    cellRef (
        signatureB uint512,
    )

# TODO
- Initial state is not so clear. Need to understand how to undestand from where smart contract receive money. Possible solution - we initialize smart contract (creating boc with provided amount a and b and we don't make any transactions on top of initial state before smart contract doesn't get money from both parties). But it works only for onedirectional funding (when a == 0 or b == 0).
To make some atomic swaps to top up channel for B from TON network need to add support of HTLC to the contract.

- Improve fift script so it could be as one script as cli for the channel symetric for both parties.

- Add defer transaction creation for smart contract to send money if state becomes 3.

- Add wallets (addresses) for withdraw.

- Remove unnesessary get methods to reduce fee for storage.

- Problems with paying for storage so total amount will be less than initial.

- Need to be improved mechanism of message exchange because if second party (B, that already obtained new signed) doesn't send appropriate signature to its party (A), first party (A) couldn't close the channel.

- Implement `recv_internal` function instead of `recv_external` function so parties could post transactions only from their wallets and pays for them (so not do so the channel smart-contract). Any of parties could spend all money (including money of second part) as a fee just sending some messages so need to create some antifroud mechanism.
**Example**: There is a channel with (A, 0) distribution and second part (with 0) sends a lot of requests white distribution becomes (0, 0).
