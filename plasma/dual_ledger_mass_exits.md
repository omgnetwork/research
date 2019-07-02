## Dual submissions of ledger state for cheap mass-exits

We propose introducing a dual nature of the operator's submissions (commitments) to the ledger state: UTXO-based and account-based.
The goal is for users to be able to start an exit of all UTXOs in possession using a single merkle-proof and limited root chain data.

### Problem

Without mass-exit facilities, the gas cost required for users to exit the funds limits the number of UTXOs present on chain to roughly 550K.
This is because for every UTXO exited, a merkle proof must be published and verified on the root chain contract (~280K gas, judging based on recent `plasma-contracts` impl.).

### Construction

For simplicity let's assume that there's only one token.
For multi-token support, all tokens are just treated separately.

Every `submitBlock` from the operator publishes two merkle roots: that of the block of txs (as done now) and that of an **account-based ledger** of the form:
`[{owner_address, sum_of_funds}, ...]`.

The `sum_of_funds` is a commitment of the operator to much do all the accounts hold at given child chain height.
The merkle root of the account-based ledger is verified by the Watchers on every block (TODO: can be less frequent?).

There is an **account exit** available, an additional kind of exit. `startAccountExit` takes in the following parameters:
  - `{owner_address, sum_of_funds}` - the `account_state`
  - `N` - height of the account-based ledger submission used
  - the proof of the given `account_state` inclusion in account-based ledger at `N`
  - `exclusions` - a list of `[{utxo_pos, amount}]` UTXOs that were spent by `owner_address` somewhere at heights `>=N`

Assume `K` is the height of the `startAccountExit` being mined.

`startAccountExit` has the following effects:
  - an exit of amount `sum_of_funds - sum(exclusions)` to `owner_address` is scheduled according to the age of submission `N` and exit start `K` (similar to how it is done for an UTXO-based standard exit)
  - if any of the `exclusions` is incorrect, i.e. at the given `utxo_pos` there is something different than `{owner_address, amount}` - the account exit can be challenged
  - if there exists _any spend_ from `owner_address` published at height `>=N`, other than from UTXOs in `exclusions` - the account exit can be challenged. Published means included in a child block or used in an exit
  - no UTXO-based exits (SE & IFE alike) are allowed to be finalized for `owner_address` except those listed in the `exclusions`
  - funds that `owner_address` receives after `K` can be operated with normally. (TODO - is that so? see [_finality question_](https://github.com/omisego/research/pull/106#issuecomment-507705003))

The account exit effectively "closes" the account of `owner_address` and compacts it, allowing to cheaply exit all the funds held.

### Rationale

Why could it help?
  - allows to mass exit all funds belonging to `owner_address` roughly at a cost of a single UTXO standard exit, so that roughly 550K users are able to securely hold state, regardless of their UTXO count (**NOTE** the single token assumption!)
  - removes the requirement to manage UTXOs, you can have as many as you please

Why does it work?
  - operator can't put arbitrary data in the account-based ledger submission to exit, because the exit priority is observed (standard plasma security)
  - no one can use an old account-based ledger submission to exit funds which were later spent.
  Any spend that comes after height `N` can challenge
  - the `exclusions` declaration allows one to deal with funds spent after the last known correct pair of submissions from the operator - those funds are most likely to require IFEs
  - it is easy to validate an account exit and reasonably easy to compute challenge.
  It suffices to check the `balance(address)` at height `K` - it must equal to `sum_of_funds - sum(exclusions)`.
  If that's not the case, blockchain from `N` to `K` must be scanned to find the violating transaction/exit.
  After this initial check, all blocks/exits seen must be checked to not include anything spent by `owner_address` ever again, as long as it's been created before moment height `K` (TODO - is the "`K`" part possible/necessary? see _finality question_ above).
