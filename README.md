# Crumple
a payments network based on trust channels

**credit**: based on and inspired by [@fiatjaf's Rumple idea](https://fiatjaf.com/rumple.html) and refined after a discussion and feedback from him.

If you are familiar with Rumple, then you can understand Crumple as a simpler solution to the [decentralized commit](https://fiatjaf.com/3cb7c325.html) problem.

## Goals

Leverage trust relationships to build a payment network that has the following features:

1. **Scalability** No on-chain transactions for set up, and at worse a single on-chain transaction to settle a mutual credit.
2. **Onboarding** Instant inbound liquidity to anyone you trust.
3. **Non-interactivity** Transactions do not require the active involvement of the recipient. (on top of the UX advantages, keys can stay cold and secure)
4. **Self-custody** Availability providers get no custody or control over funds.

### Privacy

The network depends on trust, reputation, and relationships between participants, so perfect privacy isn't a strict goal, that being said, requiring no on-chain transactions is already a win, and we should aim to minimize metadata leaks as much as possible as we develop this concept and get feedback from others.

## How it works

For now, and until an MVP is developed the rough details of how the network works is almost exactly the same as described in [Rumple](https://fiatjaf.com/rumple.html), with one difference: replace the [Blockchain commit method](https://ripple.ryanfugger.com/Protocol/BlockChainCommitMethod.html) with a [Wintess commit method](#Witness-commit-method)

### Witness commit method

A witness is a server identified by a publicKey and a URL similar to a [Registry](https://ripple.ryanfugger.com/Protocol/RegistryCommitMethod.html) with the difference that it publishes submitted preimages, in a public Log similar to [Certificate Transparency Logs](https://certificate.transparency.dev/howctworks/).

Recipients include within their invoices (or somewhere else linked to by a URL in their invoice) either a list of Witness Logs they trust (white list), or Witness Logs they distrust (block list).

Before a Sender to attempts to pay an invoice, they check if there is a possible commonly trusted Witnees log between them and the recipient. If so they find a payment route somehow.

Once a route is discovered, for example, A(Sender) -> B -> C -> D(recipient), the Sender creates an IOU to the next node B conditioned on B having a signed root from the Witness Log, proving the inclusion of the preimage before a certain timeout.

B then does the same for C, and so does C for the final recipient D.

If the Recipient doesn't trust the Witness (Sender failed to guess the mutually trusted Witness correctly, or the Recipient's list changed in the meantime), then it will ignore the IOU from C, and the entire transaction will timeout eventually.

Otherwise, the Recipient will send the preimage to the Witness.

If the Witness is honest, it publishes the preimage in the log before the timeout (defined by block number), in which case the Recipient will have a valid IOU, and the Sender will have a receipt of the payment.

The same will apply to B, and C, either by watching the log themselves or by getting the proof from the Recipient and passing it up the chain.

However, the Witness can still disrupt the transaction, either maliciously or out of incompetence, especially if the nodes don't gossip about the log, and only directly poll the Witness API themselves:

1. **Bad clock** 
- The Witness may slow down creating (or publishing) blocks, so despite the inclusion of the preimage _before_ the timeout block, it is seen by one or all participants _after_ the timeout period they intended.
- The Recipient gets a valid payment, and the Sender gets a valid receipt however it may be too late to use that receipt as intended.
- According to the specific case, either the Sender or the Reciever will effectively lose money because of this disruption, and won't trust the Witness again.

2. **Censorship**
- The Witness does NOT publish the preimage in the public log.
- The log advances, adding more blocks until the timeout block is reached.
- The Recipient can't prove any malice, but it will not trust the Witness anymore, on the basis of incompetence, or censorship.

3. **Duplicity**
- It is possible that the Witness colludes with the Sender, by showing the Recipient proof of inclusion of the preimage (the condition of the validity of the IOU), but publicly publishes another fork that excludes the preimage until the timeout block.
- Now the Recipient serves the Sender thinking that they now have a valid IOU from their direct counterpart C.
- As soon as the Recipient tries to "collect" the IOU from C, C can offer the exclusion proof, and the IOU is considered invalid.
- However, now both the Recipient and C have proof of duplicity and can use it to prove to everyone involved and the wider network that the Witness should never be trusted again.

Put in other words, users can lose some value by the incompetence, or censorship by the Witness, but they will lose the users' trust. The log can't steal funds, but it can help the Sender steal, risking a fatal blow to its reputation.

Users can easily detect censorship and bad service, but for duplicity, the need to actively communicate and compare their perspectives of the log.

### Watchtowers

Duplicity proofs are objective and can be demonstrated by anyone to anyone, but it is probably impractical to expect all users to gossip these proofs.

Watchtowers are proposed repositories, that users can query or submit duplicity proof to.

These Watchtowers, in turn, can either federate or gossip using more p2p approaches.

Who would want to run a watchtower?

1. Competing Witnesses
2. Wallet developers
3. Exchanges
4. Markets

### Non-interactivity

In a direct payment to a trusted counterpart, there is no need for interactivity, you can sign an IOU with no conditions, and send it to your counterpart through any asynchronous communication channel like Email.

In routed payments, for example, A -> B -> C -> D if the Recipient node is offline if a route can be found to C, then the Sender A can communicate with C instead, requesting receipt from C (hash of a preimage), plus an IOU from C to D conditioned on the publishing of a preimage in a mutually trusted witness between A, C.

Once Sender A gets the preimage, it now has a valid IOU from C to D as well, and it can send it to the Recipient using any asynchronous communication channel.

The Recipient will then have access to the funds as soon as they are online. 

Assuming this is a tip, a donation, or some other sort of transaction that doesn't need an Invoice, the transaction is considered successful even if the Recipient never responds, as the Sender still has proof it was successful.
