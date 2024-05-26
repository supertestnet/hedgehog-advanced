# Hedgehog Advanced
A more advanced implementation of my hedgehog protocol

This one lets you atomically pay your counterparty to pay a lightning invoice for you. So hedgehog can pay lightning invoices now.

Next I want to add support for *receiving* via lightning. I think it's much harder but I wrote out a spec for it here:

Here’s what I think should happen: the user’s wallet should contact their HSP and ask for a hodl invoice for whatever amount they want to receive, locked to a payment hash that only the user knows. The HSP should give it to them, then, the user should show that to whoever wants to pay them. When the HSP detects a pending payment locked to that invoice's payment hash, he should offer the recipient a “payment” htlc in their hedgehog channel, with a timelock of 40 blocks or less (just ensure it’s less than whatever htlc is paying the HSP), and it should be an output of “force_close_tx_1,” not of “force_close_tx_2” or “alt_force_close_tx_1” or “alt_force_close_tx_2” (alt_force_close_tx_1 and alt_force_close_tx_2 being the tx's Bob is supposed broadcast in case Alice force closes in the state prior to when she last sent Bob money). The recipient is supposed to settle this htlc by disclosing the preimage of the lightning invoice and then its balance will be added to the recipient’s side of the hedgehog channel in another state update.

I want to examine that in more detail. The state with the htlc is created by the HSP, who adds the htlc to force_close_tx_1 and deducts the htlc’s value from the balance he is supposed to receive via force_close_tx_2 or alt_force_close_tx_2. If the HSP force closes at this stage, he will use the prior tx1 and tx2, and those do not contain an htlc for Bob. So it is important that Bob gets the HSP to “fully” revoke that state before Bob discloses to the HSP the preimage to the htlc. The HSP and Bob can probably do it like this: after the HSP sends Bob the “new” tx1 and tx2 where tx1 contains an htlc for Bob, Bob should reply with a state update that changes nothing, and the HSP can then reply again with another state update that changes nothing.

That ensures that the “prior” state (containing a tx1 and a tx2 without htlcs for Bob) is “fully” revoked by the HSP: the only state the HSP can safely force close in is one where Bob has an inbound htlc. And for Bob it’s only slightly different. By “replying” he “fully” revoked whatever state he had coming to him before the HSP offered him an htlc, and “conditionally” revoked the state containing an inbound htlc that the HSP initially offered. So now he has an inbound htlc and can either force close in a state where *he* creates the inbound htlc or wait til the HSP force closes and then *update* the HSP’s force closure to one where he (Bob) has an inbound htlc. Either way, no matter who force closes, Bob gets an inbound htlc where the HSP can take the money back after an absolute timelock of 40-ish blocks expires, and the rest of the money will be dispersed after a relative timelock of 2016 blocks. But there are 4 different transactions with 4 different txids that might produce this nearly guaranteed scenario.

What I want to happen next is for Bob to disclose the preimage for the htlc. If he does that, and we force close in any of the above-mentioned 4 scenarios, no big deal: Bob can sweep the money using the preimage and the HSP can use it to settle his inbound lightning payment. But if Bob does *not* disclose the preimage, the HSP should force close when the 40 blocks approach. If Bob *then* sweeps the funds with the preimage, well and good, the HSP can get the preimage from the blockchain and use it to settle his inbound lightning payment. And if Bob *still* does not disclose the preimage, it's still no big deal: the HSP simply sweeps his money back using the timelock path and Bob loses the inbound payment, but it’s his fault – he waited too long.

Now, there is an issue here: I need to prevent the HSP from broadcasting an old, unsafe, “fully revoked” (or “partially revoked”) force_close_tx_1 tx while the recipient is offline and then sweeping the attached htlc thanks to its 40 block timelock being expired. To prevent that, after an htlc’s 40-ish block “absolute” timelock expires, it should only be sweepable into a “revocable script” address where the HSP has to wait 2016 blocks (a relative timelock) or the recipient can take the money immediately if he learns the HSP’s revocation key – a random one that the HSP makes up, not the one he is supposed to reveal when revoking this state in the “normal” hedgehog flow. In the “happy” path, the HSP reveals the revocation key to this address after resolving the htlc, and also gives the recipient a signature letting him sweep the htlc immediately (without the preimage) if the HSP force closes later in a state where this htlc exists. If the htlc resolves and the HSP does not immediately reveal this key & sig, the intended recipient is online, so he can force close. This will result in one of two things happening: due to a race condition, the HSP might broadcast an alternative force closure transaction where the HTLC was not yet resolved, in which case the intended recipient can sweep it using his preimage *and* get the rest of his money from the *regular* hedgehog protocol. Or it force closes in a state where the htlc is resolved, which is almost equivalent to the happy path: the recipient gets *all* of his money from the regular hedgehog protocol, and the HSP has the preimage (because the htlc resolved) so he gets reimbursed the normal way. The only downside here is there's a force closure, but that's just the fault of whoever decided to force close, which either party is always allowed to do anyway, so no big deal, they just force closed in an unusual way.

Whereas if the HSP *did* reveal the revocation key and fully revoked this state, all is well. If a force closure happens, one of two “good” things results: either the intended recipient sweeps it using the preimage or, if he's offline, the HSP may sweep it into a “revoked” revocable address after 40 blocks, but no harm is done: whenever the intended recipient gets online (as long as it's within 2 weeks), he can sweep it from there, thus foiling the HSP’s attempted theft.

Meanwhile, what happens if the recipient broadcasts an old, unsafe, “fully revoked” (or “partially revoked”) force_close_tx_1 tx? Well, the HSP can sweep all funds except maybe not the HTLC. If the 40 blocks are up, the HSP can try to sweep those too, but there’s a race condition: the intended recipient might get them unlawfully via the preimage path. To prevent that, the intended recipient should “only” be allowed to sweep them into a “revocable script” address where the intended recipient has to wait 2016 blocks (a relative timelock) or the HSP can take the money immediately if he learns the recipient’s revocation key – one which the recipient makes up. In the “happy” path, the recipient reveals the revocation key to this address after resolving the htlc. If the htlc resolves and the recipient does not immediately reveal this key, the HSP should force close. This will result in one of two things happening: due to a race condition, the recipient might broadcast an alternative force closure transaction where the HTLC was not yet resolved, in which case the HSP either gets the money back (yay, he earned money by being honest that he wouldn’t otherwise have gotten) or the intended recipient uses the preimage to sweep the HTLC into a revocable address that he hasn’t disclosed the preimage to – and here the HSP doesn’t earn anything for his honesty, but he also didn’t lose anything except a dishonest counterparty, so all is well. Another thing that might happen is this: if the channel force closes in a state where the htlc is resolved, the recipient gets *all* of his money from the regular hedgehog protocol, and, again, the HSP loses nothing except a channel partner (one who violated the protocol, so he should be happy he's gone).

Whereas if the recipient *did* reveal the revocation key and fully revoked this state, all is well. If a force closure happens, one of two “good” things results: either the intended recipient sweeps the htlc using the preimage and the HSP sweeps it right back, thus penalizing the wrongful broadcasting of old state, or the HSP sweeps it using the timelock path and the intended recipient *still* loses an HTLC he would otherwise have gotten to keep -- so he is still penalized for wrongfully broadcasting old state.

Yay! I think that covers everything and I now have a protocol for receiving lightning payments into a hedgehog channel!
