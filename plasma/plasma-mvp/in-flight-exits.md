# Magic Withdrawals

## TL;DR

## Background

### Exit Priority

The [original Plasma MVP specification](https://ethresear.ch/t/minimal-viable-plasma/426) requires that exits be processed in priority order. Each exit references an unspent transaction output, and priority is determined by the age of the transaction output. This is generally necessary so that valid exits can be processed before exits stemming from invalid "out of nowhere" transactions. More information about exit priority can be found at the OmiseGO research repository [here](https://github.com/omisego/research/blob/master/plasma/plasma-mvp/explore/priority.md).

This post introduces a new way to calculate exit priority, which we call "youngest-input" priority. Instead of ordering exits by the age of the *output*, we now order exits by the age of the *youngest input*. This has the effect that exits of outputs, even if they're included in withheld blocks after "out of nowhere" transactions, will be correctly processed as long as they only stem from valid inputs.

### Exit Game

Plasma MVP also introduces an exit game, whereby some outputs are deemed "exitable". This game correctly incentivizes parties to challenge a "non-exitable" output. For example, spent outputs are considered "non-exitable." The game requires the exitor to place a bond, such that anyone who knows the output to be spent is sufficiently incentivized to reveal the transaction and take the bond. 

The basic exit game has one challenge condition - outputs must not be spent. This condition is enough to make the exit game complete because of a construction of confirmation signatures. Basically, both parties to a transaction must "sign off" on a transaction's inclusion in the chain. Honest clients won't sign off on withheld (and therefore possibly invalid) transactions, so honest clients won't accept funds that might be stolen by the operator (discussed more below).

### In-flight Transactions

Without confirmation signatures, the above exit game is exploitable by the operator in two ways:

1. The operator may include, but withhold, a transaction, and later grief the sender(s) by challenging exits from the transaction's inputs. 
2. More importantly, the operator may include a transaction after some invalid "out of nowhere" transaction that creates money for the operator. Standard exit priority dictates that the operator's outputs will be processed before the valid outputs, thereby stealing money. 

Our new exit game solves these problems without confirmation signatures. 

## Youngest-Input Priority

This proposal replaces output-age priority with something we call "youngest-input" priority. Basically, the exit priority of an output is now the age of its youngest input. We claim that this construction is safe (honest clients cannot have money stolen) as long as clients don't spend any outputs included after any invalid or witheld transaction.

We show that if there exists an "out of nowhere" input, then any transactions stemming from that input would have an even higher queue number than the "out of nowhere" itself. Therefore, valid outputs will be processed first.

### Safety Proof

We begin with definitions to be used throughout this post.

TX is the transaction space, of the form (`inputs`, `outputs`), where `inputs` and `outputs` are sets of integers representing position in the Plasma chain:

$$
TX: ((I_{1}, I_{2}, \ldots, I_{n}), (O_{1}, O_{2}, \ldots, O_{m}))
$$

For all $t \in TX$, we define the "inputs" function $I(t)=(I_{1}, I_{2}, \ldots, I_{n})$ and the "outputs" function $O(t)=(O_{1}, O_{2}, \ldots, O_{n})$.

Transactions have older inputs than outputs, more formally:

$$
\max(I(t)) < \min(O(t))
$$

We also define the mapping $T_{n}$ which maps a Plasma chain's first $n$ transaction indices to its first $n$ transactions.

$$
T_{n}: (0, n] \rightarrow TX
$$

As a shortcut,

$$
t_{i} = T_{n}(i)
$$

We define the priority number ("lower exits first", "lower is better") of a transaction, $p(t)$, as:

$$
p(t) = \max(I(t))
$$

We say that a transaction $t_{k}$ stems from a transaction $t_{j}$ (an output from $t_{j}$ eventually chains into an input of $t_{k}$) if the following holds:

$$
stems\_from(t_{j}, t_{k}) = (O(t_{j}) \cap I(t_{k}) \neq \varnothing) \vee (\exists t : stems\_from(t_{j}, t) \wedge stems\_from(t, t_{k}))
$$

A transaction $t_{i}$ with $n$ inputs is considered an "out of nowhere" transaction if any input "points to itself":

$$
\exists j \in [0, n): I(t_{i})_{j} = \max(O(t_{i-1})) + j
$$

This is how deposits are structured, so it's like a deposit that never happened - creating money "out of nowhere". 

Let $t_{v}$ be some transaction and $t_{nw}$ be an out of nowhere transaction such that, from our previous definition:

$$
\max(I(t_{v})) < \max(I(t_{nw}))
$$

and therefore by our definition of $p(t)$:

$$
p(t_{v}) < p(t_{nw})
$$

So $p(t_{v})$ will exit before $p(t_{nw})$. We now need to show that for any $t'$ that stems from $t_{nw}$, $p(t_{v}) < p(t')$ as well. Because $t'$ stems from $t_{nw}$, we know that:

