## Fast Finality

Fast finality is important, especially in the context of payments or exchanges. There hasn't been a significant amount of research into fast finality on Plasma. Currently, users need to wait until root chain finality before blocks on the child chain are considered valid. This waiting period can be quite large (6 blocks = ~1m30s at an absolute minimum, typically). 

Long waiting periods aren't great for payments. **The 5s credit card wait period is probably the maximum we're willing to accept.** No one wants to wait a minute and a half just to find out if their payment went through or not. This is motivation to develop a scheme for fast finality.

Unfortunately, fast finality in a real sense is probably not possible without payment channels. We might see the development of something like the lightning network on top of Plasma, but it could be useful to develop some idea of native fast finality. Let's go through some of attempts on fast finality to explore the various issues. 

### Shorter Block Times

Shorter block times seem like a natural way to reach faster finality. If we can reduce Plasma block times to ~5s and ensure that most transactions on the network can be cleared within a single block, then we're in business. Assuming VISA throughput of ~25,000 TPS (yes, this is played out), then we can basically hit this rate with 2^16 (= 65536) tx blocks. 

Note that this would mean multiple child chain blocks within a single parent block. The main problem is equivocation by the operator - the operator might *say* that block N is X, but then might publish Y to the root chain. We can't stop an operator from signing two blocks at the same height, but we *can* severely punish this behavior. The most effective way to accomplish this is by having the operator put up a massive bond.

At this point it's probably worth considering the attack vectors here. The severity of available attacks will determine how much we have to punish the operator. In the context of a payments chain, we're most likely thinking about very many small-value payments between consumers and merchants. A payments chain might contain significant value, but realistically most users won't hold thousands of dollars on Plasma. Therefore, it's unlikely that an operator will directly gain a lot by equivocating. This isn't an ideal assumption, but it's probably good enough for most purposes.

The bigger problem in payment chains is that the operator can reverse *lots* of transactions. They might not gain anything directly, but they'll definitely cause harm to many end users. Therefore we need to make the cost of attack high enough that the "madman" attack isn't worth it. Ballpark, if we're transacting ~150,000 USD of value per second, then it's probably good enough to have the operator put up a 100-200m USD bond. This will almost definitely cover the value being transacted between any finality period on the root chain, plus some additional value on top to deter madman attacks. We're still vulnerable if transaction value suddenly shoots up to more than 50% of the bond.

**Note:** The operator needs to bond up 2x the transaction value. Operator can make a purchase for $$ => seller sends item worth $$ => operator reverses transaction => operator has $$ AND item worth $$ (= 2x $$). Seller punishes => operator loses 2x $$ => operator has nothing.

### Finality Contracts

100-200m USD is still an absolutely massive bond. The operator doesn't even get anything in return for it! It's also useful to note that not *everyone* needs fast finality - a user sending money to their family is probably okay with waiting a few minutes. This is where finality contracts come into play. In a nutshell, the operator tells a user that their transaction will be included within X blocks or the user can "force" the operator to send the recipient funds.

The general idea here is that the operator sells "finality bandwidth". Users can buy this bandwidth to ensure that their transactions finalize instantly or merchants can buy this bandwidth for their users to use. Users can only make purchases of less value than the bandwidth. If the operator fails to include the transaction, then the bandwidth will be used to pay out the recipient. 

My guess is that this will be mostly purchased by merchants to give their users "free" transactions that finalize instantly in exchange for some maintenance fee paid by the merchant. 

The cool thing about this construction is that the recipient will *always* get the money. So this isn't real finality, but it *is* cryptoeconomic finality. We're usually assuming that the recipient is some sort of merchant, so it seems realistic to want the customer to experience instant finality and the merchant to have to deal with any difficulties. Merchant software will develop to handle these situations automatically.

There's an alternative to this solution where users or merchants actually purchase specific transaction slots in advance, but I think bandwidth is more general and usable.

### Payment Channels

There's another (similar) alternative to finality bandwidth where we use payment channels. Instead of allowing users or merchants to buy bandwidth, we instead allow merchants to pay for payment channels with the operator. The operator agrees to lock up some funds in the channel in exchange for a maintenance fee. 

When users want to pay the merchant, they sign a special type of transaction that's contingent on the successful completion of a payment from the operator to the merchant on a payment channel. The idea here is to shift responsibility to the operator, but it also requires cooperation from the operator to actually complete the payment channel payment on time.

## Summary

I think finality bandwidth is the way to go. It's the best of both worlds in terms of finality and generality, and it can be purchased by anyone on the network. We need to come up with a good way to construct the purchasing of bandwidth - my best solution here is a Plasma Cash chain where each "token" represents some amount of bandwidth. 
