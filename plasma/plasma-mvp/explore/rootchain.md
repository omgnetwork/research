Exploring [`plasma-mvp`](https://github.com/omisego/plasma-mvp): Root Chain Contracts
===

Author: Kelvin Fichter

---

**Note:** This document is part of a series titled "Exploring `plasma-mvp`" that attempts to document and explain the underlying functionality of `plasma-mvp`. I created this series so that I (and hopefully others) could gain a deeper understanding of every aspect OmiseGO's `plasma-mvp` codebase. This document *will* change as the codebase changes.

Each document will begin with a general target audience.

Feedback and request for changes are always appreciated!

---

## Target Audience

This document targets **developers who want to gain a deeper understanding of what a Plasma implementation actually looks like under the hood**. As a result, the rest of this document assumes you have a basic working knowledge of smart contracts on Ethereum. Zeppelin Solutions' [Hitchhiker’s Guide to Smart Contracts](https://blog.zeppelin.solutions/the-hitchhikers-guide-to-smart-contracts-in-ethereum-848f08001f05) is a great place to start if you'd like to learn more.

---

## Root chain

Any Plasma implementation requires at least one smart contract be deployed to whatever blockchain we're using as a "root chain". Before we start describing what that might look like, it's useful to define a root chain. When you hear "root chain" in regard to Plasma, it's referring to the underlying blockchain that we're eventually going to have to submit blocks to. `plasma-mvp` is built on top of Ethereum, so any references to the root chain throughout the rest of this document will also be referring to Ethereum.

**Note:** The [Plasma paper](https://plasma.io/plasma.pdf) briefly mentions using more than one root chain. Current implementations (including `plasma-mvp`) don't require this or make use of it. There might eventually be some benefits to this construction, but it doesn't seem like that's been explored significantly (yet).

## Root chain contracts

Plasma smart contracts generally need to be able to do the following things:

1. Allow users to submit deposits into the Plasma chain.
2. Allow users to exit from the Plasma chain.
3. Allow users to challenge an exit.
4. Allow some operator to submit Plasma blocks.

We'll define some key terms before we continue.

A `deposit` can be made in any coin or token. `plasma-mvp` only allows deposits in ETH for simplicity, but you could theoretically allow for deposits in any type of asset. The Plasma paper assumes that most production Plasma implementations will accept deposits in ERC20 (or a similar standard) tokens.

Users can `exit` from the Plasma chain by referring to a specific [Unspent Transaction Output (UTXO)](https://bitcoin.org/en/glossary/unspent-transaction-output). An exit is really just a claim by a user stating that the user has the right to spend a specific UTXO. Exits are finalized after a waiting period, during which the exit can be challenged. Other users can attempt to prove that the exit is invalid by issuing a challenge. A successful challenge blocks ("invalidates") the exit.

A `Plasma block` is very similar to an Ethereum block. It's effectively a Merklized set of transactions, along with some metadata and a signature attesting that it was created by the `authority`. 

The Plasma `authority` is a deliberately vague term. When we talk about the Plasma authority from a high level standpoint, we're describing some mechanism that submits Plasma block to the root chain. This could really be any sort of consensus mechanism and entirely depends on the needs of the individual Plasma chain. For the purposes of `plasma-mvp`, this authority is just one user who's permitted to submit Plasma blocks. However, the `authority` could also be a group of people through something like proof-of-stake, a few trusted users, or any mechanism in between.

**Note:** Just because `plasma-mvp` has a centralized authority **does not** mean that Plasma is centralized or that all Plasma implementations will be centralized. No matter what, Plasma derives its security from the root chain, so a bad authority can't steal your money as long as you're vigilant and exit when things go badly. `plasma-mvp` is a proof of concept Plasma implementation designed to demonstrate that the core ideas behind Plasma actually do, in fact, work. My guess is that most public Plasma implementations will use some decentralized proof-of-stake mechanism. 

Now that that's out of the way, we'll begin a line-by-line breakdown of how `plasma-mvp` accomplishes the above goals.

### Imports

Let's start with some imports:

```
// RootChain.sol

pragma solidity 0.4.18;
import 'SafeMath.sol';
import 'Math.sol';
import 'RLP.sol';
import 'Merkle.sol';
import 'Validate.sol';
import 'PriorityQueue.sol';

...
```

`SafeMath` is used throughout `plasma-mvp` to make sure that math operatons are carried out safely. You can read more about it [here](https://ethereum.stackexchange.com/questions/25829/meaning-of-using-safemath-for-uint256).

`Math` doesn't actually seem to be used anywhere in `plasma-mvp` and might just be an unnecessary import. 

`RLP` is a library for [RLP decoding](https://github.com/ethereum/wiki/wiki/RLP). This is specifically used to decode Plasma transactions. We'll get to that later.

`Merkle` is a library that allows for the verification of Merkle proofs, explained well in [this article](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum/) by Vitalik. We'll use these proofs whenever users start or challenge an exit in order to concisely validate that some transaction was actually included in a particular block.

`Validate` is used to check that the signatures attached to a transaction are valid.

`PriorityQueue` is pretty self descriptive. Exits are given a priority based on the age of their corresponding input and are therefore held in a priority queue. This priority exists so that we're always processing exits in correct age order. If a Byzantine authority creates an invalid block, then all earlier (valid) transactions will process correctly before the attacker can attempt to exit from the invalid transactions. More on this later.

### `Using For` statements

The contract starts off by declaring that it's using some libraries for certain types:

```
...

contract RootChain {
    using SafeMath for uint256;
    using RLP for bytes;
    using RLP for RLP.RLPItem;
    using RLP for RLP.Iterator;
    using Merkle for bytes32;
    
...
```

This behavior is described in more detail in the [Solidity docs](https://solidity.readthedocs.io/en/develop/contracts.html#using-for). Basically this just attaches library functions to objects of a certain type. The object that the function is called on will be passed as the first parameter to that function. For example, the statement `using SafeMath for uint256;` means we can later do something like:

```
currentChildBlock = currentChildBlock.add(1);
```

instead of 

```
currentChildBlock = SafeMath.add(currentChildBlock, 1);
```

Later on we'll get to how the contract actually uses these libraries.

### Events

`RootChain.sol` defines two events, `Deposit` and `Exit`:

```
...

    event Deposit(address depositor, uint256 amount);
    event Exit(address exitor, uint256 utxoPos);

...
```

`Deposit` is called whenever a deposit is made. It emits the address of the user that created the deposit and the deposit amount (in ETH). The Plasma client watches for this event and automatically adds a block locally whenever a deposit is made to the root chain. 

`Exit` is called whenever an exit is *started*. It emits the address of the user that created the exit and the "position" (`utxoPos`) of the UTXO being exited in the blockchain. This `utxoPos` is determined by `blknum * 1000000000 + index * 10000 + oindex`. `blknum` is the number of the block in which the UTXO was included. `index` is the index of the transaction within that block. `oindex` is either 0 or 1, depending on which of the transaction's multiple outputs is being exited (transactions in `plasma-mvp` currently have two outputs). Clients listen to this event and challenge the exit if it's invalid.

### Storage

We define some structs and variables that make up the storage of our contract:

```
...

    mapping(uint256 => childBlock) public childChain;
    mapping(uint256 => exit) public exits;
    mapping(uint256 => uint256) public exitIds;
    PriorityQueue exitsQueue;
    address public authority;
    uint256 public currentChildBlock;
    uint256 public recentBlock;
    uint256 public weekOldBlock;

    struct exit {
        address owner;
        uint256 amount;
        uint256 utxoPos;
    }

    struct childBlock {
        bytes32 root;
        uint256 created_at;
    }

...
```

`childChain` maps from a set of `uint256` block numbers to a set of `childBlock` structs. Each `childBlock` consists of a `bytes32` [Merkle root](https://bitcoin.stackexchange.com/questions/10479/what-is-the-merkle-root) of the transaction set in the block and a `uint256` timestamp that represents the time the block was created. This timestamp is determined by the `block.timestamp` of the Ethereum block in which the Plasma block submission transaction was included. We need this timestamp to make sure that outputs in blocks older than some amount of time (1 week, in our case) are given some maximum priority. 

`exits` maps from a set of `uint256` priorities to a set of `exit` structs. Each `exit` is comprised of the claimant, the amount being exited, and the "position" of the referenced UTXO (as described above). Priority should be unique so that these exits can't be overwritten. 

`exitIds` maps from a set of `uint256` UTXO positions to a set of `uint256` priorities. This feature was designed for usability so that users only need to maintain a single unique value (`utxoPos`). This should probably be removed and be replaced with client-side method of converting from `utxoPos` to `priority`. 

`exitsQueue` is a priority queue that holds a list of `priorities` in decreasing order (highest priority first). This is used when processing exits so that the oldest outputs (with the highest priority) are processed before newer outputs. 

`authority` is the address of the user allowed to submit Plasma blocks. This is only a single user in `plasma-mvp` but, as stated before, this can be modified to be some decentralized mechanism.

`currentChildBlock` is the block number of the current Plasma block. This is used to make sure that we know where to insert new blocks into the `childChain` mapping (because maps aren't lists and don't have an `append` method).

`recentBlock` isn't actually used anywhere. We need to get rid of that. Don't worry about it.

`weekOldBlock` keeps track of the number of the oldest Plasma block that's less than a week old. This is used to limit the maximum `priority` that any transaction can have. Without this, outputs greater than two weeks old would be able to exit immediately and we definitely don't want that!

### Modifiers

Now let's start getting into some contract logic! `RootChain.sol` currently has two modifiers. There's definitely some room for improvement here.

```
...

    modifier isAuthority() {
        require(msg.sender == authority);
        _;
    }

...
```

`isAuthority` is a simple modifier that verifies that the sender is also the specified `authority`. Nothing too complex. 

```
...

    modifier incrementOldBlocks() {
        while (childChain[weekOldBlock].created_at < block.timestamp.sub(1 weeks)) {
            if (childChain[weekOldBlock].created_at == 0) 
                break;
            weekOldBlock = weekOldBlock.add(1);
        }
        _;
    }

...
```

`incrementOldBlocks` is a weird one. The idea here is to make sure that we're constantly updating our `weekOldBlock`. `incrementOldBlocks` is added to a few functions so that there's (hopefully) never a case in which the `while` loop costs a prohibitive amount of gas. There's probably a better way to do this. If you've got ideas, feel free to make a pull request!

The logic here is pretty simple - if the current `weekOldBlock` is more than a week old, increment `weekOldBlock` by one and check again. If `weekOldBlock` would ever point to a block that doesn't exist (meaning we haven't had a child block in at least a week), then `weekOldBlock` won't be incremented further. 

### Functions

We're going to explore each contract function one by one. If we get to a library call that's worth explaining, we'll go through that too. Without further ado, here's the constructor:

#### `RootChain`

```
...

    function RootChain()
        public
    {
        authority = msg.sender;
        currentChildBlock = 1;
        exitsQueue = new PriorityQueue();
    }

...
```

The constructor doesn't really do much - it just sets `authority` as the contract creator, `currentChildBlock` to 1, and `exitsQueue` to a new instance of `PriorityQueue`. I'm not going to go into it here, but if you're interested in seeing the source code for `PriorityQueue`, it's available on [GitHub](https://github.com/omisego/plasma-mvp/blob/master/plasma/root_chain/contracts/DataStructures/PriorityQueue.sol).

#### `submitBlock`

Here's the logic for submitting blocks:

```
...

    // @dev Allows Plasma chain operator to submit block root
    // @param root The root of a child chain block
    function submitBlock(bytes32 root, uint256 blknum)
        public
        isAuthority
        incrementOldBlocks
    {
        require(blknum == currentChildBlock);
        childChain[currentChildBlock] = childBlock({
            root: root,
            created_at: block.timestamp
        });
        currentChildBlock = currentChildBlock.add(1);
    }

...
```

This method takes the `bytes32 root` of the child chain block and the block's `uint256 blknum` as parameters.

We've attached both of our modifiers to this function so the user calling this function `isAuthority` and any attempt to submit blocks will also `incrementOldBlocks`.

There's a check here to make sure that `blknum == currentChildBlock`. This has the effect of preventing an `authority` from ever accidentally submitting the wrong block or submitting blocks in the wrong order. 

If everything checks out, we insert a new `childBlock` with the given `root` and `created_at: block.timestamp`. Finally, we add 1 to the `currentChildBlock`.

#### `deposit`

`deposit` is where things start to get a little bit more complicated, so I'll descrbie things in more detail. The idea behind `deposit` is to create a new `childBlock` with a *single transaction* that creates a UTXO for the user "out of thin air". 

```
...

    // @dev Allows anyone to deposit funds into the Plasma chain
    // @param txBytes The format of the transaction that'll become the deposit
    // TODO: This needs to be optimized so that the transaction is created
    //       from msg.sender and msg.value
    function deposit(bytes txBytes)
        public
        payable
    {
        var txList = txBytes.toRLPItem().toList(11);
        require(txList.length == 11);
        for (uint256 i; i < 6; i++) {
            require(txList[i].toUint() == 0);
        }
        require(txList[7].toUint() == msg.value);
        require(txList[9].toUint() == 0);
        bytes32 zeroBytes;
        bytes32 root = keccak256(keccak256(txBytes), new bytes(130));
        for (i = 0; i < 16; i++) {
            root = keccak256(root, zeroBytes);
            zeroBytes = keccak256(zeroBytes, zeroBytes);
        }
        childChain[currentChildBlock] = childBlock({
            root: root,
            created_at: block.timestamp
        });
        currentChildBlock = currentChildBlock.add(1);
        Deposit(txList[6].toAddress(), txList[7].toUint());
    }
    
...
```

`deposit` only takes a single argument, `txBytes`. `txBytes` is an RLP encoded `plasma-mvp` transaction. Specifically, this transaction is the Plasma transaction that will create a UTXO for the user "out of thin air". Transactions in `plasma-mvp` are a little different from the ones specified in [Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) and have the following form:

```
[
    blknum1, txindex1, oindex1, # Input 1
    blknum2, txindex2, oindex2, # Input 2
    newowner1, amount1, # Output 1
    newowner2, amount2, # Output 2
    fee, # Fee
    sig1, sig2 # Signatures
]
```

So when you see something like `txBytes[i]`, it's referring to the component of the transaction at the index `i`. For example, `txList[6]` refers to `newowner1` and `txList[7]` refers to `amount1`.

A valid `deposit` transaction for 1 ETH will look like this (before RLP encoding):

```
[
    0, 0, 0, # Input 1
    0, 0, 0, # Input 2
    0x281055afc982d96fab65b3a49cac8b878184cb16, 1000000000000000000, # Output 1
    0, 0, # Output 2
    0, # Fee
    0, 0 # Signatures
]
```

**Note:** The requirement that the user submit an encoded transaction will probably change if we can cheaply RLP encode in the contract. This is an [open problem](https://github.com/omisego/plasma-mvp/issues/65) and PRs are welcome!

The first thing we're doing with

```
var txList = txBytes.toRLPItem().toList(11);
```

is using the `RLP` library to decode our `txBytes` to a list of 11 components (because our transaction has 11 parts). Remember, we can do this because we specified that our contract is:

```
using RLP for bytes;
using RLP for RLP.RLPItem;
using RLP for RLP.Iterator;
```

So now we've got a list that looks like the transaction specified above. We then `require(txList.length == 11);` to make sure that the transaction provided is, in fact, 11 parts.

Next, we make sure that the first 6 components are set to 0.

```
for (uint256 i; i < 6; i++) {
    require(txList[i].toUint() == 0);
}
```

We do this because the transaction is created "out of thin air", so there's no input to reference.

We also verify that `txIndex[7]` (the first output amount, `amount1`) is equal to the `msg.value` and that `txIndex[9]` (the second output amount, `amount2`) is equal to 0.

```
require(txList[7].toUint() == msg.value);
require(txList[9].toUint() == 0);
```

This way we're sure that the deposit amount specified in the user's `txBytes` is correct.

Now we do some hashing to build up a height 16 Merkle tree:

```
bytes32 zeroBytes;
bytes32 root = keccak256(keccak256(txBytes), new bytes(130));
for (i = 0; i < 16; i++) {
    root = keccak256(root, zeroBytes);
    zeroBytes = keccak256(zeroBytes, zeroBytes);
}
```

All we're doing is repeatedly hashing the deposit transaction with a bunch of zero bytes. This is done so that `deposit` blocks look just like any other block (which also use height 16 Merkle trees). Fun fact, this means `plasma-mvp` permits a maximum of 65536 transactions per block. 

Finally, we insert the `childBlock` and emit a `Deposit` event.

```
childChain[currentChildBlock] = childBlock({
    root: root,
    created_at: block.timestamp
});
currentChildBlock = currentChildBlock.add(1);
Deposit(txList[6].toAddress(), txList[7].toUint());
```

#### `startExit`

On to even more complicated stuff: exits. `startExit` allows a user to begin the process of exiting the Plasma chain. The user needs to specify what specific UTXO they're withdrawing, along with the raw transaction that created that output, proof that the transaction is actually in the specified block, and the signatures that prove the transaction is valid.

```
...

    // @dev Starts to exit a specified utxo
    // @param utxoPos The position of the exiting utxo in the format of blknum * 1000000000 + index * 10000 + oindex
    // @param txBytes The transaction being exited in RLP bytes format
    // @param proof Proof of the exiting transactions inclusion for the block specified by utxoPos
    // @param sigs Both transaction signatures and confirmations signatures used to verify that the exiting transaction has been confirmed
    function startExit(uint256 utxoPos, bytes txBytes, bytes proof, bytes sigs)
        public
        incrementOldBlocks
    {
        var txList = txBytes.toRLPItem().toList(11);
        uint256 blknum = utxoPos / 1000000000;
        uint256 txindex = (utxoPos % 1000000000) / 10000;
        uint256 oindex = utxoPos - blknum * 1000000000 - txindex * 10000;
        bytes32 root = childChain[blknum].root;

        require(msg.sender == txList[6 + 2 * oindex].toAddress());
        bytes32 txHash = keccak256(txBytes);
        bytes32 merkleHash = keccak256(txHash, ByteUtils.slice(sigs, 0, 130));
        require(Validate.checkSigs(txHash, root, txList[0].toUint(), txList[3].toUint(), sigs));
        require(merkleHash.checkMembership(txindex, root, proof));

        // Priority is a given utxos position in the exit priority queue
        uint256 priority;
        if (blknum < weekOldBlock) {
            priority = (utxoPos / blknum).mul(weekOldBlock);
        } else {
            priority = utxoPos;
        }
        require(exitIds[utxoPos] == 0);
        exitIds[utxoPos] = priority;
        exitsQueue.insert(priority);
        exits[priority] = exit({
            owner: txList[6 + 2 * oindex].toAddress(),
            amount: txList[7 + 2 * oindex].toUint(),
            utxoPos: utxoPos
        });
        Exit(msg.sender, utxoPos);
    }
    
...
```

Once again we add our `incrementOldBlocks` modifier. This is necessary because we're actually going to be using `weekOldBlock` here. 

The four function parameters are `uint256 utxoPos`, `bytes txBytes`, `bytes proof`, and `bytes sigs`. 

`utxoPos` is a unique UTXO identifier that's given by `blknum * 1000000000 + txindex * 10000 + oindex`. For example, if I'm attempting to exit the second UTXO of the second transaction in the fifth Plasma block, my `utxoPos` would be `utxoPos = blknum * 1000000000 + txindex * 10000 + oindex = 5 * 1000000000 + 1 * 10000 + 1 = 5000010001`. This `utxoPos` is unique as long as we put some constraints on how many transactions a single block can have. 

`txBytes` is another RLP encoded transaction, but this time it's the raw transaction that created the UTXO at `utxoPos`. 

`proof` is a Merkle proof that the transaction given by `txBytes` is actually the transaction that created the UTXO at `utxoPos`. 

`sigs` is the signatures to make this transaction, concatenated into a single `bytes`. `sigs` has the form `sig1 + sig2 + confsig1 + confsig2` where each signature is 65 bytes long. We use a library `ByteUtils` to slice off individual signatures from the larger byte string, i.e. `ByteUtils.slice(sigs, 0, 130)` is `sig1 + sig2`.

Now to the logic. First we convert our `txBytes` to a list of components like we did in `deposit`:

```
var txList = txBytes.toRLPItem().toList(11);
```

Next we decompose our `utxoPos` into its components:

```
uint256 blknum = utxoPos / 1000000000;
uint256 txindex = (utxoPos % 1000000000) / 10000;
uint256 oindex = utxoPos - blknum * 1000000000 - txindex * 10000;
```

This is effectively just doing the opposite of `blknum * 1000000000 + txindex * 10000 + oindex`. 

We also pull the Merkle root of the block that corresponds to `blknum`:

```
bytes32 root = childChain[blknum].root;
```

Now we're verifying that the user creating the exit actually owns the output being referenced:

```
require(msg.sender == txList[6 + 2 * oindex].toAddress());
```

Remember that `txList[i]` represents some component of the transaction. `oindex` can either be 0 or 1 (first or second output), so `6 + 2 * oindex` is either `6` or `8`. `txList[6]` is `newowner1` and `txList[8]` is `newowner2`.

Next we're validating signatures:

```
bytes32 txHash = keccak256(txBytes);
bytes32 merkleHash = keccak256(txHash, ByteUtils.slice(sigs, 0, 130));
require(Validate.checkSigs(txHash, root, txList[0].toUint(), txList[3].toUint(), sigs));
require(merkleHash.checkMembership(txindex, root, proof));
```

The first thing we do is take the `keccak256` hash of `txBytes`. `txHash` gives us enough information to validate that the signatures on the transaction are valid. Validation is handled in [Validate.sol](https://github.com/omisego/plasma-mvp/blob/master/plasma/root_chain/contracts/Libraries/Validate.sol). 

Next we hash `txHash` again with `ByteUtils.slice(sigs, 0, 130) = sig1 + sig2`. This hash represents a leaf node in our Merkle tree and means we can verify that the transaction was actually included in the `blknum` (derived from `utxoPos`). We do this with `require(merkleHash.checkMembership(txindex, root, proof));`, where `proof` is a compact Merkle inclusion proof.

Priority calculation is up next. The idea here is that older outputs should generally be processed before newer outputs. A detailed explanation of *why* we need priority is described [here](https://hackmd.io/s/BJZdignFf). This is a definite area for improvement because [priority is a lot more complicated than it seems](https://github.com/omisego/plasma-mvp/issues/29).

```
// Priority is a given utxos position in the exit priority queue
uint256 priority;
if (blknum < weekOldBlock) {
    priority = (utxoPos / blknum).mul(weekOldBlock);
} else {
    priority = utxoPos;
}
```

Here we're actually calculating the exit's priority. If the output being referenced is more than a week old (determined by `weekOldBlock`), then `blknum` is replaced by `weekOldBlock`. This actually creates a collision where two transactions might have the same priority. A potential fix for this collision is proposed [here](https://github.com/omisego/plasma-mvp/issues/29).

Lastly, we make some checks and insert the new exit:

```
require(exitIds[utxoPos] == 0);
exitIds[utxoPos] = priority;
exitsQueue.insert(priority);
exits[priority] = exit({
    owner: txList[6 + 2 * oindex].toAddress(),
    amount: txList[7 + 2 * oindex].toUint(),
    utxoPos: utxoPos
});
Exit(msg.sender, utxoPos);
```

`require(exitIds[utxoPos] == 0);` just ensures that we aren't making the same exit twice. Then, to be able to access an exit given its unique identifier (`utxoPos`), we map `utxoPos` to `priority` in `exitIds`. Finally, we insert the exit into our list of exits and emit an `Exit` event. 

#### `challengeExit`

Users need to be able to challenge exits of double spends. The general process for this is:

1. Check that the challenge tx is included in the referenced block and that it has correct signatures.
2. Check that the exiting UTXO is included as an input to the challenge tx OR that the challenge tx comes before the exiting UTXO and both have a common input.

The first part looks a lot like the checks we made in `startExit`. The second part simply verifies that the UTXO being exited was actually spent in the transaction we're providing. Note: the current implementation as shown below [is actually broken](https://github.com/omisego/plasma-mvp/issues/63).


```
...

    // @dev Allows anyone to challenge an exiting transaction by submitting proof of a double spend on the child chain
    // @param cUtxoPos The position of the challenging utxo
    // @param eUtxoPos The position of the exiting utxo
    // @param txBytes The challenging transaction in bytes RLP form
    // @param proof Proof of inclusion for the transaction used to challenge
    // @param sigs Signatures for the transaction used to challenge
    // @param confirmationSig The confirmation signature for the transaction used to challenge
    function challengeExit(uint256 cUtxoPos, uint256 eUtxoPos, bytes txBytes, bytes proof, bytes sigs, bytes confirmationSig)
        public
    {
        uint256 txindex = (cUtxoPos % 1000000000) / 10000;
        bytes32 root = childChain[cUtxoPos / 1000000000].root;
        uint256 priority = exitIds[eUtxoPos];
        var txHash = keccak256(txBytes);
        var confirmationHash = keccak256(txHash, root);
        var merkleHash = keccak256(txHash, sigs);
        address owner = exits[priority].owner;

        require(owner == ECRecovery.recover(confirmationHash, confirmationSig));
        require(merkleHash.checkMembership(txindex, root, proof));
        delete exits[priority];
        delete exitIds[eUtxoPos];
    }

...
```

`challengeExit` takes six parameters. 

`cUtxoPos` is the position of one of the two UTXOs inside the conflicting transaction. This is probably a misleading parameter, because you actually only need to provide proof of a transaction and not a specific input. We just use this to figure out the block and txindex of the challenging tx. 

`eUtxoPos` is the `utxoPos` of the exit being challenged.

`txBytes` is the RLP encoded transaction that's being used to challenge this exit. 

`proof` is a Merkle proof that the transaction described by `txBytes` was actually included at the specified block and txindex.

`sigs` are the signatures for the challenging transaction.

`confirmationSig` is a signature showing that the owner of the exit has seen this transaction.

I'm going to describe this one in more detail once the issues described [here](https://github.com/omisego/plasma-mvp/issues/63) are fixed.

#### `finalizeExits`

[Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) describes a "passive loop" that finalizes exits that are more than two weeks old. The current implementation is actually also broken because it's verifying that the *output*, and not the exit of that output, is more than two weeks old.

```
...

    // @dev Loops through the priority queue of exits, settling the ones whose challenge
    // @dev challenge period has ended
    function finalizeExits()
        public
        incrementOldBlocks
        returns (uint256)
    {
        uint256 twoWeekOldTimestamp = block.timestamp.sub(2 weeks);
        exit memory currentExit = exits[exitsQueue.getMin()];
        uint256 blknum = currentExit.utxoPos.div(1000000000);
        while (childChain[blknum].created_at < twoWeekOldTimestamp && exitsQueue.currentSize() > 0) {
            currentExit.owner.transfer(currentExit.amount);
            uint256 priority = exitsQueue.delMin();
            delete exits[priority];
            delete exitIds[currentExit.utxoPos];
            currentExit = exits[exitsQueue.getMin()];
        }
    }

...
```

Before we do anything we `incrementOldBlocks`.

Next we calculate what time it was 2 weeks ago based on the current block's timestamp:

```
uint256 twoWeekOldTimestamp = block.timestamp.sub(2 weeks);
```

Then we figure out what the current exit to be processed is based on our priority queue:

```
exit memory currentExit = exits[exitsQueue.getMin()];
```

**Note:** Here's where things are broken. We *should* be calculating the age of the exit, but we're instead checking the age of the *output*:

```
uint256 blknum = currentExit.utxoPos.div(1000000000);
while (childChain[blknum].created_at < twoWeekOldTimestamp && exitsQueue.currentSize() > 0) {
```

If that check passes, then we send the owner of the exit their money and delete their exit:

```
currentExit.owner.transfer(currentExit.amount);
uint256 priority = exitsQueue.delMin();
delete exits[priority];
delete exitIds[currentExit.utxoPos];
```

**Note:** There's another bug here! We aren't stopping people from attempting to exit twice. We should really be maintaining a list of UTXOs that have already exited.

Finally, we continue the loop with the next exit to be processed

```
currentExit = exits[exitsQueue.getMin()];
```

### Constant Functions

Lastly, we have some constant functions.

#### `getChildChain`

Returns the block with a specific `blockNumber`.

```
...

    function getChildChain(uint256 blockNumber)
        public
        view
        returns (bytes32, uint256)
    {
        return (childChain[blockNumber].root, childChain[blockNumber].created_at);
    }

...
```

#### `getExit`

Returns the exit with a specific `priority`.

```
...

    function getExit(uint256 priority)
        public
        view
        returns (address, uint256, uint256)
    {
        return (exits[priority].owner, exits[priority].amount, exits[priority].utxoPos);
    }
```

