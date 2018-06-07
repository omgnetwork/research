# Magic Withdrawals

## TL;DR

## Background

### Exit Priority

The [original Plasma MVP specification](https://ethresear.ch/t/minimal-viable-plasma/426) requires that exits be processed in priority order. Each exit references an unspent transaction output, and all exits are ordered by the age of the transaction output. This is generally necessary so that valid exits can be processed before invalid "out of nowhere" transactions. More information about exit priority can be found at the OmiseGO research repository [here](https://github.com/omisego/research/blob/master/plasma/plasma-mvp/explore/priority.md).

This post introduces a new type of exit priority, called "youngest-input" priority. Instead of ordering exits by the age of the output, we now order exits by the age of the youngest input to that output. This has the effect that transactions, even if they're included in withheld blocks after "out of nowhere" transactions, can still be exited as long as they only spend valid inputs. We provide a proof that this new construction is safe under full data availability, and then present a special exit game that makes the construction safe under data withholding. 

### Exit Game

Plasma MVP also introduces an exit game, whereby some outputs are deemed "exitable". This game correctly incentivizes some third party to challenge a non-exitable output. For example, spent outputs are considered non-exitable - the game requires the exitor to place a bond, such that anyone who knows the output to be spent is sufficiently incentivized to reveal the transaction and take the bond. 

The basic exit game has one challenge condition - outputs must not be spent. This condition is enough to make the exit game complete because of a construction of confirmation signatures. Basically, both parties to a transaction must "sign off" on a transaction once it's included in the chain. Honest clients won't sign off on withheld (and therefore possibly invalid, if the operator tries to cheat) transactions, so it's possible to say that honest clients won't receive funds that might be stolen by the operator (discussed more below).

### In-flight Transactions

Assuming no confirmation signatures, the exit game introduces trickiness in the event that a transaction has been signed and sent to the operator, who then witholds data. Because an operator might have included the transaction, an exit of the "in-flight" transaction's input can then be challenged by the (previously witheld) inclusion, forcing the user to exit from that transaction. Before our "youngest-input" construction, if the operator had included "out of nowhere" spends earlier in the chain, they could use them and drain the contract without challenge.

## Youngest-Input Priority

This proposal replaces output-age priority with something we call "youngest-input" priority. Basically, the exit priority of an output is now the tuple (`age of youngest input`, `age of output`). The priority of an output is first determined by the age of the youngest input to the transaction that created that output, then the age of the output itself. We claim that this construction is safe (honest clients cannot have money stolen) as long as clients don't spend any outputs received after any invalid (or possibly invalid, in the withholding case) transaction.

The gist of the following safety proof is that if there exists an "out of nowhere" input, then any transactions stemming from that input must have at most the priority of that input. Therefore, outputs only stemming from inputs created before the "out of nowhere" input must have a higher priority than any invalid output. 

### Safety Proof

Assuming full data availability, an exit game is simple to construct.  If an "out of nowhere" spend exit suddenly occurs, then any legitimate transactions' inputs must be older and can be exited first under the `age of youngest input`priority. Similarly if another spend of their transaction's inputs attempts an exit, they can present theirs, which must fulfill the `age of output` priority. Formally:

TX is the transaction space, taking the form (inputs, outputs), where inputs and outputs are integers representing their indices in the Plasma chain:

$$
TX: ((I_{1}, I_{2}, \ldots, I_{n}), (O_{1}, O_{2}, \ldots, O_{m}))
$$

For all $t \in TX$, we define the input function $I(t)=(I_{1}, I_{2}, \ldots, I_{n})$ and the output function $O(t)=(O_{1}, O_{2}, \ldots, O_{n})$ as a transaction's inputs and outputs.

Transactions have older inputs than outputs, more formally:

$$
\max(I(t)) < \min(O(t))
$$

We also define the transaction function $T_{n}(i)$ which points to  the ith transaction for the plasma chain's first n transactions.

$$
T_{n}: (0, n] \rightarrow TX
$$

As a shortcut,

$$
t_{i} = T_{n}(i)
$$

We define the priority number (lowest exits first) of a transaction, $p(t)$, as:

$$
p(t) = \max(I(t))
$$

We say that a transaction $t_{2}$ stems from a transaction $t_{1}$ (an output from $t_{1}$ eventually chains into an input of $t_{2}$) if the following holds:

$$
stems\_from(t_{1}, t_{2}) = (O(t_{1}) \cap I(t_{2}) \neq \varnothing) \vee (\exists t : stems\_from(t_{1}, t) \wedge stems\_from(t, t_{2}))
$$

A transaction $t_{i}$ with $n$ inputs is considered an "out of nowhere" transaction if any of its input "points to itself":

$$
\exists j \in [0, n): I(t_{i})_{j} = \max(O(t_{i-1})) + j
$$

We need to show that nothing that stems from an "out of nowhere" transaction can have a lower priority number than a transaction with inputs older than the "out of nowhere" transaction. Let $t_{v}$ be some transaction and $t_{nw}$ be an out of nowhere transaction such that, from our previous definition:

$$
\max(I(t_{v})) < \max(I(t_{nw}))
$$

and therefore by our definition of $p(t)$

$$
p(t_{v}) < p(t_{nw})
$$

So $p(t_{v})$ will exit before $p(t_{nw})$. We now need to show that for any $t'$ that stems from $t_{nw}$, $p(t_{v}) < p(t')$. Because $t'$ stems from $t_{nw}$, we know that:

$$
(O(t_{nw}) \cap I(t') \neq \varnothing) \vee (\exists t : stems\_from(t_{nw}, t) \wedge stems\_from(t, t'))
$$

If the first is true, then we can show $p(t_{nw}) < p(t')$:

$$
p(t') = \max(I(t')) \geq \max(I(t') \cap O(t_{nw})) \geq \min(O(t_{nw})) > \max(I(t_{nw})) = p(t_{nw})
$$

Otherwise, there's a chain of transactions from $p_{nw}$ to $p'$ for which the first is true, and therefore the inequality holds by transitivity.

So basically, anyone can exit before anything that stems from an "out of nowhere" transaction, as long as they're following the protocol (don't spend after a potentially invalid transaction!).

## Required Properties

Our exit game basically defines a set of "spendable outputs" (exits count as spends). For example, spent outputs are not spendable in our rules. We need to define some rules that this game must satisfy before we actually prove that the game is correct. We call these rules "safety" and "liveness", because they're largely analogous.

### Safety

The safety rule, in English, says "if an output was exitable at some time and is not spent in a later transaction, then it must still be exitable". If we didn't have this condition, then it might be possible for a user to receive money but not be able to spend or exit from it later.

Formally, if we say that $F(T_{n})$ represents the set of spendable outputs for some Plasma chain and $T_{n+1}$ is $T_{n}$ plus some new transaction $t_{n+1}$:

$$
\forall f \in F(T_{n}) : f \not\in I(t_{n+1}) \implies f \in F(T_{n+1})
$$

### Liveness

The liveness rule basically says that "if an output was exitable at some time and *is* spent later, then immediately after that spend, either it's still exitable or all of the spend's outputs are exitable, but not both".

The second part probably makes sense - if something is spent, then the resulting outputs should be spendable. The first case is special - if the spend is *invalid*, then the outputs should not be spendable and the input should therefore be spendable. So a short way to describe "liveness" is that all *valid* transactions (and *only* valid transactions) should impact the set of spendable transactions.

Formally:

$$
\forall f \in F(T_{n}), f \in I(t_{n+1}) \implies f \in F(T_{n+1}) \oplus O(t_{m+1}) \subseteq F(T_{n+1})
$$

## Proof

### Exit Game Formalization

#### Spends

We give the list of "competing" spends of an input as follows:

$$
spends(t) = \{ t_{i} | i \in (0, n], I(t_{i}) \cap I(t) \neq \varnothing \}
$$

#### Canonical Transactions

We first define a function that determines which of a set $T$ of transactions came "first":

$$
first(T) = t \in T : \forall t' \in T, t \neq t', min(O(t)) < min(O(t'))
$$

Then we can define a function $canonical$ that takes a transaction and returns the "canonical" spend in its set of competing spends.

$$
canonical: TX \rightarrow TX
$$

$$
canonical(t) = first(spends(t))
$$

Finally, we say that "reality" is the set of canonical transactions for a given Plasma chain.

$$
reality(T_{n}) = \{ canonical(T_{n}(i)) | i \in (0, n]\}
$$

#### Unspent, Double Spent

We define two helper functions "unspent" and "double spent". $unspent$ takes a set of transactions and returns the list of outputs that haven't been spent. $double\_spent$ takes a Plasma chain and returns any outputs that have been included in more than one transaction.

First, we define a function $txo$ that takes a transaction list and returns a list of transaction outputs.

$$
txo(T) = \bigcup_{i = 1}^{n} O(T_{i}) \cup I(T_{i})
$$

Now we can define $unspent$:

$$
unspent(T) = \{ o \in txo(T) | \forall t \in T, o \not\in O(t) \}
$$

And finally, $double\_spent$:

$$
double\_spent(T) = \{ o in txo(T) | \exists t \exists t' \in T, t \neq t', o \in O(t) \wedge o \in O(t') \}
$$

#### Exitable Outputs

Combining all of these things, we define our function for exitable outputs given a set of transactions, $F$, as:

$$
F(T_{n}) = unspent(reality(T_{n})) \setminus double\_spent(T_{n})
$$

Basically, the set of exitable outputs are the outputs that are part of a canonical transaction, are unspent, and were not spent in two or more non-canonical transactions. This last part effectively punishes users for double spending. 

### Satisfies Properties

This next section demonstrates that the function describes above satisfies the desired properties of safety and liveness.

#### Safety

Our safety property says:

$$
\forall f \in F(T_{n}), f \not\in I(t_{n+1}) \implies f \in F(T_{n+1})
$$

So to prove this for our $F(T_{n})$, let's take some $f \in F(T_{n})$. From our definition, $f$ must be in $unspent(reality(T_{n}))$, and must not be in $double\_spent(T_{n})$.

$f \not\in I(t_{n+1})$ means that $f$ will still be in $reality$, because only another transaction spending $f$ would change this. Also, $f$ can't be spent (or double spent) if it wasn't used as an input. So our function is safe!

#### Liveness

Our liveness property states:

$$
\forall f \in F(T_{n}), f \in I(t_{n+1}) \implies f \in F(T_{n+1}) \oplus O(t_{n+1}) \subseteq F(T_{n+1})
$$

This is a little more annoying to prove, because we need to show each implication holds separately, but not together. Basically, given $\forall f \in F(T_{n}), f \in I(t_{n+1})$:

$$
f \in F(T_{n+1}) \implies O(t_{n+1}) \cap F(T_{n+1}) = \varnothing
$$

and

$$
O(t_{n+1}) \subseteq F(T_{n+1}) \implies f \not\in F(T_{n+1})
$$

Let's prove the first. $f \in I(t_{n+1})$ means $f$ was spent in $t_{n+1}$. However, $f \in F(T_{n+1})$ means that it's unspent in any canonical transaction. Therefore, $t_{n+1}$ cannot be a canonical transaction. $O(t_{n+1}) \cap F(T_{n+1})$ is empty if $t_{n+1}$ is not canonical, so we've shown the first statement. 

Next, we'll show the second statement is true. $O(t_{n+1}) \subseteq F(T_{n+1})$ implies that $t_{n+1}$ is in $reality(T_{n+1})$. If $t_{n+1}$ is in $reality(T_{n+1})$, and $f$ is an input to $t_{n+1}$, then $f$ is no longer in $unspent(reality(T_{n+1}))$, and therefore $f \not\in F(T_{n+1})$.

## In-flight Exit Game

### Construction

Our exit game involves two periods, triggered by a user who posts a bond alongside the transaction in limbo.  The first period consists of the following:

1. Honest owners of inputs or outputs to the posted transaction "piggyback" and post a bond, claiming that they have not double spent.
2. Any user may present other spends of the posted transaction's inputs.

Users who present a spend (2.) must place a bond. Any other user may present an earlier spend, claim the bond, and place a new bond. This construction ensures that the last presented spend is the earliest known spend. 

The first user to present a spend kicks all outputs from the exit queue. 

If no other spends of the posted transaction are brought to light in this first period (as per 2.), then the second period asks for on-chain pointers to spends of the outputs, which invalidates that output's exit and removes it from the queue. The bonds of all inputs and honest outputs are returned.

Otherwise, if other spends of the posted transaction's inputs are brought to light, period 2 serves to determine whether the posted transaction is canonical. Any user may present:
1. The posted transaction, included in the chain before any alternative spends presented in period 1. 
2. Spends of any "piggybacked" inputs or outputs (except the presented transaction itself).
3. That some input is not in the set of "unspent reality" by demonstrating that the transaction that created the input is not canonical. This means revealing both the transaction that created the input, as well as some alternative spend of any of that revealed transaction's inputs such that the alternative spend came first and can be properly prioritized.

If (1.) is presented, then all (unspent, piggybacked) inputs are kicked out of the exit queue, all (unspent, piggybacked) outputs are added back into the queue, and remaining bonds over the inputs are refunded.

Challenges of type (3.) basically ensure that the piggybacked inputs are part of a canonical transaction.

Note that challenges of piggybacked inputs or outputs (via 2. or 3.) make the inputs or outputs unexitable, even if the input/output isn't in the exit queue at that time. 

This means that at the end of the game, only correctly exitable transactions will still be in the queue. 

### Justification

This construction essentially allows any individual output to exit if it satisfies the following conditions:
1. A valid earlier spend of one of its inputs does not exist on-chain.
2. A valid spend of it does not exist on-chain.
3. It has not double spent (though if it has, the first valid spend IS exitable by the game)

### Exit Game Implements Function

Finally, we show that the exit implements the function for determining exitable outputs. 

## Summary 

We fixed Plasma.
