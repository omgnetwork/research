# More Viable Plasma

## Introduction

[Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) (“Plasma MVP”) describes a simple specification for a UTXO-based Plasma chain. A key component of the Plasma MVP design is a protocol for “exits,” whereby a user may withdraw back to the root chain any funds available to them on the Plasma chain. The protocol presented in the specification requires users sign two signatures for every transaction. Concerns over the poor user experience presented by this protocol motivated the search for an alternative exit protocol. 

In this document, we describe More Viable Plasma (“MoreVP”), a modification to the Plasma MVP exit protocol that improves user experience. The MoreVP exit protocol ensures the security of assets for clients correctly following the protocol. We initially present the most basic form of the MoreVP protocol and provide intuition towards its correctness.  We formalize certain requirements for Plasma MVP exit protocols and provide a proof that this protocol satisfies these requirements.

An optimized version of the protocol is presented to account for restrictions of the Ethereum Virtual Machine. This optimization effectively caches the results of gas-heavy operations to decrease cost to the end user. A proof of the reduction from the optimized protocol to the basic protocol is provided to demonstrate that this optimized design also satisfies our formalized requirements. 

We note the existence of certain attack vectors in the protocol, but find that most of these vectors can be largely mitigated and isolated to a relatively small attack surface. These attack vectors and their mitigations are described in detail. Although we conclude that the design is safe under certain reasonable assumptions about user behavior, some points are highlighted and earmarked for future consideration.

Overall, the MoreVP exit protocol is a significant improvement over the original Plasma MVP exit protocol. We can further combine several optimizations to enhance user experience and reduce costs for users. Future work will focus on decreasing the implementation complexity of the design and continuing to minimize gas usage. 


## Exit Protocol

In this section, we specify the MoreVP exit protocol and give an intuitive argument toward its correctness. A formal treatment of the protocol is presented in the appendix. 


### Definitions

#### Deposit Transaction

A deposit transaction is a special type of transaction that creates a new balance on the Plasma chain. This is similar to a Bitcoin "coinbase" transaction. In practice, this takes the form of a transaction with a single “special” input.


#### Spend Transaction

A spend transaction is any transaction that spends a UTXO present on the Plasma chain. Unlike a deposit transaction, a spend transaction does not create any new balance.


#### In-flight Transaction

A transaction is considered to be “in-flight” if it has been broadcast but has not yet been included in the Plasma chain. 


#### Canonical Transaction

A transaction is “canonical” if none of its inputs were previously spent in any other transaction. The definition of “previously spent” depends on whether or not the transaction in question is included in the Plasma chain.

If the transaction was included in the chain, an input to that transaction would be considered previously spent if another transaction also spending the input was included in the chain *before* the transaction in question. Transaction position is determined by the tuple (block number, transaction index). If the transaction was not included in the chain, an input to that transaction would be considered previously spent if another transaction also spending the input is *known to exist*.

Note that in this second case it’s unimportant whether or not the other transaction is included in the chain. If the other transaction is included in the chain, then the other transaction is clearly included before the transaction in question. If the other transaction is not included in the chain, then we can’t tell which transaction “came first” and therefore simply say that neither is canonical. 


#### Exitable Transaction

If a transaction is “exitable,” then a user may attempt to start an exit that references the transaction. All deposit transactions are “exitable” by default. A spend transaction can be called “exitable” if the transaction is correctly formed (e.g. more input value than output value, inputs older than outputs, proper structure) and is properly signed by the owners of the transaction’s inputs. 


#### Valid Transaction

A deposit transaction is “valid” if and only if it corresponds exactly to a single deposit on the root chain and is the first deposit transaction to correspond to that deposit. A spend transaction is “valid” if and only if it is exitable, canonical, and only stems from valid transactions (i.e. all transactions in the history are also valid transactions). Note that a transaction would therefore be considered invalid if even a single invalid transaction is present in its history. An exitable transaction is not necessarily a valid transaction.


### Basic Specification

The MoreVP exit protocol allows the owners of both inputs and outputs to a transaction to attempt an exit. 

The owner of an input `in` to a transaction `tx` must prove that:
1. `tx` is exitable.
2. `tx` is non-canonical.
3. `in` is not spent in any transaction other than `tx`.

The owner of an output `out` to a transaction `tx` must prove that:
1. `tx` is exitable.
2. `tx` is canonical.
3. `out` is not spent.

Exits are then ordered by the position of the *youngest input* to the transaction referenced in each exit. Proof XXX in the appendix shows that this ordering ensures that all outputs created by valid transactions will be paid out before any output created by an invalid transaction. 


