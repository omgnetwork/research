Bloom Filters in Plasma Cash
============================

Author: Kelvin Fichter

---

## Acknowledgements

Thank you to [Dan Robinson](https://twitter.com/danrobinson?lang=en) from Chain for the original [comment](https://ethresear.ch/t/plasma-cash-plasma-with-much-less-per-user-data-checking/1298/3) that inspired this post.

## Background

### Plasma Cash

Plasma Cash is a novel Plasma design that eliminates the need for every user to validate every other user's transactions. The basic idea of this design is that users can only transact *unique* coins - kind of like spending a *specific* $20 bill. Just like a merchant might check that your $20 bill is legitimate, recipients on a Plasma Cash chain will verify that your unique coin is legitimate. 

The original Plasma Cash [specification](https://ethresear.ch/t/plasma-cash-plasma-with-much-less-per-user-data-checking/1298) describes a two-part process for verifying coins. First, a user requests the coin's full transaction history, including proof that the coin was actually spent in the given blocks. Then, the user requests proof that the coin was *not* spent in any other block.

### Bloom Filters

A Bloom filter is a probabilistic data structure that can test whether something is a member of a set or not. Bloom filters are designed so that false positives are possible, but false negatives are not. If the Bloom filter says that the word "hello" *isn't* in the set of inputted words, then "hello" is definitely not in the list. However, if a filter says that "world" *is* in the set of inputted words, there's a chance that it might not be.

[Dan Robinson](https://twitter.com/danrobinson?lang=en) first mentioned the idea of using Bloom filters to decrease the required Plasma Cash proof size in a [comment](https://ethresear.ch/t/plasma-cash-plasma-with-much-less-per-user-data-checking/1298/3) on ethresear.ch. 

## Methodology

### Quick Pitch

Dan's comment described an easy way to drastically reduce the Plasma Cash proof size. The idea is simple: instead of sending out lots of proofs that coins haven't been spent each block, the operator simply sends out a Bloom filter populated with every transaction in the block. Whenever a user is verifying a coin's history, they'll just check their cached Bloom filters. If the filter returns a false positive for a specific block, then the user will request a proof that the coin hasn't been spent in that block. 

### Correctness

Plasma Cash requires a few small modifications to ensure that the Bloom filter design works correctly. Let's discuss some of the potential ways things can go wrong and solutions to those problems:

#### Operator includes a transaction in a block but not in the Bloom filter

This situation would mean that there *is* a valid transaction in a block, but that the receiving user wouldn't request that transaction. In this case, the user should act as if that transaction never existed. A recipient of a coin should verify that the coin is included in a block *and* that the coin is included in the block's Bloom filter.

It's possible that the operator doesn't include the transaction in the Bloom filter, but that the transaction appears as a false positive. In this case, it's as if the operator included the transaction in the filter and there's nothing to worry about.

#### Operator includes a transaction in a Bloom filter when there is no transaction in the block

This is equivalent to a false positive. The recipient would see that the transaction is in the Bloom filter and request a proof that the transaction is not in the block.

#### Operator gives different Bloom filters to different users

The idea behind this attack is better described with an example:

1. Mallory is the operator on a Plasma Cash chain.
2. Mallory sends a coin to Bob in block #5.
3. Mallory wants to cheat Alice by double spending the coin.
4. Mallory sends Alice a Bloom filter for block #5 that doesn't include the coin.
5. Alice knows that Bloom filters can't have false negatives, so she doesn't request a proof for block #5.
6. Alice accepts Mallory's coin.

Obviously, this makes double spending pretty easy. This problem really stems from the fact that Bloom filters are quite large (order of a few KB). **It isn't economical to publish the whole filter to the root chain**. Instead, this problem can be solved if the operator is required to publish the keccak256 *hash* of the Bloom filter to the root chain. A user would then receive a Bloom filter and verify that the hash of the received filter matches the published hash. The user will only trust the filter if the hashes match.

The above situation now works out as follows:

1. Mallory sends a coin to Bob in block #5.
2. Mallory publishes $hash(filter)$ to the root chain.
3. Mallory sends Alice a Bloom filter for block #5 that doesn't include the coin.
4. Alice checks that $hash(received filter) = hash(filter)$ and finds that they differ. 
5. Alice doesn't accept Mallory's transaction.

### Size Considerations

The main problem with Bloom filters is that they grow in size with the number of elements in the set. As stated above, this means that it's too expensive to publish the whole filter to the root chain. We want to make sure that we're not in effect *increasing* the Plasma Cash proof size. Let's do some math to ensure that this isn't the case.

Plasma Cash currently requires a Merkle proof for each block in the chain. The size of a Merkle proof depends what hash function we're using and how large the tree is. Let's make some assumptions in order to calculate the data requirements. It's likely that many implementations will use keccak256 (a 256 bit hash function) to calculate Merkle roots. If we want to support 1,000,000 unique coins, then we'll need a Merkle tree 20 layers deep ($2^{20} = 1048576$).

Every Merkle proof requires log(n) hashes + the data in the transaction. Let's ignore the size of the transaction for now. The size of the required proof is therefore:

$$
S_{merkle proof} = log_{2}(2^{20}) * 256 bits = 20 * 256 bits = 0.64 KB
$$

Note that this is the size of the proof per coin, per block!

Now let's figure out how large the Bloom filters would have to be. Bloom filters scale linearly with the number of items in the filter, so we'll have to estimate the number of transactions per Plasma block. Assume that we're creating a new Plasma block once per every Ethereum block. If we want a ~10x order of magnitude increase in transactions per second over Ethereum, we'll need to include approximately 5000 transactions in every block.

A bloom filter with 5000 elements and a 1/100000 false positive rate is [14.62 KB](https://krisives.github.io/bloom-calculator/). That's pretty big - much bigger than the Merkle proof size. Is it worth it? The answer is, "it depends."

The Bloom filter for each block is ~23 times bigger than the Merkle proof. This means that if users are making lots of transactions with lots of different coins, the Bloom filter might actually be more efficient than the Merkle proofs. Bloom filters also have a great user experience. Users don't have to download large blocks of data and can use the same Bloom filter to verify the validity of all received transactions. The low false positive rate makes the Merkle proofs basically irrelevant. 

### Summary

Bloom filters can be used to simplify the proofs required by Plasma Cash at the cost of increased storage in the average case. Bloom filters work particularly well when a Plasma Cash chain has relatively low TPS. The element-linear size of each filter makes the scheme difficult to scale. 

At the same time, Bloom filters vastly improve the user experience of Plasma Cash - users aren't required to send entire transaction histories and proofs of non-inclusion in the form of huge chunks of data. A Bloom filter-based Plasma Cash implementation doesn't sacrifice any security guarantees.

## Research Topics

- Can we use [Cuckoo filters](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf) instead? 
- In what cases might Plasma Cash + Bloom filters be practically better than sticking to Merkle proofs?
