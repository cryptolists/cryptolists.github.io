---
layout: post
title: 'Thoughts on List Signature'
categories: concept
---

Why the PGP-enabled mailserver should replace mail signatures.

In my [Google Summer of Code Proposal](http://cryptolists.github.io/gsoc/2016/03/25/gsoc-proposal.html),
I described how the Mailman3 PGP plugin would act as as re-encryption gateway. I also proposed a
*PGP-PostingPipeline*-pipeline that would validate the subscribers signature and replace it with a
footer containing information about the original signature before signing the message with the lists
own keypair:

… → `pgp-decrypt` → `pgp-check-signature` → `decorate` → `pgp-decorate` → `pgp-sign` → `pgp-send` → `after-delivery` → …

This mode of operation is not self-evident. As an alternative, the original signature could be kept and subscribers could verify the signature using the composers public PGP-key.
The send pipeline could look like this:

… → `pgp-decrypt` → `pgp-check-signature` → `pgp-send` → `after-delivery` → …

Here is a quick comparison of the advantages of each method:

|End-to-End Signature|List-to-Subscriber Signature
|--------------------|----------------------------
|The listserver can't break mail integrity|Subscribers could get the impression that encryption was also end-to-end.
|Subscribers could notice more easily if the admin accidentally accepted an impersonator to the list|Subscribers would not have to keep track of other members keys to validate signatures
|Mailman passes PGP signatures End-to-End at the moment|Decorations, digests etc. would be possible

Most notably would end-to-end signature, in comparison to list/subscriber signature introduce a new
attack scenario – it would not be possible for a recipient to validate that an encrypted mail they
received really passed the encrypted list or if it was forged, signed with a signature controlled
by the attacker and encrypted using the public key of a recipient. This way, any attacker could pretend
to any subscriber that he as well was a list member.

To prevent this, every subscriber would have to keep a collection
of the trusted subscribers themselves. This wouldn't be practical.

For these reasons the PGP-enabled mailserver should remove the composers signature after validation
and replace it with its own by default although both modes of operation may have their uses.