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

1. Each tx add a field: `targetBlock`. This need to be the same as the child chain block that the tx is mined into.
1. If a tx is using an input in previous block, it would need confirmation signature of the input owner signing-off previous block.
1. If the tx input is from same block (chained within a block), do not need the confirmation signature of input block (it will be the same block anyway).
1. To exit an output or to use an output in next tx which is in different `targetBlock`, the user needs to disclose the confirmation signature that sign-off the whole block (signing block hash). So in any case, the proof information to finalize the txs within the block would be disclosed at once (promising atomic).