## Exit Protocol Implementation

### Motivation

The MoreVP exit protocol described above guarantees that correctly behaving users will always be able to withdraw any funds they hold on the Plasma chain. However, we avoided describing how the users actually prove the statements they’re required to prove. This section presents a specification for an implementation of the exit protocol. The implementation is designed to be deployed to Ethereum and, as a result, some details of this specification take into account limitations of the EVM.

Additionally, the MoreVP exit protocol is not necessary in all cases. Previous work shows that we can use the Plasma MVP exit protocol without confirmation signatures for any transaction included before an invalid (or, in the case of withheld blocks, potentially invalid) transaction. We only need to make use of the MoreVP protocol for the set transactions that are in-flight when a Plasma chain becomes byzantine. 

The MoreVP protocol guarantees that if transaction is exitable then either the unspent inputs or unspent outputs can be withdrawn. Whether the inputs or outputs can be withdrawn depends on if the transaction is canonical. However, in the particular situation in which MoreVP exits are required, users may not be aware that an in-flight transaction is actually non-canonical. This can occur if the owner of an input to an in-flight transaction is malicious and has signed a second transaction spending the same input. 

To account for this problem, we allow exits to be ambiguous about canonicity. Users can start MoreVP exits with the *assumption* that the referenced transaction is canonical. Other owners of inputs or outputs to the transaction can then join (or “piggyback”) the exit. We add cryptoeconomic mechanisms that determine whether the transaction is canonical and which of the inputs or outputs are unspent. The end result is that we can correctly determine which inputs or outputs should be paid out.  


### MoreVP Exit Protocol Specification

#### Timeline

The MoreVP exit protocol requires the use of a “challenge-response” mechanism, whereby users can submit a challenge but are subject to a response that invalidates the challenge. To give users enough time to respond to a challenge, the exit process is split into two “periods.” When challenges are subject to a response, we require that the challenges be submitted before the end of the first exit period and that responses be submitted before the end of the second. We define each period to have a length of half the minimum exit period (MEP). Currently, the MEP is set to 7 days, so each period has a length of 3.5 days. 


#### Starting the Exit

Any user may initiate an exit by presenting a transaction and proving that the transaction is exitable. The user must submit a bond, `exit bond`, for starting this action. This bond is later used to cover the cost for other users to publish statements about the canonicity of the transaction in question.

We provide several possible mechanisms that allow a user to prove a transaction is exitable. First, remember that deposit transactions are exitable by default. Two ways in which spend transactions can be proven exitable are as follows:
1. The user may present the transaction along with each of the transactions that created the inputs to the transaction, a Merkle proof of inclusion for each of these transactions, and a signature from each input owner. The contract can then validate that these transactions are the correct ones, that they were included in the chain, that the signatures are correct, and that the exiting transaction is correctly formed. This proves the exitability of the transaction.
2. The user may present the transaction along signatures the user claims to be valid. The contract can validate that the exiting transaction is correctly formed. Another user can challenge one of these signatures by presenting some transaction that created an input such that the true input owner did not sign the signature. In this case, the exit would be blocked entirely and the challenging user would receive `exit bond`.

Option (1) checks that a transaction is exitable when the exit is started. This has lower communication cost and complexity but higher up-front gas cost. This option also ensures that only a single exit on any given transaction can exist at any point in time.
Option (2) allows a user to assert that a transaction is exitable, but leaves the proof to a challenge-response game. This is cheaper up-front but adds complexity. This option must permit multiple exits on the same transaction, as some exits may provide invalid signatures.

These are not the only possible mechanisms that prove a spend transaction is exitable. There may be further ways to optimize these two options.


#### Proving Canonicity

Whether unspent inputs or unspent outputs are paid out in an exit depends on the canonicity of the referenced transaction. Unfortunately it’s too expensive to directly prove that a transaction is or is not canonical. Instead, we assume that the referenced transaction is canonical by default and allow a series of challenges and responses to determine the true canonicity of the transaction.

The process of determining canonicity involves a challenge-response game. In the first period of the exit, any user may reveal a conflicting transaction that potentially makes the exiting transaction non-canonical. This conflicting transaction must be exitable, must share an input with the exiting transaction, and must be included in the Plasma chain. Multiple conflicting transactions can be revealed during this period, but only the oldest presented transaction is considered for the purposes of a response.

If any transactions have been presented during the first period, any other user can respond to the challenge by proving that the exiting transaction is actually included in the chain before the oldest presented conflicting transaction. If this response is given, then the exiting transaction is determined to be canonical and the responder receives the `exit bond` placed by the user who started the exit. Otherwise, the exiting transaction is determined to be non-canonical and the challenger receives `exit bond`.

