---
title: 'GSoC Proposal'
categories: gsoc
---

This is my GSoC proposal for this year with which I applied today.
On the 22. April, I will find out if I'm accepted.

# PGP-encrypted lists for Mailman 3, 2016 #

## Synopsis ##

The goal of this project is to write a Mailman 3 plugin that enables
the listserver to send and receive PGP-signed and -encrypted mail.

To achieve this, a mailinglist acts as a re-encryption gateway with its own
private keypairs and a collection of trusted subscribers public keys.
PyGPGME provides the functions required for this task.
The steps of decryption, signature validation, signature and encryption take
place as handlers in PGP-enabled pipelines that replace the standard pipelines on
PGP-enabled lists.

## Proposal Description ##

Email is a protocol known to be prone to manipulation and eavesdropping, for
example by service operators or anyone with access to the plain network
communication.[^empiry]
Although mail transfer traffic may be encrypted using SSL, mail servers still
have access to plaintext mail communication because SSL only protects the mails
while being transferred from one server to the next.

[^empiry]: An Empirical Analysis of Email Delivery Security, <http://conferences2.sigcomm.org/imc/2015/papers/p27.pdf>

The goal of this Project is to allow Mailman 3 to host PGP-enabled lists where
the integrity and confidentiality of mail from the listserver to subscribers and
from composers to the listserver is secured using GnuPG.

This could protect list communication even in cases where plain SMTP is used,
where a mail is transported over a malicious or insecure mail server or when an
eavesdropper gains access to a subscriber's inbox because the mail is encrypted
on the composers computer and not decrypted before it is received by Mailman.
Mailman will encrypt it for each of the subscribers and it can only be decrypted
with their respective PGP keys which they store on their computers. Similarly,
the mail is *"end-to-end"* signed with a member on one end and the
listserver that processes it on the other – and vice-versa. Therefore, not even a
mail server that transports the signed mail could alter its contents without
breaking the signature as long as it has no access to the senders private PGP key.
Neither could it access the message plaintext without having access to the
recipient's private PGP key.

Confidentiality of the system can be broken if an attacker gains access to the
listserver, to decrypted mails from subscribers or to the private PGP key
of the listserver or any of the subscribers. Also, it is crucial that the exact
public keys used for encryption and signature validation are hand authorized by
the list administrator or a different authority to guarantee that they in fact belong to
that real person to be privy to list communication. Therefore, an Interface for
keyring management is required.

An example to illustrate the possibilities and limits of this list encryption
approach[^practical]:

Assume Alice and Bob are working in software development and they have a private
mailinglist for security critical issues in their software.

Eve is a black-hat hacker and she wants to gain access to the list communication
to exploit the discussed security vulnerabilities.
This are some examples how she could access the list communication:

 1. Eve steals Alice's computer on which she stores local copies of
   her mail
 2. She gains access to the listserver and manipulates it to send her copies of
   all the mails when they are processed
 3. Bob has to send an urgent mail to the list from a public wifi that doesn't allow
   him to protect his mail communication using SSL. Eve can use packet sniffing
   to eavesdrop on the mail
 4. Eve gains access to a mailserver that transports mail of this list
 5. Eve sends an email to the list that is manipulated to look like it was sent
   by Alice, asking Bob to send a list of vulnerabilities to Eve.
 6. Eve offers Bob money in exchange for him to reveal the mails

In some of the cases, the confidentiality of the communication could have been
protected if a re-encrypting PGP gateway had been used:

 1. Usually, Alice's PGP-enabled mail client would encrypt her local mail copies with her
    private key which she can symmetrically encrypt with a password.
    The PGP-encryption would encourage subscribers to encrypt their local mail
    storage this way.
 2. The PGP-list feature would offer no protection in this case, the listserver has to
    be protected against illicit access.[^mod]
 3. Eve would not be able to eavesdrop on the plaintext message bodies sent through a
    PGP-enabled list by accessing the plain SMTP traffic.
 4. As of now, mailservers have full access to all the listmail they relay. If PGP
    was used, they could only access the mail headers, the body would be
    encrypted.
 5. The signature validation feature of the PGP-enabled listserver would foil
    a such attack because Eve would not be able to sign the forged mail with
    Alice's private key which she safely keeps on her computer.
 6. The proposed plugin could not prevent subscribers to reveal the mails however, with the
    re-encryption gateway approach Bobs public key can be removed from
    the keyring more easily than with other systems and future mails on the list won't be encrypted for him.


[^practical]: Practical Encrypted Mailing Lists by Neal H. Walfield, feb 2016 <http://hssl.cs.jhu.edu/~neal/encrypted-mailing-lists.pdf>
[^mod]: Implementing the PGP-enabled listserver as a re-encryption gateway gives the listserver access to the list communication but other methods would require modifications to the subscribers' mail clients.

## Implementation ##

PyGPGME could provide all cryptographic functions required for this project.
It offers easy access to the popular PGP implementation GnuPG in python.
PyGPGME, GpgME and GnuPG are under GNU Licenses.

In detail, for the core functionality of sending and receiving encrypted and
signed messages,
[`gpgme.Context.decrypt`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.decrypt)
and
[`gpgme.Context.encrypt`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.encrypt)
in combination with
[`gpgme.Context.armor`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.armor)
can be used to decrypt the ACII-armored cyphertext in the body of incoming
messages with the lists private key and encrypt outgoing mail using the public
key of the respective list member. The proper keys/keypairs can be selected
from the keyring using
[`gpgme.Context.get_key`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.get_key).

