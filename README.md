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

Otherwise the Recipient will send the preimage to the Witness.

If the Witness is honest, it publishes the preimage in the log before the timeout (defined by block number), in which case the Recipient will have a valid IOU, and the Sender will have a reciept of the payment.

Same will apply to B, and C, either by watching the log themselves, or by getting the proof from the Recipient and passing it up the chain.

However, the Witness can still disrupt the transaction, either maliciously or out of incompetence, especially if the nodes don't gossip the log, and only directly poll the Witness API themselves:

1. **Bad clock** 
- The Witness may slow down creating (or publishing) blocks, so despite the inclusion of the preimage _before_ the timeout block, it is seen by the one or all participants _after_ the timeout period they intended.
- The Recipient gets a valid payment, and the Sender gets a valid receipt however it maybe too late to use that reciept as intended.
- According to the specific case, either the Sender or the Reciever will effectively lose money because of this distruption, and won't trust the Witness again.

2. **Censorship**
- The Witness does NOT publish the preimage in the public log.
- The log advances, adding more blocks until the timeout block is reached.
- The Recipient can't prove any malice, but it will not trust the Witness anymore, on the basis of incompetence, or censorship.

3. **Duplicity**
- It is possible that the Witness colludes with the Sender, by showing the Recipient a proof of inclusion of the preimage (the condition of the validity of the IOU), but publicly publish another fork that excludes the preimage until the timeout block.
- Now the Recipient serves the Sender thinking that they now have a valid IOU from their direct counterpart C.
- As soon as the Recipient tries to "collect" the IOU from C, C can offer the exclusion proof, and the IOU is considered invalid.
- However, now both the Recipient and C have a proof of duplicity, and can use it to prove to everyone involved, and the wider network that that log should never be trusted again.

Put in other words, users can lose value by the incompetence, censorship, but they will lose the users' trust. The log can't steal funds, but it can help the Sender steal, risking a fatal blow to its repuatation.

Users can easily detect censorship and bad service, but for duplicity, the need to actively communicate and compare their prespectives of the log.

### Watchtowers

Duplicity proofs are objective, and can be demonstrated by anyone to anyone, but it is probably impracticle to expect all users to gossip these proofs.

Watchtowers are proposed repositories, that users can query or submit duplicity proof to.

These Watchtowers in turn, can either federate, or gossip using more p2p approaches.

Who would want to run a watchtower?

1. Competing Witnesses
2. Wallet developers
3. Exchanges
4. Markets