Note that this challenge means it’s possible for an honest user to lose `exit bond` as they might not be aware their transaction is non-canonical. We address this attack vector and several mitigations in detail later.


#### Joining an Exit

As noted earlier, it’s possible that some participants in a transaction may not be aware that the transaction is non-canonical. Owners of both inputs and outputs to a transaction may want to start an exit in the case that they would receive the funds from the exit. However, we want to avoid the gas cost of repeatedly publishing and proving statements about the same transaction. We therefore allow owners of inputs or outputs to a transaction to join (“piggyback”) an existing exit that references the transaction. This effectively “caches” the result of the “exitable” and “canonical” checks. 

Users must join an exit within the first period. To join an exit, an input or output owner places a bond, `piggyback bond`. This bond is used to cover the cost of challenges that show the input or output is spent. A successful challenge blocks the specified input or output from exiting. These challenges must be presented before the end of the second period. 

Note that it isn’t mandatory to piggyback an exit. Users who choose not to join an exit are choosing not to attempt a withdrawal of their funds. If the chain is byzantine, not piggybacking could potentially mean loss of funds.


### Attack Vectors and Mitigations

#### Honest Exit Bond Slashing

It’s possible for an honest user to start an exit and have their exit bond slashed. This can occur if one of the inputs to a transaction is malicious and signs a second transaction spending the same input. 

The following scenario demonstrates this attack: 
1. Mallory spends `UTXO1` in `TX1` to Bob, creating `UTXO2`.
2. `TX1` is in-flight.
3. Operator begins withholding blocks while `TX1` is still in-flight.
4. Bob starts an exit referencing `TX1` and places `exit bond`.
5. Mallory spends `UTXO1` in `TX2`.
6. In period 1 of the exit for `TX1`, Mallory challenges the canonicity of `TX1` by revealing `TX2`.
7. No one is able to respond to the challenge in period 2, so `TX1` is determined to be non-canonical.
8. After period 2, Mallory receives Bob’s `exit bond`.

Mallory has therefore caused Bob to lose `exit bond`, even though Bob was acting honestly. We want to mitigate the impact of this attack as much as possible so that this does not prevent users from receiving funds. 
 

##### Mitigations

###### Bond Sharing

One way to partially mitigate this attack is for each user who piggybacks to cover some portion of `exit bond`. This cuts the per-person value of `exit bond` proportionally to the number of users who have piggybacked. 


###### Small Exit Bond

The result of the above attack is that users may not exit from an in-flight transaction if the gas cost of exiting plus the value of `exit bond` is greater than the value of their input or output. We can reduce the impact of this attack by minimizing the gas cost of exiting and the value of `exit bond`. Gas cost should be highly optimized in any case, so the value of `exit bond` is of more importance.

`exit bond` is necessary to incentivize challenges. However, we believe that challenges can be sufficiently incentivized if `exit bond` simply covers the gas cost of challenging. Observations from the Bitcoin and Ethereum ecosystems suggest that sufficiently many nodes will verify transactions without a direct in-protocol incentive to do so. Our system requires only a single node be properly incentivized to challenge, and it’s likely that many node operators will have strong external incentives. Modeling the “correct” size of the exit bond is an ongoing area of research.


###### Timeouts

We can add timeouts to each transaction (“must be included in the chain by block X”) to reduce number of vulnerable transactions at any point in time. This will probably also be necessary from a user experience point of view, as we don’t want users to accidentally sign a double-spend simply because the first transaction hasn’t been processed yet.


## Alice-Bob Scenarios

### Alice & Bob are honest and cooperating:

1. Alice spends `UTXO1` in `TX1` to Bob, creating `UTXO2`.
2. `TX1` is in-flight.
3. Operator begins withholding blocks while `TX1` is still in-flight. Neither Alice nor Bob know if the transaction has been included in a block.
4. Someone with access to `TX1` (Alice, Bob, or otherwise) starts an exit referencing `TX1` and places `exit bond`.
5. Alice and Bob piggyback onto the exit and each place `piggyback bond`.
6. Alice is honest, so she hasn’t spent `UTXO1` in any transaction other than `TX1`.
7. After period 2, Bob receives the value of `UTXO2`. All bonds are refunded.


### Mallory tries to exit a spent output:

1. Alice spends `UTXO1` in `TX1` to Mallory, creating `UTXO2`.
2. `TX1` is included in block `N`.
3. Mallory spends `UTXO2` in `TX2`.
4. Mallory starts an exit referencing `TX1` and places `exit bond`.
5. Mallory piggybacks onto the exit and places `piggyback bond`.
6. In period 2, someone reveals `TX2` spending `UTXO2`. This challenger receives Mallory’s `piggyback bond`.
7. Alice is honest, so she hasn’t spent `UTXO1` in any transaction other than `TX1`.
8. After period 2, Mallory’s `exit bond` is refunded.


### Mallory double spends her input:

1. Mallory spends `UTXO1` in `TX1` to Bob, creating `UTXO2`.
2. `TX1` is in-flight.
3. Operator begins withholding blocks while `TX1` is still in-flight. Neither Mallory nor Bob know if the transaction has been included in a block.
4. Mallory spends `UTXO1` in `TX2`. 
5. `TX2` is included in a withheld block. `TX1` is not included in a block.
6. Bob starts an exit referencing `TX1` and places `exit bond`.
7. Bob piggybacks onto the exit and places `piggyback bond`.
8. In period 1, someone challenges the canonicity of `TX1` by revealing `TX2`.
9. No one is able to respond to the challenge in period 2, so `TX1` is determined to be non-canonical.
10. After period 2, Bob’s `piggyback bond` is refunded. The challenger receives Bob’s `exit bond`.


### Mallory spends her input again later:

1. Mallory spends `UTXO1` in `TX1` to Bob, creating `UTXO2.
2. `TX1` is included in block `N`.
3. Mallory spends `UTXO1` in `TX2`.
4. `TX2` is included in block `N+M`
5. Mallory starts an exit referencing `TX1` and places `exit bond`.
6. In period 1, someone challenges the canonicity of `TX1` by revealing `TX2`.
7. In period 2, someone responds to the challenge by proving that `TX1` was included before `TX2`.
8. After period 2, the user who responded to the challenge receives Mallory’s `exit bond`.


### Mallory attempts 

1. Mallory spends `UTXO1` and `UTXO2` in `TX1.
2. Mallory spends `UTXO1` in `TX2`.
3. `TX1` and `TX2` are in-flight.
4. Mallory starts an exit referencing `TX1` and places `exit bond`.
5. Mallory starts an exit referencing `TX2` and places `exit bond`.
6. In period 1 of the exit for `TX1`, someone challenges the canonicity of `TX1` by revealing `TX2`.
7. In period 1 of the exit for `TX2`, someone challenges the canonicity of `TX2` by revealing `TX1`.
8. After period 2 of the exit for `TX1`, the challenger receives `exit bond`.
9. After period 2 of the exit for `TX2`, the challenger receives `exit bond.


### Operator tries to steal funds from an included transaction

1. Alice spends `UTXO1` in `TX1` to Bob, creating `UTXO2`.
2. `TX1` is included in (valid) block `N`.
3. Operator creates invalid deposit transaction `TX2`, creating `UTXO3`.
4. Operator spends `UTXO3` in `TX3`.
5. Operator starts an exit referencing `TX3` and places `exit bond`.
6. Operator piggybacks onto the exit and places `piggyback bond`.
7. Bob starts a *standard* exit for `UTXO2`.
8. Operator’s exit will process will have priority of position of `UTXO3`. Bob’s exit will have priority of position of `UTXO2`.
9. Bob’s exit processes first, Operator’s exit processes second. All bonds are refunded.


### Operator tries to steal funds from an in-flight transaction.

1. Alice spends `UTXO1` in `TX1` to Bob, creating `UTXO2`.
2. `TX1` is in-flight.
3. Operator creates invalid deposit transaction `TX2`, creating `UTXO3`.
4. Operator spends `UTXO3` in `TX3`.
5. Operator starts an exit referencing `TX3` and places `exit bond`.
6. Operator piggybacks onto the exit and places `piggyback bond`.
7. Bob starts an exit referencing `TX1` and places `exit bond`.
8. Bob piggybacks onto the exit and places `piggyback bond`.
9. Alice is honest, so she hasn’t spent `UTXO1` in any transaction other than `TX1`.
10. Operator’s exit will process will have priority of position of `UTXO3`. Bob’s exit will have priority of position of `UTXO1`.
11. Bob’s exit processes first, Operator’s exit processes second. All bonds are refunded.


## Appendix
### Formalization of Definitions
#### Transaction
#### Chain
#### Deposit Transaction
#### Spend Transaction
#### Canonical
### Requirements
#### Safety
### Basic Exit Game
#### Formalization
#### Proof of Requirements
### Exit Game Implementation
#### Formalization
#### Proof of Implementation
