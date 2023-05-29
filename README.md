# Crumple
a payments network based on trust channels

**credit**: based on and inspired by [@fiatjaf's Rumple idea](https://fiatjaf.com/rumple.html) and refined after a discussion and feedback from him.

If you are familiar with Rumple, then you can understand Crumple as a simpler solution to [decentralized commit](https://fiatjaf.com/3cb7c325.html) problem.

## Goals

Leverage trust relationships to build a payment network that has the following features:

1. **Scalability** no on-chain transactions for setup, and at worse a single on-chain transaction to settle a trust channel.
2. **Onboarding** instant inboud liquidity to anyone you trust.
3. **Non-interactivity** transactions do not require the active involvement of the reciepient. (on top of the UX advantages, keys can stay cold and secure)
4. **Self-custody** availbility providers get no custody or control over funds.

### Privacy

The network depends on trust, reputation, and relationships between participants, so perfect privacy isn't a strict goal, that being said, requiring no on-chain transactions is already a win, and we should aim to minimizing metadata leaks as much as possible as we develop this concept and get feedback from others.

## How it works

For now, and until an MVP is developed the rough details of how the network works is almost exactly the same as described in [Rumple](https://fiatjaf.com/rumple.html), with one difference: replace the the [Blockchain commit method](https://ripple.ryanfugger.com/Protocol/BlockChainCommitMethod.html) with a [Wintess commit method](#Witness-commit-method)

### Witness commit method

A witness, is a server identified by a publicKey and a URL similar to a [Registry](https://ripple.ryanfugger.com/Protocol/RegistryCommitMethod.html) with the difference that it publishes submitted preimages, in a public Log similar to [Certificate Transparency Logs](https://certificate.transparency.dev/howctworks/).

Reciepients include within their invoices (or somewhere else linked to in their invoice) either a list of Witness Logs they trust (white list), and/or Witness Logs they distrust (block list).

Before a sender to attempts to pay an invoice, they check if there is a possible commonly trust Witnees log between them and the reciepient. If so they find a payment route somehow.

Once a route is discovered, for example A -> B -> C -> D, the sender A creates an IOU to B condiditioned on B having a signed root from the Witness Log, proving the inclusing of the preimage before a certain timeout.

B then does the same for C, and so does C for D.

Now one of the following scenarious may occur:

1. **Mismatch**
D doesn't actually trust that Witness Log (A made a mistake)
So D rejects the IOU and let it timeout.
2. **Stolen preimage**
D does trust the Witness Log and shares the priemage with it, but the log fails to include it in time or at all.
Now possibly the log might share the preimage with A, resulting in a loss to D
D will not trust the log anymore, so the log loses D as a customer and reputation among D's friends.
3. **Broadcast fail** 
D does share the preimage with the log, and the log responds to D with the new signed root of the log, but it fails to broadcast the log in time, and D doesn't share that signature upstream either. 
A doesn't see the preimage so she doesn't have a reciept in time, to recieve what she paid for.
Eventually A will see the preimage and realize her IOU to B is valid, and she lost money.
A will not trust the log anymore, the log loses a customer and reputation within A's friends.
5. **Duplicity (forks)**
D does share the preimage with the log but it the log shows a different fork to D than the rest.
D now has a valid IOU from C and continue to trust the log.
A, and B do not see the preimage in the public fork, so they assume the transaction timed out.
Eventually A and B will see the preimage, as D show it to C to redeem the IOU's value, and C does the same with B.
Now everyone has a proof that the log is a cheater, and that log will never be trusted by anyone who can see that proof (two signed merkle roots with the same height, but two different hashes).
