Exploring `plasma-mvp`: Exit Priority
===

Author: Kelvin Fichter

---

Plasma's exit priority a necessary mechanism to ensure that exits can be processed safely and effectively. This document explains why it's necessary and how `plasma-mvp` attempts to implement it. 

## Example

Let's illustrate why priority is important by showing what could happen if we didn't have it:

1. Alice makes valid transactions on the Plasma chain and ends up with a UTXO worth 1 ETH.
2. The operator creates an invalid block and does not publish it. This block contains a transaction that creates a 1 ETH UTXO for the operator "out of nowhere" (like a deposit).
3. The operator immediately exits on this UTXO. The exit can't be challenged because it's valid (as far as Plasma is concerned).
4. Alice later attempts to exit on her 1 ETH UTXO, but the contract doesn't have enough ETH to pay her because the operator took 1 ETH "out of nowhere".

Here's how that situation plays out with priority:

1. Alice makes valid transactions on the Plasma chain and ends up with a UTXO worth 1 ETH.
2. The operator creates an invalid block and does not publish it. This block contains a transaction that creates a 1 ETH UTXO for the operator "out of nowhere" (like a deposit).
3. The operator immediately exits on this UTXO. The exit can't be challenged because it's valid (as far as Plasma is concerned).
4. Alice later attempts to exit on her 1 ETH UTXO. Her exit is given a higher priority than the operator's because the corresponding UTXO was included in an earlier block. Alice's exit will be processed before the operator's exit.
5. The operator's exit fails as long as all other (valid) exits are created before the operator's exit is processed.

## Implementation

So how is this actually implemented?

[Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) specifies that `priority` be implemented as follows:

> startExit must arrange exits into a priority queue structure, where priority is normally the tuple (blknum, txindex, oindex) (alternatively, blknum * 1000000000 + txindex * 10000 + oindex). However, if when calling exit, the block that the UTXO was created in is more than 7 days old, then the blknum of the oldest Plasma block that is less than 7 days old is used instead. There is a passive loop that finalizes exits that are more than 14 days old, always processing exits in order of priority (earlier to later).
> 
> This mechanism ensures that ordinarily, exits from earlier UTXOs are processed before exits from older UTXOs, and particularly, if an attacker makes a invalid block containing bad UTXOs, the holders of all earlier UTXOs will be able to exit before the attacker. The 7 day minimum ensures that even for very old UTXOs, there is ample time to challenge them.

So we generally need to ensure that exits are ordered first by block, then transaction index, and then output index. `plasma-mvp` uses `blknum * 1000000000 + txindex * 10000 + oindex` (also called `utxoPos`) as priority. If the output is more than a week old, `blknum` is replaced by `weekOldBlock`, the oldest block less than a week old. 

The code for that looks like this:

```
// Priority is a given utxos position in the exit priority queue
uint256 priority;
if (blknum < weekOldBlock) {
    priority = (utxoPos / blknum).mul(weekOldBlock);
} else {
    priority = utxoPos;
}
```

Now we just need to make sure that exits are placed in a queue and ordered by priority. `plasma-mvp` uses `priority` in a mapping between `priority` and `exits`, so `priority` is necessarily unique. This might eventually be changed so that `priority` doesn't need to be unique but the priority queue somehow holds some other identifying information about the exit. There's an open issue on GitHub about this [here](https://github.com/omisego/plasma-mvp/issues/29).

