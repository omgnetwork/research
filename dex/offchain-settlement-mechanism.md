Original Github issue: https://github.com/omisego/research/issues/88

## Why
In the [alternative proof doc](https://omisego.atlassian.net/wiki/spaces/RES/pages/26673228/Alternative+DEX+proof+design?atlOrigin=eyJpIjoiZWY3ZTI1ZDg2MzYxNGM2OGJiYjM3MjIxMGU2NDZlOTEiLCJwIjoiYyJ9) from clearpool it seems like they would like to have some off chain "settlement" mechanism. This just means they would like to be able double-check (likely by a third party) events history off-chain before finalizing them on-chain.

In current plasma chain, you can verify one tx off chain. However, if venue wants to verify a set of txs (chained txs) this would need some alternative mechanism.

## Possible Solution 0

Venue and Validator double signs each transaction. This would need more liveness assumption on validator, as validator would need to catch up on each transaction generation. However, this is the simplest solution.

## Possible Solution 1 - Chaining txs + Confirmation signature (MVP) by validator

One possible solution is to use same flow as [chaining txs within block](https://github.com/omisego/research/blob/master/plasma/plasma-mvp/chaining-txs-within-block.md). Venue chains all txs and send them to operator. However, instead of venue providing confirm sig, it is the validator to provide that. So validator gets block data and confirms whatever is there. Venue needs to get the confirm sig to be able to keep on generating next settlement transaction.

Be aware that the mechanism does not promise operator would put all txs into same block. In the case that operator rejected some chained transactions to be put in same block, those rejected transactions need to be regenerated and add the confirm signatures of validator.

## Possible Solution 2 - Mine Chained Txs First + Explicit range to finalize
This is a complex solution. Venue first submit chained txs to plasma. However, those txs needs one more confirm tx + one confirm signature to finalize.
1. The confirm tx would explicit the range of the chained txs to finalize. For example, a chained txs: A->B->C->D, this confirm tx can state it only wants to finalize: A->B->C without D.
2. The main trick here is that exit priority of transactions would be using the position of the confirm tx instead of their original tx position. Let's say somebody chained C->D', and confirmed both C->D and C->D'  afterward (double spending). If fraud operator let both confirm txs mined, only one of the two would have higher priority depends on which confirm tx got mined first.
3. Since exit priority is decided by the confirm tx, we would need another confirm signature (of the confirm tx) to confirm no invalid tx in front the confirm tx. MoreVp does not really works here as there might be no "input" for the confimation tx to change to higher priority.

So we can let the validator/regulator create the confirmation tx and confirm signature. Cool things here is that venue can just keep submit the event data and let validator/regulator comes in and stamp on the finalization process. If any of the settlement data does not looks right, venue can even start a new branch/fork of event history from any non-finalized position.

One thing to be aware is that this basically means we delay the verification of transaction to the time when confirmation tx is mined. How can we make watcher/operator efficiently knows the txs need to be verified from the history data might be an issue. (As currently it just need the verify whenever new tx comes) A naive solution is the confirmation signature would need to include all positions of the transaction so watcher does not need to run through all txs. 

## Alternative Solution - Validation token (Jamie & Chris's idea)
This idea is that the validator/regulator provide some "token" as the proof of validation. Venue has to put the token as input of the settlement. 

Also, note that validator/regulator might not need to do a full verification. For instance, they might be verifying the settlement price is correctly or not only insteald of also checking utxos.