$$
(O(t_{nw}) \cap I(t') \neq \varnothing) \vee (\exists t : stems\_from(t_{nw}, t) \wedge stems\_from(t, t'))
$$

If the first is true, then we can show $p(t_{nw}) < p(t')$:

$$
p(t') = \max(I(t')) \geq \max(I(t') \cap O(t_{nw})) \geq \min(O(t_{nw})) > \max(I(t_{nw})) = p(t_{nw})
$$

Otherwise, there's a chain of transactions from $p_{nw}$ to $p'$ for which the first is true, and therefore the inequality holds by transitivity.

So basically, anyone following the protocol can exit before anything that stems from an "out of nowhere" transaction.

## Required Exit Game Properties

Our exit game defines a set of "exitable outputs" given a transaction $t$. For example, spent outputs are not exitable in our rules. We formally define the properties that this game must satisfy. We call these properties "safety" and "liveness", because they're largely analogous.

### Safety

The safety rule, in English, says "if an output was exitable at some time and is not spent in a later transaction, then it must still be exitable". If we didn't have this condition, then it might be possible for a user to receive money but not be able to spend or exit from it later.

Formally, if we say that $E(T_{n})$ represents the set of exitable outputs for some Plasma chain and $T_{n+1}$ is $T_{n}$ plus some new transaction $t_{n+1}$:

$$
\forall o \in E(T_{n}) : o \not\in I(t_{n+1}) \implies o \in E(T_{n+1})
$$

### Liveness

The liveness rule basically says that "if an output was exitable at some time and *is* spent later, then immediately after that spend, either it's still exitable or all of the spend's outputs are exitable, but not both".

The second part probably makes sense - if something is spent, then all the resulting outputs should be exitable. The first case is special - if the spend is *invalid*, then the outputs should not be exitable and the input should still be exitable. So a short way to describe "liveness" is that all "canonical" transactions (and *only* "canonical" transactions) should impact the set of exitable transactions.

We define "canonical" later.

Formally:

$$
\forall o \in E(T_{n}), o \in I(t_{n+1}) \implies o \in E(T_{n+1}) \oplus O(t_{m+1}) \subseteq E(T_{n+1})
$$

## Proof

### Exit Game Requirements

We need to formally show our exit game produces the exitable outputs $E$. To help us along the way, we make a few useful definitions.

#### Spends

We call two transactions "competing" if they contain any of the same inputs. The list of "competitors" to a transaction is formally defined as follows:

$$
competitors(t) = \{ t_{i} | i \in (0, n], I(t_{i}) \cap I(t) \neq \varnothing \}
$$

Note that a transaction is a competitor to itself. 

#### Canonical Transactions

We call a transaction "canonical" if it's the first included of any of its competitors. 

We define a function that determines which of a set $T$ of transactions came "first":

$$
first(T) = t \in T : \forall t' \in T, t \neq t', min(O(t)) < min(O(t'))
$$

Then we can define a function that takes a transaction and returns the "canonical" spend in its set of competitors.

$$
canonical: TX \rightarrow TX
$$

$$
canonical(t) = first(competitors(t))
$$

Finally, we say that "reality" is the set of canonical transactions for a given Plasma chain.

$$
reality(T_{n}) = \{ canonical(t_{i}) | i \in (0, n]\}
$$

#### Unspent, Double Spent

We define two helper functions "unspent" and "double spent". $unspent$ takes a set of transactions and returns the list of outputs that haven't been spent. $double\_spent$ takes a list of transactions and returns any outputs that have been used as inputs to more than one transaction.

First, we define a function $txo$ that takes a transaction and returns a list of its inputs and outputs.

$$
txo(t) =  O(t) \cup I(t)
$$

Next, we define a function $TXO$ that lists all inputs and outputs for an entire set of transactions:

$$
TXO(T_{n}) = \bigcup_{i = 1}^{n} txo(t_{i})
$$

Now we can define $unspent$:

$$
unspent(T) = \{ o \in TXO(T) : \forall t \in T, o \not\in I(t) \}
$$

And finally, $double\_spent$:

$$
double\_spent(T) = \{ o \in TXO(T) : \exists t,t' \in T, t \neq t', o \in I(t) \wedge o \in I(t') \}
$$

#### Requirements

Combining all of these things, we define our function for exitable outputs given a set of transactions, $E$, as:

$$
E(T_{n}) = unspent(reality(T_{n})) \setminus double\_spent(T_{n})
$$

Basically, the set of exitable outputs are the outputs that are part of a canonical transaction, are unspent, and were not spent in two or more non-canonical transactions. This last part effectively punishes users for double spending. 

### Satisfies Properties

This next section demonstrates that the function described above satisfies the desired properties of safety and liveness.

#### Safety

Our safety property says:

$$
\forall o \in E(T_{n}), o \not\in I(t_{n+1}) \implies o \in E(T_{n+1})
$$

So to prove this for our $E(T_{n})$, let's take some $o \in E(T_{n})$. From our definition, $o$ must be in $unspent(reality(T_{n}))$, and must not be in $double\_spent(T_{n})$.

$o \not\in I(t_{n+1})$ means that $o$ will still be in $reality$, because only another transaction spending $o$ would change this. Also, $o$ can't be spent (or double spent) if it wasn't used as an input. So our function is safe!

#### Liveness

Our liveness property states:

$$
\forall o \in E(T_{n}), o \in I(t_{n+1}) \implies o \in E(T_{n+1}) \oplus O(t_{n+1}) \subseteq E(T_{n+1})
$$

This is a little more annoying to prove, because we need to show each implication holds separately, but not together. Basically, given $\forall o \in E(T_{n}), o \in I(t_{n+1})$:

$$
o \in E(T_{n+1}) \implies O(t_{n+1}) \cap E(T_{n+1}) = \varnothing
$$

and

$$
O(t_{n+1}) \subseteq E(T_{n+1}) \implies o \not\in E(T_{n+1})
$$

Let's prove the first. $o \in I(t_{n+1})$ means $o$ was spent in $t_{n+1}$. However, $o \in E(T_{n+1})$ means that it's unspent in any canonical transaction. Therefore, $t_{n+1}$ cannot be a canonical transaction. $O(t_{n+1}) \cap E(T_{n+1})$ is empty if $t_{n+1}$ is not canonical, so we've shown the first statement. 

Next, we'll show the second statement is true. $O(t_{n+1}) \subseteq E(T_{n+1})$ implies that $t_{n+1}$ is in $reality(T_{n+1})$. If $t_{n+1}$ is in $reality(T_{n+1})$, and $o$ is an input to $t_{n+1}$, then $o$ is no longer in $unspent(reality(T_{n+1}))$, and therefore $o \not\in E(T_{n+1})$.

## In-flight Exit Game

We consider a game $g(t):TX \rightarrow (o_1,o_2,...,o_n)$ which returns the list of exitable txo's of a given transaction. We must show that:

$$
\forall o \in TXO(t), o \in F(T_{n}) \iff o \in g(t)
$$

In other words, if the game is played on a transaction, it will correctly exit the transaction's entire list of exitable inputs or outputs.

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
3. It has not been double spent (though note that if it has, the unspent outputs of the first valid spend, if such a spend exists, are exitable separately).

(1.) is satisfied for the posted transaction by the challenge-response game between periods 1 and 2, and for each of the inputs by a type (3.) challenge in period 2. 

Both (2.) and (3.) are satisfied by any type (2.) challenge in period 2. 

In other words for any transaction $t \in T_{n}$ submitted to the game, any output $o \in t$, the game determines if:

1. $t \in reality(T_{n})$
2. $o \in unspent(reality(T_{n}))$
3. $o \not\in double\_spent(T_{n})$

If all of these are true, then:

$$
o \in unspent(reality(T_{n})) \setminus double\_spent(T_{n}) = E(T_{n})
$$

So our game will determine $E(T_{n})$ correctly.

## Summary 

We fixed Plasma.
