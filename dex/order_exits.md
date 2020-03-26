# Order Exits: non-custodial continuous trading DEX

Based on a lot of prior art by the rest of the team, among others:
  - exit upgrades
  - some initial thoughts on partially signed txs
  - MoreVP
  - all flavors of restricted custody DEXes (and friends)
  - other more recent DEX designs

### Situation

`alice` wants to buy some `token` for a `utxo1` she has (of `other_token`). Conversely `bob` wants to buy some `other_token` for `utxo2` (in `token`).

Both decide to be executing with the executor `elena`

### Order placement

Alice places the order by submitting this partially signed transaction to Elena:

```
tx_incomplete:
  inputs: [utxo1, _]
  outputs: [{alice, token, _, amount_not_less_than} = output1_incomplete, {_, other_token, amount} = output2_incomplete]
  executor: elena
  ...
  [ecsign(alice_priv_key, {utxo1, output1_incomplete, output2_incomplete, elena}) = alice_sig]
```

(this isn't submitted to the child chain yet. Elena will now look for the matching order.)

Bob places his order:

```
tx_incomplete_other:
  inputs: [utxo2, _]
  outputs: [{bob, other_token, _, other_amount_not_less_than} = ... the rest analogically
  executor: elena
  ...
  [...bob signed similar to alice above = bob_sig]
```

### Settlement

Elena, having these two pieces of signed data, can now combine them into this:

```
tx_complete_filled:
  inputs: [utxo1, utxo2]
  outputs: [{alice, token, amount, amount_not_less_than} = output1, {bob, other_token, amount, other_amount_not_less_than} = output2]
  executor: elena
  ...
  [alice_sig, bob_sig, ecsign(elena_priv_key, {inputs, outputs})]
```

and submit to the child chain. Partial fills aside, this should be exitable from normally by Alice and Bob respectively.
Correctness is checked by the child chain, but all the data is contained in `tx_complete_filled`.
Only the signing scheme is peculiar, since Alice, Bob and Elena all signed various pieces of this transaction.

If all is OK, the journey ends here.

### Cancellation

To cancel on a happy path, this is submitted by Alice:

```
tx_complete_canceled:
  inputs: [utxo1, _]
  outputs: [{alice, token, _, amount_not_less_than} = output1_canceled, {alice, other_token, amount} = output2_canceled]
  executor: elena
  ...
  [alice_sig, ecsign(alice_priv_key, {inputs, outputs})]
```

this is submitted to the child chain. If this happens, then the `output2_canceled` is exitable from normally by Alice.
After that, Elena would not be allowed to submit `tx_complete_filled` to the child chain.

### Unhappy paths ;_;

What needs to be looked at now, is what can go wrong. I think this is the list:
  - Alice needs to exit `utxo1`, no matter the availability of Elena or the child chain server
  - Alice cannot undo any valid settlement by exiting `utxo1` or any other action by herself
  - Alice must recover her funds in case `tx_complete_canceled` is caught in-flight
  - Alice must recover her funds in case `tx_complete_filled` is caught in-flight
  - the state of `utxo1` and any orders and spends that use it as inputs must be reconciled, to have the contract hold correct end result state
  - Alice shouldn't be able to double-order on 2 executors
  - invalid trades must be void and not jeopardize anyones funds

Enter order exits (OE).
But first, let's set the stage by laying out what is the status of `utxo1`

#### `utxo1` standard-exitability

`utxo1` can standard exit unless `tx_incomplete` are presented to challenge.
**NOTE** that `tx_incomplete` can be extracted out from `tx_complete_filled` or `tx_complete_canceled`.

So until Alice broadcast `tx_incomplete`, she exits `utxo1` with a standard exit normally.
After she broadcasts `tx_incomplete` she cannot as she would be challenged.

#### Order Exits

If Alice broadcast `tx_incomplete`, to exit `utxo1` she starts an order exit (OE) in the contract by presenting her `tx_complete_canceled`.

From now on, Alice's order can be filled in the root chain contract.
This would be something like promoting (upgrading) the `tx_complete_canceled` to `tx_complete_filled`.

The idea here is similar to IFE. Alice starts off with her optimistic "state of knowledge" and we reconcile the state using an interactive game.

Behavior of an order exit:
1. OE requires one to prove inclusion and ownership of `utxo1`, similar to SE
2. Starting an OE marks `utxo1` as exiting and should not allow any another OE to start from `utxo1`
3. If no fill is presented, the order will exit canceled (to Alice)
4. If `tx_complete_filled` is presented the order will exit filled (to Bob/Alice)
    1. filling can only be done by a transaction signed by Elena; this action is somewhat similar to `startIFE`, requires posting an IFE bond
    2. fills undergo a process similar to IFE - only the most canonical fill will really fill an order
    3. fills check the soundness wrt. the orders, that is, the settlement satisfies the intent to trade
    4. there can be many fills per OE, in case Elena double-filled. They all should resolve like normal IFEs
    5. any fill submitted must pre-mark "the other orders", in case `startOE` is used on their inputs, like `startOE(utxo2)` from Bob could happen later on
    6. as with normal IFEs, piggybacking will apply in a slightly modified form - it must work "globally" not per IFE. So Bob would piggyback `utxo2` if Bob knows he didn't place _an order other than `tx_incomplete_other` for that `utxo2`_. It would be "good" for any fill which mentions `utxo2` globally. It would give Bob's `utxo2` back only if all the fills `utxo2` took part in turned out non-canonical (which is somewhat similar to cancellation of Bob's order). If one of Bob's fills is canonical, Bob will get his money from that fill's output (but not `utxo2` anymore, which he sold :joy:)

So to summarize and rephrase a bit:
  - Alice and Bob attest to placing an order and placing it only once per their inputs `utxo1` and `utxo2` respectively
  - Elena attests to filling every order once
  - Operator provides blocks, which if available, decide on the oldest included cancellation/fill to reconcile the state
  - Order exits should work for `utxo1` also when there is byzantine behavior from any other actor, be it Elena or Operator.

Let's look now at the unhappy paths laid out above:

#### Alice needs to exit `utxo1`

_Alice needs to exit `utxo1`, no matter the availability of Elena or the child chain server._

Assume Alice did broadcast her `tx_incomplete`.

1. Alice sees a problem with the child chain, she has 1 week to exit her monies
2. Alice starts an OE from `utxo1`
3. Alice waits until Elena presents a fill, if she does, Alice piggybacks `utxo1` **and** any of the trade's outputs that belong to her
4. Canonicity game unfolds, depending on that Alice gets `utxo1` or any result of her trade out.
5. (in case there's no fill, Alice's order is effectively "in-flight canceled")

The priority is a tiny bit tricky here:
 - if canceled, the priority to exit is that of `utxo1`,
 - if filled the priority needs to be calculated separately for Alice and Bob, based on their age of `utxo1` and `utxo2`.

The reason for this being the case is that Alice and Bob cannot "ruin the other trader's" priority.
Imagine a situation, where Alice's old and secure (priority-wise) `utxo1` crossed with negligent Bob's fresh `utxo2`.
Picking the youngest input priority IFE-style would jeopardize Alices funds.

We can do such a stunt because the settlement is essentially an atomic swap between a pair of actors and you can link output ownership with input ownership.

#### Alice cannot undo any valid settlement

_Alice cannot undo any valid settlement by exiting `utxo1` by herself or any other action._

If Alice spends `utxo1` in a normal transaction, it can be included in the child chain only before the settlement.
Alice should however not do it, since it could potentially invalidate her piggyback of `utxo1` in case of an order exit.

If Alice starts a standard exit from `utxo1`, this will be challenged by Elena using `tx_incomplete` (or anyone else, if the settlements / cancellations were published)

If Alice starts an IFE touching `utxo1` (and piggybacks), this will be challenged just the same.

#### `tx_complete_canceled` in-flight

_Alice must recover her funds in case `tx_complete_canceled` is caught in-flight_

See [Alice needs to exit `utxo1`](#alice-needs-to-exit-utxo1).

#### `tx_complete_filled` is caught in-flight

_Alice must recover her funds in case `tx_complete_filled` is caught in-flight_

See [Alice needs to exit `utxo1`](#alice-needs-to-exit-utxo1).

#### State reconciliation

_The state of `utxo1` and any orders and spends that use it as inputs must be reconciled, to have the contract hold correct end result state_

This is probably the toughest part, where we should argue, that no matter the condition of the child chain and no matter the amount and flavors of exits being started in the root chain contract, there is a way to reconcile the state and exit from a valid state.

Precise argument pending (**TODO**), but let's hand-wave one instead: This is just MoreVP and IFEs with the modification that we can do an "exit upgrade" (following terminology from prior art), from a `utxo1`-based order exit to an IFE. The "exit upgrade" is done by filling the exiting order.

#### No double-order

_Alice shouldn't be able to double-order on 2 executors_

Problem arises, if you think about Alice sending `tx_incomplete` over to Elena and Elvis, two distinct executors.

This would make reconciliation problematic and also, even if reconciliation is not necessary, it is unfair to the executors.

We solve this by tainting `utxo1` and `utxo2`.
The mechanism of tainting is similar to private/custodial addresses from previous designs, but simpler:

`utxo1.outputGuard == hash(alice, elena, nonce)`.
Wherever providing `ecsign(alice_priv_key, something)` was required, it becomes `{ecsign(alice_priv_key, something), elena, nonce}`.
Such output guard doesn't impact ownership or control over `utxo1`.
Compared to a normal `utxo1.outputGuard == alice` it only makes supplying `elena` and `nonce` necessary to spend `utxo1` (satisfy `spendingCondition`).
The only consequence of the `elena` being there is that only `elena` can validly fill this order.
`nonce` is there to not have the tainting give away the intent to trade.

For simplicity, we're not including this detail in the rest of the write-up, assuming that there's only one executor `elena`.

#### No invalid trades and hacked/dishonest exchanges

_Invalid trades must be void and not jeopardize anyones funds_

If an invalid settlement (invalid `tx_complete_filled`, where funds move without satisfying the orders) is submitted to an honest child chain server it is rejected.

If it is included (by a byzantine child chain server), it is treated like an invalid Plasma transaction - everyone exits with higher priority. Block withholding doesn't hurt.

It cannot be presented as a challenge to a standard exit from `utxo1`.

It cannot impact an order exit from `utxo1` (cannot do the "upgrading fill").

It cannot challenge or make non-canonical any valid fill.
