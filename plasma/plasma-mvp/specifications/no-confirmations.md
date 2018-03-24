Plasma w/o Confirmations
===

Authors: David Knott, Kelvin Fichter

---

**Note:** This is a cleaned up version of David's original "Plasma w/o Confirmations" specification. The original version of this document can be found [here](https://hackmd.io/s/BkgnUZALf). 

---

## Background

[Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) specifies that a transaction be "confirmed" before the transaction is considered valid. Transaction confirmations are necessary to avoid problems created by block withholding. A confirmation proves that a user has accesss to the block in which their transaction was included. 

However, confirmations are a little annoying because a user needs to sign a transaction, wait for their transaction to be included in a block, and then sign *another* confirmaton transaction. You can probably see how this could quickly become problematic.

This document specifies a version of Plasma that does *not* require confirmations and therefore only requires a single user signature to submit a transaction. Some implementation details are provided. As always, this construction has its own pros and cons and is subject to change as it's refined.

## Overview

The high-level summary of this specification is that transaction confirmations are removed by requiring that transactions be included within some number of Plasma blocks. Transactions that have "timed out" are considered invalid and should not be included in future Plasma blocks. An additional root-chain construction ensures that transactions are still safe if they're included but withheld.

## Why Confirmations Matter

Before we describe what Plasma might look like without confirmations, its important to understand why it currently has them. Remember that exits on Plasma MVP must be submitted within 7 days of a chain becoming byzantine in order to be considered safe.

### Block Withholding

One of the main attacks on a Plasma chain is a block withholding attack. This basically means that the Plasma operator either publishes a block root and fails to publish the actual block contents or stops publishing blocks entirely. Block withholding attacks have several consequences, and confirmations are used to mitigate a few of them.

Let's think about what might happen if we didn't have transaction confirmations. Consider the case where a user makes a transaction, the operator includes this transaction in a block, and then block is then withheld. The user will want to exit and retrieve their funds, but they have no clue if the transation they just sent was included in the block or not. The user can try to exit from the UTXO they just spent, but there's a chance that the operator challenges the exit. If happens then the original recipient has a chance to exit from the now-published transaction. However, the operator has 14 days to challenge the exit and could simply wait more that 7 days (the safety point) before challenging.

Confirmations completely mitigate this problem. A user in the situation from above would be able to safely exit from their original UTXO because a transaction is only considered valid once it's been confirmed. Any solution that gets rid of confirmations needs to make sure this issue isn't reintroduced.

### Late Transactions

Confirmations also mean that a user can choose which of their published transactions are considered valid. Imagine a user submits a transaction at `t=0`, but the operator doesn't include it in a block immediately. A long time later (maybe `t=10000`), the operator includes the transaction in a block. However, the user no longer wants that transaction to occur because they've already completed that transaction some other way. The user can simply refuse to confirm the transaction.

A construction without confirmations needs to give users some similar guarantees. We'll talk about a few different ways to potentially accomplish this, although some are more convenient than others.

## Problems with Confirmations

This list is not complete, but it describes some of the more obvious ways that confirmations make things difficult. 

### Fees

Confirmations could potentially result in grieving if we aren't careful. We want to incentivise users against flooding the network with transactions that they don't confirm. This could be accomplished by requiring that users pay a transaction fee for every transaction, even if the transaction isn't confirmed. 

### User Experience

Confirmations require a second signature. This is generally a bad user experience. Users need to sign the transaction, wait for it to be included, and then sign a confirmation. The waiting period between the two signatures could be significant depending on Plasma network load. Users will need to be available for both signatures and transactions will fail if users lose connection before they're able to send a confirmation.

## David's Implementation

This section describes David Knott's original no-conf specification.

### Rules

We specify a few rules to make this construction as simple as possible:

1. Transactions must be included within `n` Plasma blocks.
2. Transaction inputs must be at least `n + 1` Plasma blocks old.

Rule #1 prohibits a Plasma operator from including a transaction a long time (`>n` blocks) after it's published. This wouldn't matter if we were using confirmations because a user could just refuse to confirm a very late transaction. 

Rule #2 ensures that an invalid exit can't have higher priority than a valid exit (as long as users exit in a timely manner) while minimizing the amount of necessary on-chain work. We'll get back to this later.

### Modifications to Plasma MVP

We need to modify the Plasma MVP specification in order to implement these rules.

#### Change #1: The root chain needs to validate a transaction *and* its inputs before it can be exited

This is necessary to ensure that we can't break Rule #2 in any useful way. Remember that Rule #2 requires that inputs be at least `n + 1` Plasma blocks old. The following example demonstrates why this change is necessary:

1. Alice submits a transaction to Bob.
2. The operator includes Alice's transaction after including a few invalid transactions. The operator reveals Alice's transaction (so Bob can exit), but doesn't reveal their own (so no one can prove those are invalid).
3. The operator attempts to exit on their invalid transactions.
4. Bob attempts to exit on his valid transaction.
5. The operator's exit has higher priority than Bob's transaction, the operator withdraws first.

This isn't a problem if we have confirmations - Alice could simply refuse to confirm her transaction. We need to get more creative if we remove confirmations.

If we require that inputs be at least `n + 1` blocks old, then anyone who stops submitting transactions once the chain is byzantine can safely exit. An operator attempting to create a UTXO from "out of thin air" inputs would have to create at least `n + 1` invalid blocks. 

This breaks down if a user submits a transaction after the chain becomes byzantine. A client should *never* submit a transaction if they haven't received a block by the time the next block should've been published.

#### Change #2: Exits must be challenged within 4 days

This way, a withheld transaction can be safely withdrawn before an attacker completes any invalid exits. 

Plasma MVP mitigates block withholding attacks by requiring users to submit confirmations. If we remove confirmations, then a transaction can be included and be considered valid even if the block is withheld. This means that an operator could challenge an exit with a withheld transaction. Change #2 attempts to solve this problem.

To better understand why this change is necessary, consider the following potential scenario.

1. Alice submits a transaction to Bob.
2. The operator includes Alice's transaction but does not publish the block. Alice has no way to tell if her transaction has been included or not.
3. Alice attempts to exit from a UTXO that she tried to spend in her transaction to Bob.
4. The operator submits exits for some invalid UTXOs that drain the entire contract. 
5. After 7 days, the operator challenges Alice's exit with her transaction to Bob. 
6. Bob now has the information he needs in order to exit from Alice's transaction. Bob submits an exit.
7. The operator's invalid exits process before Bob's exit because Bob didn't submit his exit within 7 days of the chain becoming byzantine. The contract is now empty.
8. Bob cannot withdraw once his exits process because the operator has already empties the contract. 

Consider the same scenario once we apply Change #2. If the operator only has 4 days to challenge Alice's exit, then Bob has 3 days to safely submit his exit. The operator's exits will be given a lower priority than Bob's exit, and Bob will complete his (valid) withdrawal.

#### Alternative to Change #2: Special exit transactions

Change #2 is somewhat unsatisfying because Alice *will* lose her exit deposit. We'd prefer to come up with some mechanism that avoids restricting the challenge period and allows Alice to keep her exit deposit.

Piotr Dobaczewski came up with another construction that addresses this case. In a nutshell, Piotr's idea is that Alice creates a special type of exit that allows Bob to claim the funds within a period of time. If Bob doesn't complete the exit, then Alice can claim the funds. This special "limbo" exit can be invalidated by demonstrating that the UTXO (from Alice to Bob) was already spent.

Piotr gave the following example of how this might work out:

1. Alice buys a pear from Bob.
2. The operator includes Alice's transaction but withholds the block. Alice's payment is now stuck in "limbo" because she doesn't know if her transaction was included or not.
3. Alice starts a "limbo exit" that references her transaction to Bob. 
4. Bob has 3 days to complete the exit. 

One of two things can happen at this point:

1. Bob completes the exit.
2. Bob does not complete the exit. Alice completes the exit after 3 days.

In either case, this exit can be challenged if a user demonstrates that Alice's UTXO to Bob is spent. If no challenges are submitted, then the exit processes and the funds are withdrawn.

This is really cool, but a little complex. There's a potential for grieving if users are not actively watching for these type of transactions. For example, Alice could decide that she wants to cheat and get the money for her pear back. Assume that blocks are not being withheld and that Bob hasn't spent the UTXO from Alice's transaction. Alice could start the "limbo exit", which forces Bob to respond within 3 days. If Bob is offline for 3 days, then she might be able to steal Bob's funds. Bob's safest bet is therefore to immediately spend the funds to himself.

We can avoid the grieving case by modifying the construction slightly so that the limbo exit can *only* exit to Bob and *only* if Bob signs off on it. The idea behind this modification is that Alice and Bob will want to complete their transaction if the two parties are cooperating. If blocks are being withheld and the two parties are not cooperating, then Alice can attempt to exit normally. If the transaction is included (and therefore complete), then Alice will need to find some extra-protocol settlement anyway. It's reasonable to assume that Alice can ask for her deposit as part of this settlement. 
