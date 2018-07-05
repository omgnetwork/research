MoreVP Alice-Bob Scenarios
---

Note that these scenarios work with N participants, not just limited to 1 input 1 output. 

Also, note that in all of these scenarios the exits are given the priority of the youngest input to the transaction.

#### Alice & Bob are honest and cooperating:

Happy path.

1. Alice spends UTXO1 in TX1 to Bob.
2. Operator withholds, neither Alice nor Bob know if the transaction has been included. 
3. Someone with access to TX1 (Alice, Bob, or otherwise) starts an exit and places `exit bond`.
4. Alice and Bob "piggyback" onto the exit and each put up `piggyback bond`.
5. After 1 week, Bob receives the money as if the transaction were included. All bonds are refunded.


#### Bob spends:

Bob trying to cheat.

1. Alice spends UTXO1 in TX1 to Bob, creating UTXO2.
2. TX1 is included in block N.
3. Bob spends UTXO2 in TX2.
4. TX2 is included in block N+M.
5. Bob starts an exit from TX1.
6. Bob "piggybacks" onto the exit and places `piggyback bond`
7. Someone reveals TX2 spending UTXO2, so Bob's `piggyback bond` is slashed.


#### Alice double spends:

This is the case where Alice is malicious and has a double-spend secretly included before her transaction to Bob.

1. Alice spends UTXO1 in TX1 to Bob.
2. Operator withholds, neither Alice nor Bob know if the transaction has been included. 
3. Unbeknownst to Bob, Alice has double spent UTXO1 in TX2. TX2 is included in the withheld block, but TX1 isn't.
3. Someone with access to TX1 (Alice, Bob, or otherwise) starts an exit and puts up `exit bond`.
4. Alice and Bob "piggyback" onto the exit and each put up `piggyback bond`.
5. Within 3.5 days, the operator reveals TX2. 
6. No one is able to reveal TX1 in response, so Bob's `piggyback bond` is refunded and Bob doesn't get to exit. Alice's `piggyback bond` is slashed. Operator gets `exit bond` for (5). 

#### Alice spends later:

Here we're preventing Alice from creating a later spend that steals money from Bob.

1. Alice spends UTXO1 in TX1 to Bob.
2. TX1 is included in block N.
3. Later, Alice double spends UTXO1 in TX2 to Carol in block N+M (chain now invalid).
4. Alice attempts an exit on TX1 and places `exit bond`.
5. Alice "piggybacks" onto the exit and puts up `piggyback bond`.
6. Within 3.5 days, the someone reveals TX2, requiring that someone else reveal TX1 in response.
7. Someone reveals TX1 and proves that TX1 was included before TX2, and gets `exit bond`. Alice's `piggyback bond` is refunded.

#### Alice's second spend is non-canonical:

Here's the scenario in which Alice can have funds locked by double spending:

1. Alice and Carol spend (UTXO1, UTXO2) in TX1 to Bob. 
2. Carol double spends UTXO2 in TX2 to herself.
3. Alice sees this spend included in the chain, so she spends UTXO1 in TX3 to Bob. 
4. Operator witholds, includes TX1 in the withheld block.
5. Someone starts an exit on TX3 and places `exit bond`.
6. Alice and Bob "piggyback" onto the exit and each put up `piggyback bond`.
7. Operator reveals TX1, no one is able to reveal TX3. Operator gets `exit bond`.
8. Alice's `piggyback bond` is slashed, Bob's `piggyback bond` is refunded. 
9. Alice "loses" UTXO1.

To mitigate this scenario, we should add timeouts to each transaction. 

##### Multiple exits on the same input

1. Alice has UTXO1 and UTXO2. 
2. Alice creates TX1 spending UTXO1 and UTXO2 to herself.
3. Alice creates TX2 spending UTXO1 to herself.
4. Alice starts an exit for TX1 and puts up `exit bond`.
5. Alice starts an exit for TX2 and puts up `exit bond`.
6. Someone reveals TX1 to challenge TX2. Someone reveals TX2 to challenge TX1.
7. Only UTXO2 is exitable, because UTXO1 was shown to have been double spent.

#### Operator tries to steal funds

We assume here that:
(1) The operator can create UTXOs "out of nowhere"
(2) Correct nodes never spend any UTXOs created after the first "out of nowhere" UTXO
(3) "Out of nowhere" transactions cannot be exited with the in-flight protocol (easy to implement as a contract check)

##### Simple case

1. Alice has UTXO1, included in (valid) block N.
2. Operator creates UTXO2 "out of nowhere" and spends it to themself in TX2. 
3. Operator tries to exit TX2 (places `exit bond`, piggybacks onto the output(s) and places `piggyback bond` for each).
4. UTXO1 existed before UTXO2, so Alice submits a simple exit for UTXO1 and will have higher priority than TX2 (in-flight exits have "youngest input priority").

##### Multiple in-flights txs

1. Alice has UTXO1, included in (valid) block N.
2. Alice spends UTXO1 in TX1 to Bob, which is in-flight.
3. Operator withholds TX1.
4. Operator creates UTXO2 "out of nowhere" and spends it to themself in TX2.
3. Operator tries to exit TX2 (places `exit bond`, piggybacks onto the output(s) and places `piggyback bond` for each)
6. Alice submits an exit for TX1. Bob piggybacks. The "youngest input priority" rule means that TX1 will process before TX2 because UTXO1 will always be older than UTXO2 by assumption (2).
