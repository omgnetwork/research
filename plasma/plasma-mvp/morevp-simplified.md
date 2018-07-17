# More Viable Plasma

## TL;DR

This document provides a largely simplified specification for More Viable Plasma, a version of [Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) (“Plasma MVP”) that removes the need for confirmation signatures.

## Background

Transactions in Plasma MVP are processed in time-wise order.
The “exit priority” of any UTXO is determined by the block in which the transaction was included, then the index of the transaction within that block, and finally the index of the output in the list of transaction outputs (“output-age priority”).
Plasma MVP requires the use of confirmation signatures to prevent the operator from including a valid transaction time-wise after an invalid transaction.
Without confirmation signatures, the Plasma MVP exit priority mechanism would process the invalid exit before the valid exit. 

Confirmation signatures are generally bad user experience.
Users must sign a transaction, wait for it to be included in a block, and sign a second signature attesting that the transaction was included in a valid block.
Although clients with “hot” wallets can quickly (and usually automatically) sign these secondary signatures, clients with “cold” wallets will face an annoying user-flow (e.g. clicking some button on a hardware wallet).  

Confirmation signatures must also be published to the Plasma chain to avoid certain attack vectors.
This significantly reduces the amount of block space available for transactions. 

This motivates the search for an alternative exit game that removes the need for confirmation signatures and simultaneously ensures all valid UTXOs can be processed before all invalid UTXOs. 

## More Viable Plasma

More Viable Plasma (“MoreVP”) proposes an exit game that removes the need for confirmation signatures and ensures correct transaction processing order.

### Basic Exit Game

Under this new protocol, we allow owners of both the *inputs or outputs* to any transaction to attempt an exit. 

If the owner of an input `in` to a transaction `tx` wants to exit, the owner must prove that:
1. `tx` is correctly formed and properly signed.
2. One of the inputs to `tx` has already been spent in some transaction other than `tx`.
3. `in` has not been spent in any transaction other than `tx`.

If the owner of an output `out` of a transaction `tx` wants to exit, the owner must prove that:
1. `tx` is correctly formed and properly signed.
2. None of the inputs to `tx` have already been spent in some transaction other than `tx`.
3. `out` has not been spent.

The first requirement in both games ensures that the transaction is correctly formed (input amounts are greater than or equal to outputs amounts, referencing valid inputs) and signatures over the inputs come from their correct respective owners.

The second requirement has something to do with a concept we call transaction “canonicity.”
Simply defined, a transaction is “canonical” if none of the transaction’s inputs have been previously spent.
Let's suppose there exist two transactions `tx` and `other_tx` such that `tx` and `other_tx` share at least one input.
If `tx` *is* included in the Plasma chain, then `tx` is not canonical if `other_tx` is included before `tx` (in an earlier block or in the same block and an earlier transaction index).
If `tx` is *not* included in the Plasma chain, then `tx` is not canonical if `other_tx` can be proven to exist.
In this sense, the shared input is being double-spent in `tx` because it was already used in `other_tx`. 
`tx` is canonical if a transaction `other_tx` that would make `tx` not canonical by the above rules does not exist.

We say that a transaction is *fully valid* if it's correctly formed, properly signed, and canonical.

If a transaction is correctly formed and signed, but the transaction *is not* canonical, then, as shown above, some input to that transaction must have been double-spent.
However, it may be the case that not *all* of the inputs to the transaction are double-spent - other than this transaction, these inputs are unspent.
In this case, we should allow the owners of these inputs to retrieve their funds.
The third requirement filters out any inputs that have been double-spent. 

If a transaction is correctly formed and signed, and the transaction *is* canonical, then this is a fully valid transaction.
In this case, the outputs to this transaction should receive the resulting funds, as is expected.
Here, the third requirement then simply ensures that only *unspent* outputs will be able to withdraw funds from the contract.

This very basic exit game simply ensures that the “true” owners of money within any transaction will be able to exit.

### Priority

Note, however, that the above exit game fails to provide safety if exit priority is still determined by the position of the transaction input or output in the Plasma chain (“output-age priority”). 

To illustrate, imagine the following scenario:

1. Alice sends transaction `tx` to Bob.
2. The operator creates a block with some "bad" (violates a consensus rule) transaction `inv_tx` and includes `tx` after `inv_tx`.
3. Bob starts the exit game with `tx`.
4. The operator starts the exit game with `inv_tx`.

Note: The operator's "bad" transaction is deliberately vague.
The specific reason why this transaction is "bad" can vary for any number of reasons.
It could be that this transaction is a double spend, that it creates money on the Plasma chain without a corresponding deposit, or that it violates any other consensus rule.
The important thing here is that transactions that don't violate consensus rules should be able to exit before transactions that do. 

Bob and the operator can both prove that their respective transactions (`tx`, `inv_tx`) are properly formed and signed.
Similarly, both can prove that none of the inputs to `tx` and `inv_tx` have been spent in some transaction other than `tx` and `inv_tx`, respectively.
Finally, both can prove that they haven't spent their outputs.

Now we have a situation where both exits can be processed. 
Obviously, we don’t want the operator’s exit to be processed before Bob’s exit.
This will unfortunately be the case if we determine priority based on the age of the output.
To solve this problem, we change introduce a slight change to how exit priority is handled.

In the MoreVP exit game, within an exit on a transaction `t`, we calculate the priority every input or output of `t` based on the position (block number, transaction index, output index) of the youngest input to `t` (“youngest-input priority”).

Now we can very easily see that, as long as Alice’s input was included in the Plasma chain before the "bad" transaction, Bob’s exit will be processed first.
It’s pretty easy to prove that this holds for *any* transaction, as long as all of a transaction’s inputs were included in the chain before the "bad" transaction.

Note that if Alice's input was included *after* the "bad" transaction and Alice spent that input, the inputs/outputs to this transaction can no longer be safely exited.
In practice, this simply means that clients should never spend a UTXO if they haven’t validated the entire chain up (and including) the transaction in which that UTXO was created. 
