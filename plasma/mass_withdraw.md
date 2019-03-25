Improve efficiency of mass-withdrawals.

By @DavidKnott and @kfichter:

# Mass withdrawals

Talks with Kelvin

Plasma MVP's exit time constraints make it vulnerable to Ethereum network congestion.  Mass withdrawals have the ability to improve upon this weakness by decreasing the average cost per utxo exit.

There are a few ways we can go about doing mass withdrawals.

### Single owner mass withdrawals
The initialy way I thought of to do single owner mass withdrawals was to have have the owner make a list of all the utxos they own client side.  From this list they'd submit a start position as well as a bitfield in which each bit would represent an exitor utxo from the start position onward.  Then if any of the UTXO's being exited is invalid anyone will be able to challenge the exit
This doesn't work though because there's no efficient way to prove that a given UTXO challenge response is the utxo from the challenged position in the bitfield.

To fix this problem we'll require the operator to add a field to each transaction when it's merkilized stating the it's utxos position in relation to it's owners other UTXO's


### Multi owner mass withdrawals
Multi owner mass withdrawals require the owner of every utxo being exited to sign off.  The leader of the exit is the one who submits the mass exit to Ethereum.  The leader must specify the block and start position of the bitfield.  Their signatures are represented in a bitfield.  It's the responsibility of each owner to watch all mass withdrawals to make sure that their own UTXO's aren't withdrawn without their permission.  The exit leader will also have to submit the merkle root of all the transactions being withdrawn.

Both mass withdrawals will wait a challenge period which will allow for anyone to request the exit leader to provide the utxo and signature for the utxo that they claimed to have in the bitfield the previously submitted.

### Targeted Mass Withdrawals
By requiring the operator to create an incrementing nonce for each address's UTXO's we can allow for mass withdrawals where the exit leader submits a bitfield and a list of addresses they're withdrawing from.


### Tracking UTXO's Seperately
Unpsent transactions outputs can be tracked in a seperate merkle tree that's used to facilitate spending and exiting transactions.  This makes mass withdrawals more efficeient because each position in the bitfield is refering to an exitable UTXO.


### Getting Rid of Spent UTXO's
With the current Plasma MVP design we're unable to get rid of spent UTXO's because we need them to challenge exits that double spend.  Though if a double spending exit goes through it hurts everyone using the child chain except for the party who submitted the exit.  This means that we don't need everyone to keep spent UTXO's but enough to be able to submit a challenge in the case that one is needed.  We can further incentivize those wishing to keep spent UTXO's by rewarding them for submitting a succesfful challenge when someone attempts to double spend.




### Referencing inputs by transaction hash
We currently reference transaction inputs by their position in the child chain.  This is the easiest to think about but makes it hard to create sequences of transactions refering to each other without waiting for each transaction to be included in the child chain and submitted to the root chain.   This is because the operator chooses transactions position in the child chain as opposed to the transactions creators.  By having inputs use a transactions hash to reference it as opposed to it's position we can create sequences of transactions that build on each other while only committing them to the child chain if something goes wrong. 

### Value Summation in Mass Exits

It seems like it'll be necessary for the user that submits a mass exit to attach a summation of the value of all referenced UTXOs. For example, if exiting UTXOs worth (10 ETH, 20 ETH, 15 ETH), then the user will also attach the value "45 ETH" in some way. When the mass exit processes, this sum will be "reserved" for mass exit to be processed on a per-UTXO basis later. This is necessary so that invalid exits that process after 2 weeks can't steal money while the mass exit is still being processed.

#### Summation Validity

Unfortunately, it isn't easy to prove that this summation is valid. The above user might attach "50 ETH", which would obviously be invalid. We don't want users to be able to steal money in this manner.

One possible solution to this problem is a sum tree of sorts. Each leaf node in the tree would contain the tuple (`utxo_value`, `total_sum`), where `total_sum` represents the sum of all leaf nodes to the left of this node, inclusive of the node itself. For example, if the UTXO values are 10 ETH, 20 ETH, 15 ETH, then the leaf nodes would be (10, 10), (20, 30), (15, 45).

These leaves would be Merklized and the final sum + tree root would be published. The tree could be challenged in a TrueBit-esque game where two users iterate down the tree until they find the first node at which they disagree. They reveal this node as well as the node to the left of this node. The root chain makes a calculation to determine which party is correct (`left_total_sum` + `right_utxo_value` = `right_total_sum`). 

This requires log(n) transactions to the root-chain in the worst case, which is not ideal. It may be possible to construct more concise proofs, but I haven't figured anything better out yet. 
