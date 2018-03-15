Thinking about "Plasma Cash"
===

**Note:** this post and other similar writings represent unfinished or generally rambling thoughts. It's my opinion that organized train-of-thought documents can help other researchers in a multitude of ways. Certain things in this document might be completely wrong! I wanted to make sure that readers could see my thought process in *how* I came to certain (possibly mistaken!) conclusions. 

---

On March 3rd, 2018, Vitalik published [this](https://ethresear.ch/t/plasma-with-much-less-per-user-data-checking/1298/2) post on ethresearch. The post details a scheme that Vitalik later dubbed "Plasma Cash." Vitalik also gave a detailed presentation about the idea at EthCC, available via YouTube [here](https://www.youtube.com/watch?v=uyuA11PDDHE). 

## A problem

Plasma MVP currently requires users store and check every transaction submitted by every user on the chain. This is potentially a huge amount of data! So although Plasma MVP can scale Ethereum already, this data storage requirement is pretty significant and might become an unwanted barrier to entry. 

## A solution

From a high level, Plasma Cash is a version of Plasma that tackles this data storage/checking problem by taking advantage of some convenient properties of identifiable, indivisble, non-mergable tokens. Unlike deposits on Plasma MVP, which can be broken up into smaller pieces or merged together into larger ones, deposits on Plasma Cash cannot be divided or merged.

Each invididual deposit on Plasma Cash is assigned a unique "coin ID." This is stored on the root chain as a mapping from coin IDs to denominations. This is implemented as something like:

```
mapping (uint32 => uint256) coins;
```

One specific coin ID might be worth 1 ETH, while another might be worth 5 ETH. However, you can't join the two to make one coin worth 6 ETH or split them to make three coins worth 2 ETH. It's effectively like transacting with physical money, where we can only transact in specific discrete denominations (unless we make change!). Hence, Plasma *Cash*.

## So what?

Instead of requiring users to check *every* transaction, Plasma Cash only requires users check:

1. The full transaction history of the specific coin(s) being received.
2. Proof that the specific coin(s) weren't spent in any blocks not mentioned in (1).

This is cool because it means users suddenly have to check *much* less data in order to make a transaction. 

The exact process of validating a transaction is actually a little bit more complex, and generally looks like this:

1. Check that the coin ID being sent is valid and has the correct denomination (check the root chain mapping).
2. Check that the transaction history is valid (does it correctly stem from the deposit that created this coin ID?).
3. Check that the coin isn't referenced in any other block not mentioned in the full transaction history. If our chain is 5 blocks long and the tx history references blocks 1, 2, and 4, then we need to make sure the coin wasn't spent in blocks 3 or 5. 

Once we check these, we have a (relatively) compact proof that *our specific* coin is valid and hasn't been double spent anywhere. We don't have to know anything about any other coins! 
 
## Cool, what's the catch?

Plasma Cash currently only really works if we're using individible, non-mergable coins. If I want to send someone 7 ETH, but I only have a coin worth 10 ETH, then I'd need to coordinate a transaction where I simultaneously send a coin worth 10 ETH and receive back a coin worth 3 ETH. This is kind of like getting physical change. Just like with physical change, if the person who I'm sending a 10 ETH coin to doesn't have a 3 ETH coin to send back, I'm out of luck.

## Change providers

You might be able to mitigate the above scenario to some extent with a construction called a "change provider." A change provider is basically someone who always has a lot of coins of different denominations available. The area in between the seat cushions of your couch would probably qualify. 

To illustrate, let's go back to our example. Alice is trying to send 7 ETH to Bob, but she only has a 10 ETH coin and some small change. Bob doesn't have any coins. This time, Alice is going to use a change provider to make this transaction work. Carol, a change provider, has lots of coins of different denominations and happens to have both a 3 ETH coin and a 7 ETH coin. Alice and Carol form a transaction with the following components:

1. A 10 ETH coin sent from Alice to Carol
2. A 3 ETH coin sent from Carol to Alice
3. A 7 ETH coin sent from Carol to Bob
4. Some small coin sent from Alice to Carol as a fee

## Fees

Unfortunately, that last component of the above transaction is a little complex because it means Alice needs to keep some small change around in order to pay for change provider fees. This is kind of like keeping quarters in your car to pay for parking meters. 

This is a good time to talk about fees. Plasma Cash operators should probably receive fees for their services. However, it's not as easy to implement fees on Plasma Cash like as it is on Plasma MVP. A few possible ways to take fees have been suggested.

### Fees as small coins

The first and most "Plasma Cash"-like way is to simply require users to always have tokens with small values that are used to pay fees. This is the most intuitive mechanism, but it has a lot of potential UX issues. For example, we don't want users to end up in a situation where it costs more to withdraw a token than the token is worth. We also don't want the transaction fees on Plasma to be limited by gas prices on Ethereum! 

We could potentially mitigate this by enabling splits on Plasma Cash through something like decimal places in the coin ID. We'll talk about this later.

### Fees as a cut from coins

Another way to implement fees is by specifying a `total fee` attached to each coin. An operator will only accept a transaction if the new total fee specified is greater than the old total fee. When a user attempts to exit from a coin, they can only exit the initial value of the coin minus the latest total fee. This works, but it has the unfortunate side effect of ruining nice denominations. 

### Fees from a "fee balance"

Fees could also be separated from Plasma Cash entirely. One way to do this is via a "fee balance". This would be a separate balance on the root chain that a user needs to maintain in order to transact on the network. When the user makes a transaction, they'll specify a `total fee` just like in the example above. The transaction will only be accepted if the operator sees that the user has enough balance and that `total fee` is greater than the `total fee` specified in the user's last transaction. This works because the operator already needs to see every transaction. 

The user can attempt to withdraw their fee balance, but an operator will challenge if they can prove that there's a transaction with some total fee greater than what the user claims. Transactions would have to contain a nonce so that users can specify an ordering to their transactions. Operators would then first order transactions by nonce before validating `total fee`. Otherwise, transactions might be thrown out for containing an invalid `total fee` if the users makes more than one transaction in the same block. 

## Merges/splits

There's a distinct possibility that either no change provider exists or no change provider has the correct change for your transaction to occur. In these cases, you'll have to figure out another way for your transaction to complete. 

### Plasma Cash split transactions

It might be possible to enable coin splitting without ever touching the root chain. One way to do this is via decimals. This basically means that when I make a deposit, my coin ID always ends with some number of 0s (e.g. if I'm using 3 decimal places, `XX...XX000`). We can use these 0s to split our coins on the Plasma chain and uniquely identify/value the split off coins.

This exact construction took me a while to come up with and it might still be wrong. We generally want to make sure that it's easy to quickly verify that a coin has never been spent (or split) somewhere else. This means making sure that my coin is unique, and that no one else can legitimately hold the same coin ID with the same (or different) value. If I want to split my coin, then I'd use some sort of special transaction that takes a coin and "burns" the input coin to create new outputs. "Burning" here just means that we're permanently changing the value of that specific coin. 

Let's demonstrate this by example. I want to create a few coins from my `XX...XX000` coin. My `XX...XX000` coin (with value 100%) could be burned in exchange for two new coins, `XX...XX000` and `XX...XX005` (with values 0.5% and 99.5%, respectively). The value of these coins is specified in the transaction fields. The 3 digits of the first new coin will be the same as the 3 of the original coin. The 3 of the second new coin will be the 3 of the original + the value of the first coin. Using this method, it's impossible for two users to have valid coins with the same ID. 

Now let's assume this splitting process occurs a few more times until I have a coin `XX...XX000` worth 0.1% of the deposit. I now want to send this coin and prove that I haven't double spent anything. I send a full transaction history and proofs of non-inclusion for `XX...XX000`. This transaction history necessarily includes each split transaction that I've made from `XX...XX000`. The receiver would be able to see if I tried to hide a split from them.

Therefore the user can be sure that `XX...XX000` that I'm sending does indeed have the value that I've specified, and that no one else also has an `XX...XX000` with any other value. If the user then wants to exit from `XX...XX000`, they'll simply attempt to exit and specify the % they claim to own. The exit can be challenged if it's invalid or if someone else can prove that they have a later iteration of the coin with a different value. 

The downside to this construction is that there's only a maximum of `n` possible coins that can be broken off, where `n` is the number of decimals specified. There's probably an equivalent construction for merging these coins back together, but I haven't really put enough thought into it.

### Root Chain merge/split transactions

David Knott came up with an excellent idea for root chain "merge/split" transactions. These transactions would occur on the Ethereum contract, just like deposits or exits. A merge/split transaction is basically self descriptive - someone who owns a specific coin can merge (or split) some coin(s) into one or more entirely new coins with the same total value.

For example, if I have a 10 ETH coin with ID `0` and a 5 ETH coin with ID `1`, I could create a new 15 ETH coin with ID `2` as long as I invalidate coins `0` and `1`. I could also choose to split the 10 ETH coin into five 2 ETH coins. I can make any combination of coins, as long as the total value of the input coins is equal to the total value of the output coins. 

This transaction needs to happen on the root chain because we need to update our mapping between coin IDs and denominations to reflect the new coins. We also want to make sure that only the real owner of the coins can merge/split them. If we didn't confirm this, users could grieve the system by merging or splitting other people's coins against their will.

We can confirm the real owner of the coins by holding a challenge period similar to that of the Plasma MVP exit period. During this time, a user puts down a bond and declares intent to merge or split a specific coin output or set of coin outputs that the user owns. Other users can then challenge that merge/split by proving that the outputs are either invalid or already spent somewhere else. 

## Plasma MVP Cash?

Plasma Cash is extremely cool, but it'd be even *cooler* if we could find a way to achieve the same result (less per-user data checking) with coins that can be merged or split. In theory, this would give us all of the benefits of Plasma Cash without the downsides involved with needing to make exact change. 

No one has figured this out (as of the writing of this post), and it's not entirely apparent that this is even possible. For the sake of research, I'll go ahead and list a few of the rabbit holes I've been down and the walls I eventually ran into. Maybe someone else can spot something that I didn't!

**Note:** these lines of thinking didn't lead to any useful end goal. However, I think it's useful for people to read about *failed* research just as they'd read about successful research. Successful research is often the result of many dead ends. Understanding why a construction doesn't work can be useful for understanding why similar constructions don't work. You might also find something I didnt, or know of a concept I didn't know that could be used to solve these problems!

### Simply checking transaction inputs

Like coin IDs, transaction inputs are unique. If we can prove that a transaction input is valid and hasn't been spent in any other block, then we've accomplished our goal.

Unfortunately, this isn't as easy (or useful) as it sounds. We can't simply check that the inputs into our current transaction haven't been included in any other block because those inputs might not be valid. To verify that the inputs are valid, we would need to verify that the inputs to those inputs are also valid.

This is check is feasible when we're talking about unique, indivisible, non-mergable coins because the transaction history can only ever have as many transactions as the block has blocks. However, if we can merge and divide inputs, then we could easily have many more transactions to check. For example, if each transaction can have at least two inputs and two outputs, then the tree of inputs to check could potentially grow to encompass every input ever. At this point we're basically passing around an entire blockchain worth of data for every transaction. This obviously isn't scalable. 

### Giant Merkle trees, maybe?

After the naive construction failed, I started to think about ways to remove the requirement that users check every single input. I figured that if I could find a way to mark inputs as spent as soon as they're included in a block, then I could remove the need to check proofs of non-inclusion for each input.

Unfortunately, even solving this issue ignores the fact that I'd still need to validate every single transaction in the (potentially) massive history tree. So attempting to accomplish the gains of Plasma Cash without non-fungible coins is hard because verifying the transaction history is a core aspect of the security guarantees. The transaction history is bounded by the length of the Plasma chain in Plasma Cash, but it's bounded by the length of the Plasma chain times the number of transactions that can fit in a block in Plasma MVP!

Anyway, I figured I'd think about this in case it had some useful outcomes for Plasma Cash. Maybe it's possible to design a system where proof of non-inclusion is handled in a more compact way. My intuition was that if I could create a *giant* sparse Merkle tree on chain where the leaf nodes represent *every* possible UTXO the Plasma chain could ever produce, then I could create a (relatively) compact proof that a certain input has never been spent. 

This massive Merkle tree would have to be updated whenever an input is spent, but it's not economical for each user to do so individually. Instead, the Plasma operator would update the tree whenever they submit a new block to the root chain. In order to make this work, we need to make sure that the operator is correctly updating the tree. This means two things:

1. The operator has updated the tree with every input spent in this block
2. The operator has not included any inputs in this block that already exist in the tree

Or (a little) more formally:

1. The set of non-null inputs of the old Merkle tree and the set of spent inputs of the new block are disjoint
2. The set of non-null inputs of the new Merkle tree is a superset of both the sets in (1)

This is easy to prove if we have the entire block, because we could update the old tree ourselves and cross check the results. However, the whole point of Plasma Cash is that we don't want users to have to receive the entire block. 

Here's where I got stuck. I couldn't figure out a way of proving that given the Merkle root of the tree and the Merkle root of the inputs to the block, that the new Merkle tree satisfied the above property. Maybe something with zkSNARKS? I'm not sure.

