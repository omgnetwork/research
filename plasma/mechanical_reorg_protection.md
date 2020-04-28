Mechanical reorg protection, proposal
==

Original doc from Pepesza, also see discussion there:
https://gist.github.com/paulperegud/3fb4bb96693abe2e0b175f7b20e64362

### How chain can be invalidated by the reorg?
A deposit is being spent by the owner on the child chain, and operator submits new block to the root chain.
Next deposit diappears from the root chain, creating a situation where operator has allowed money creation on the child chain,
leading to partial reserve and mass exit.

### Implemented protection - economical one
To protect the chain from invalidation by reorgs we do not allow users to spent deposits younger than N Ethereum blocks.
This gives us probabilistic guarantees from invalidation by random reorg.
Unfortunately, 51% attack targeting specifically OMG chain is still possible.
Its cost depends on N and current spending on security of Ethereum chain.
https://www.crypto51.app/ evaluates costs of 1 hour 51% attack at USD 85000.
Taking N=12, costs of attack against OMG chain is just USD 5000.

While viability of such attack depends not only on money but also on access to hash power, if we succeed
OMG chain will become a large target.

### Proposed solution that gives us full reorg immunity
(solution born in discussion with Pawel Thomalla)
Operator, while submitting a block, adds a hash describing his knowledge about deposits in the contract.
Contract computes his own summary of state of deposits and compares it with submitted one.
If not equal, transaction submitting the block is rejected.
Bonus points - this enables operator to spend deposits as soon as they appear, without waiting.

#### Technical description, pseudocode
```
submitBlock(bytes32 blockRoot, bytes32 depositsHash)
bytes ownHash = keccak256(all deposit blocks where blknum > N-2 and blknum < N-1)
require(ownHash == depositHash);
```

#### Gas cost analysis
200 + 42 (SLOAD, SHA3 on two words) gas per deposit made to the chain, paid by operator.

### Changes needed in client software
1. ability to rollback plasma blocks for both operator and watcher  
2. ability to evaluate such rollbacks with regard to byzantine behavior by operator  
