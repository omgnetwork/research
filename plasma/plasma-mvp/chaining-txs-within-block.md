original: https://github.com/omisego/research/issues/86
### TL;DR

The idea here propose a solution to chain txs within a block under Plasma MVP protocol.
All txs within a block would be consider successful or failed atomically.

### Why
Trying to solve the problem of plasma tx cannot be chained within a block. 
Currently, all Plasma flavors needs to wait a block to be able to start next transaction.
This can be useful in several scenerios. For example, we can chain merge utxo txs together without waiting for a block.

### Concept

By design, to use a MVP tx, it would need the confirm signature of the tx from the sender. This is the proof that user double checked the result of the tx inside the block. This two step design gives user the control to finalize the tx after block submitted.

Leveraging the same design, we can make user finalize all txs within a block instead of finalizing a single tx. By finalizing the whole txs within a block, user can decide not to finalize anything within the block and exit from previous blocks.

This does not provide fast finality, but remove the issue of plasma tx cannot be chained together within a block.

### Design

1. If a tx is using a mined input in plasma chain, it would need confirmation signature of the input owner signing-off previous block.
1. If the tx input is in-flight, do not need the confirmation signature of input block.
1. To exit an output or to use an output in next tx which is in different, the user needs to disclose the confirmation signature that sign-off the whole block that mines the chained txs (signing block hash). So in any case, the proof information to finalize the txs within the block would be disclosed at once (promising atomic).

Be aware that it is possible for a user to chain A->B->C->D, but operator mines A->B->C only without D in the next block. In this case, the transaction data of D is useless now as C is mined and would need to regenerate D' including confirmation sign of the block of A->B->C.


### Extra Challenge for MVP
In MVP, there should be an extra challenge to protect user withhold confirmation signature. For a transaction A -> B, user can possible exit from A first without confirming B, and then confirm B to exit B too. Thus, we need an extra challenge game to protect this specific double-spending via exit scenerio.

We would need to same challenge for chained txs as well. It is to protect the following situation:
```
block1: [A]
block2 [B->C->D]
```
where A->B->C->D are chained together from same owner. Now the owner can choose to not confirm B->C->D first and exit A. Later, after the exit of A is finalized, confirm the block and exit from D. This creates a double spending situation via exit.

As a result, we need an extra challenge for this. I'd like to propose a challenge for "confirmation signature invalid". We challenge such confirmation signature is invalid by:

1. We found a previous finalized exit output
1. We prove there exists another tx in the same block of the confirm sig that is using the already-exitted output.

Now, whatever exit that is using this confirmation signature is invalid.

### Limitation of chaining

The whole chain must be in the same owner. We probably cannot chained payments: Alice sends to Bob and sends to Carol. Because, what if Carol confirms the block without Bob's confirmation ?
