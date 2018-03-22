Plasma w/o Confrimations
===

Author: David Knott

---

**Note:** This is a cleaned up version of the original "Plasma w/o Confirmations" specification. The original version of this document can be found [here](https://hackmd.io/s/BkgnUZALf). 

---

## Background

[Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426) specifies that each transaction be "confirmed" by the transaction parties before it's considered valid. Transaction confirmations are necessary to avoid problems created by block withholding. A confirmation proves that a user has accesss to the block in which their transaction was included. 

However, confirmations are a little annoying because a user needs to sign a transaction, wait for their transaction to be included in a block, and then sign *another* confirmaton transaction. You can probably see how this could quickly become problematic.

This document specifies a version of Plasma that does *not* require confirmations and therefore only requires a single user signature to submit a transaction. Some implementation details are provided. As always, this construction has its own pros and cons and is subject to change as it's refined.

## Overview

The high-level summary of this specification is that transaction confirmations are removed by requiring that transactions be included within some number of Plasma blocks. Transactions that have "timed out" are considered invalid and should not be included in future Plasma blocks. An additional root-chain construction ensures that transactions are still safe if they're included but withheld.

## Rules

We specify two rules to make this construction as simple as possible:

1. Transactions must be included within `n` Plasma blocks.
2. Transaction inputs must be at least `n + 1` Plasma blocks old.

Rule #1 prohibits a Plasma operator from including a transaction a long time (`>n` blocks) after it's published. This wouldn't matter if we were using confirmations because a user could just refuse to confirm a very late transaction. 

Rule #2 limits the amount of work necessary to validate a transaction. Transactions are considered valid if they've 