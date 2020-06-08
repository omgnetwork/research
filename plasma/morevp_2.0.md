## MoreVP 2.0

For settlement transactions we introduce a special kind of in-flight exits. Let's call them "chained in-flight exits", cIFE short.

Assumptions:
  - settlement txs use `output_id` as input pointers
  - we're in Restricted Custody world, so in case child chain goes rogue, exchange doesn't

The gist of the cIFE protocol is that we treat a chain of settlement txs as one lump super-transaction `super_tx`, but we "compose" this `super_tx` in stages.
This `super_tx` is going to be treated as a single state transition, just "composed" iteratively of chained txs.
The process of a cIFE is about determining to what extent this state transition "happened or not", using the MoreVP IFE's canonicity game for the chained txs.
The exit priority is that of the youngest input for all the txs chained in the `super_tx`.

To illustrate the flow it's easiest to focus on a concrete chain of two settlements `tx1`, `tx2` submitted by exchange `E`, on behalf of traders `A`, `B`, `C`.

[**A hideous sketch of the outputs and transactions here**](https://docs.google.com/drawings/d/1lfjI6EdjGBxCtvlbEq69AlGCTwC9BwOCLD2ICURuL9k/edit)

1. Exchange `E` starts the IFE: `E.IFE(tx1)`
2. Alice can piggyback on her output `o4` out of that. Note this is a "regular" output and she can't sign spending of `o4` or she will be challenged MoreVP IFE-style.
3. Exchange is not finished with the `super_tx` at hand, so it chains the next tx to that: `E.chain(tx1, 1, tx2)`, indicating that 1st output `o5` of `tx1` has been consumed by a follow-up `tx2`.
(Think of this as a "mixture" of a `startIFE` and `piggyback_output` message.)
Observe that `tx2` consumes `o3` and `o5` as inputs: if this were a `startIFE` we'd require `E` to deliver inclusion proof of txs creating those `o3`, `o5`, but it is **only necessary** for `o3` - `o5` is chained instead of proven
4. Here Bob and Charlie do what Alice did in (2/)

If there were more txs to be chained `tx3`, `tx4`, ... the process would iterate naturally.

`super_tx` here is `[tx1, tx2]`, it is a state transition from `o1`, `o2`, `o3` to `o4`, `o6`, `o7`.

### Canonicity game and the `super_tx`

#### General flow

The "happened or not" of the `super_tx` is dependent on the canonicity of the individual chained txs.
If both `tx1` and `tx2` are canonical, the `super_tx` happened in its entirety (`o4`, `o6`, `o7` should exit)
If `tx2` turns out to be non-canonical, the `super_tx` happened only as far as `tx1` goes (`o3`, `o4`, `o5` should exit)
If `tx1` turns out to be non-canonical, the `super_tx` didn't happen at all (`o1-3` should exit).

#### Canonicity index

One might think that this would require the contract to iterate over the chain of exits to change the status of the `super_tx`, as individual canonicities of the chained txs change.
This is not the case - the exiting data can be represented in a way that allows this to be known without iterating.
This is done by introducing the concept of `canonicity_index`.
The "happened or not" of the `super_tx` is not represented by a boolean (canonical/non-canonical), but by an integer `n` meaning, `n` chained txs are "canonical".
If there's an `n+1`th transaction then it is the first non-canonical in the chain.

Using `canonicity_index`, one can organize the exiting funds of the `super_tx` so that on `processExits`, the cIFE "knows" which inputs and outputs to allow out.

#### Requirements on competitors

To ensure that the canonicity game makes sense, we are after the following property.
There must be only 2 ways a chain of txs may have been attempted to be executed:
 - by publishing by `E` in a cIFE
 - by including in child chain blocks

If `E` has double-signed some settlements, the version included in valid child chain blocks prevails and is published.
If `E` didn't double-sign, the single chain of txs is the one in the cIFEs, will be canonical.

In both cases the users have access to the funds.

##### Proof for competitors required

An important modification of the canonicity game we introduce for the chained txs is that submitting a competitor **requires an inclusion proof**.

##### Unique `output_id` cIFE-ing

In order to root out the possibility of two txs from two cIFEs being competitors, we need to keep a contract-global register of cIFE-ing `output_ids`.
This means that `o1-7`, as described uniquely by their `output_ids`, cannot be used in any other cIFE, ever.

But this is good thanks to the Restricted Custody assumption and the following property:
All inputs are in control of a single actor `E`, so non-canonicity caused by co-transacting with a fraudster is not an issue.
Probably it should even be blocked, so only settlements from `E` can play along in a cIFE.
This means that there's only ever going to be one "way" to chain txs into an cIFE-ing `super_tx`.
In the example flow given, this means in practice that `E` cannot start another cIFE starting this time from `tx2`, that would be bad!

This simplifies considerations about canonicity greatly - competitors to the chained txs in a cIFE can only be found in the child chain blocks.

If there's a competitor to a cIFE, it must've been provided to the child chain by a fraudulent `E`, meaning that (by RC assumption), child chain works fine.
Now, if the child chain works fine, it means that if the chained txs in the cIFE are proven to be non-canonical, the canonical ones are sitting in the correctly propagated valid child chain blocks - users' funds are safe.
(**TODO** if this is too weak, one can drop the assumption and strengthen the protocol by evicting the non-canonical chained txs in a cIFE).

### Piggybacks of the `super_tx`

Piggybacks have a couple of differences compared to MoreVP IFEs.

Firstly, piggybacks are considered "jointly" for the chaining outputs like `o5` - Bob piggybacks this means that he's piggybacking an output of `tx1` and input of `tx2`.
This is the same "money" so this makes sense.
`o5` is then piggybacked with the same meaning, regardless of whether `tx2` is chained/disclosed by `E` or not.

In particular, when Bob piggybacks `o5` before `tx2` is disclosed, Bob's bond is not slashed.

### Remarks

A couple of important notes:
- **exit priority** - the age of inputs needs to be declared ahead in the step (1/), so it's actually `E.IFE(tx1, inputs_not_younger_than_x)` and priority is based on `x` (in our example flow - that of `o3`).
During the process of chaining, there can be no inputs "prepended" to the `super_tx` (what happened to `o3`) that are younger than `x`.
In other words, `E` provides an _a priori_ upper bound on exit priority.
This is because we cannot easily go about updating the priority queue. If not for `x` declared, we'd been stuck with priority of `o2`, which is the youngest input at time when `E.IFE(tx1...` is done
- **transaction balance** - i.e. the `sum(inputs)>=sum(outputs)` is really tracked for the whole tx - if all chaining txs satisfy it, so does the entire `super_tx`
- **unmatched funds on the exchange** - in a variant of the flow, when the chain of txs ends with `(E, D)` holding funds, `D` piggybacks that and `D` will get the payout (see piggybacking)
- **gas** - cIFE should incur no more gas costs than IFEing all the chained txs, so if we're fine with gas costs of IFE for payment, we should be too with costs of cIFE for settlement (**TODO** - verify this, this is intuition)

Why?
- in general - simplifies the happy-path, putting the "hard stuff" on the unhappy path - the same reason that made limbo-exit/cancel & MoreVP protocols worthwhile
- MVP puts us effectively outside the UTXO model - we need some additional conditions that are unverifiable with only UTXO data
- MVP makes txs enter a shady area in terms of finality - when do we consider a tx done and when we don't?
In other words, MVP allows to manipulate the doneness of a tx by one of the parties (sender can withhold the confirm sig or receiver can claim lack thereof), and as such is hard to reason about.
- MVP is further away from MoreVP than MoreVP 2.0 is, with cIFE we're not really introducing anything dramatically new
- MVP introduces a P2P communication requirement - confirm sig must make its way to the trader when the trade is settled
- cIFEs go beyond settlement transactions - if we are careful about the various assumptions here (to be identified - **TODO**), we can have chainable, payment txs which is awesome