For signature, the [`gpgme.Context.sign`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.sign)
and [`gpgme.Context.verify`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.verify)
methods are available. The [`gpgme.Signature`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Signature)
class then stores the relevant signature information.

PyGPGME also offers functions to store – in this case the lists' – own PGP keypairs and the public keys
of contacts – subscribers in this case – in a local keyring.
This includes for example importing keys using
[`gpgme.Context.import_`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.import_),
searching for keys using
[`gpgme.Context.keylist`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Context.keylist)
and inspecting a key using the
[`gpgme.Key`](https://pygpgme.readthedocs.org/en/latest/api.html#gpgme.Key) class.

It also includes an experimental interface to manage the level
of trust given towards the public key of a contact.[^trust]
However, it might be convenient to store information about the used PGP keys and their granted level of trust in a
database instead although this could hinder the importation of existing GnuPG keyrings.

SQLAlchemy is a database library for python, is already used in Mailman
and can provide database functionality for my plugin if required.

[^trust]: The ‘GnuPG Made Easy’ Reference Manual: Trust Item Management, <https://www.gnupg.org/documentation/manuals/gpgme/Trust-Item-Management.html#Trust-Item-Management>

The features new to Mailman 3, *chains* and *pipelines*[^chains], can be used as interface
for this plugin. For each of the tasks of encryption, decryption, signature and validation, a handler could be implemented that performs the task
using PyGPGME.

[^chains]: Developing Mailman, <https://pythonhosted.org/mailman/src/mailman/docs/DEVELOP.html>

The PGP-plugin could present itself as a set of alternative pipelines – most
importantly a PGP-enabled version of the *PostingPipeline* – that replace the default pipelines for
PGP-enabled lists.

The *PGP-PostingPipeline* could for example contain these handlers in this order:

`tagger` → `member-recipients` → `cleanse` → `cleanse-dkim` → `cook-headers
→ subject-prefix` → `rfc-2369` → `pgp-decrypt` → `pgp-check-signature` → `decorate
→ pgp-decorate` → `pgp-sign` → `pgp-send` → `after-delivery` → `acknowledge`

It should not contain any archivers.

After the headers where processed, the *pgp‑decrypt* handler could decrypt
the incoming mail using PyGPGME and the lists keypair. If this failed for some
reason, for example because the mail wasn't encrypted, an internal header could
be set to prevent the mail from being distributed or further processed.
In such cases, a notification could be sent to the list administrator or the composer.

The *pgp‑validate* handler could verify if an incoming mail was properly
signed using a trusted PGP identity and also set an internal flag if not.

The *pgp‑decorate* handler could add a footer to the mail that
would contain information about the signature (e.g. that it was signed and the
public key id and trust level of the keypair it was signed with).

A *pgp-sign* handler could sign the message, including the new header and footer,
using the lists private PGP key.

Finally, a *pgp‑send* handler could, in case that all the prior PGP operations
succeeded, using PyGPGME, encrypt the message for each recipient that has a
trusted public key in the keyring and send it. The *pgp‑send* handler should
also clear the message body from the pipeline.

To allow list administration to add trusted members' public keys, an interface for
the management of the lists keyring would be required.

Public keys could be collected automatically, for example from subscription
requests or keyservers, however, in any case will the plugin ignore public keys
that don't have their trust level manually set to be trusted. This has
to be done by a list administrator or an other cautious list member,
ideally by exchanging list fingerprints over a secure channel or in person.
If the GnuPG trust levels
are used as described above, keyrings could be imported from any GnuPG
installation, managed using the GnuPG *edit‑key*[^edit-key] command line tool
and maintained by an administrator on the listserver like any local GnuPG keyring
on any GnuPG installation.

Additionally, as list communication would then be secured it would be possible to
develop a mail-command interface with a command set similar to GnuPGs *edit-key*.
For this mail interface, an administrator could identify using either
his PGP signature or the administrative password which would, along with the mail be PGP
encrypted. It would also be possible to allow list members who have their trust
level set by the list administrator to *Ultimately trusted* to enable
a public key for that list by setting its trust level to *Fully trusted* using
a PGP signed mail command.

[^edit-key]: The GNU Privacy Handbook, edit-key, <https://www.gnupg.org/gph/en/manual/r899.html>

## Documentation ##

Two documentations should be created for this software, a user- and a reference
manual. Also, my GSoC blog will document the process of development.

The user manual instructs users
how to install and use the software and addresses security considerations.
It should cover as many security precautions to be taken
and potential security pitfalls as possible while being compact enough to
serve as an introduction for listserver administrators that want to install the
plugin. It should also cover the relevant configuration options and user
interfaces.

The reference manual can be generated from sourcecode internal *docstrings*.
Its purpose is to help users to examine the source code and
it might help developers to join the project or to write similar
Mailman 3 plugins.

## Project Goals ##

The core functionalities of the proposed plugin are:

 * Storage of the subscribers' public keys as well as the lists own keypair
 * A simple interface for list administrators to manage these public keys
 * Encryption and signature of outgoing mail with the respective keys
 * Decryption of incoming mail and signature validation
 * Prevention of unencrypted/unsigned communication
 * Documentation of the plugin that covers the security aspects

Possible stretch goals are:

 * Further protection against unencrypted copies of the listmails on the server
 * Compatibility with anonymous lists
 * Administration mail command interface
 * Postorius integration of administration features
 * Attachment of signature Information to distributed list mails
 * A mode of operation that makes pgp usage optional for the transition of
    existing lists to PGP-enabled lists
 * Notification emails when an email didn't pass the PGP chain