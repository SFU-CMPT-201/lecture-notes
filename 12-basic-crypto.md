# Basic Cryptography

* Cryptography is a broad area and here we introduce the basics of cryptography, focusing on the
  application of it.
* This is not even scratching the surface but really the very basics.

## The CIA Model

* The CIA model is the classic security model.
    * C stands for *c*onfidentiality.
    * I stands for *i*ntegrity.
    * A stands for *a*vailability.
* Confidentiality: information is only disclosed to those authorized to know it.
* Integrity: only modify information in allowed ways by authorized parties.
* Availability: those authorized for access are not prevented from it.
* Threat examples
    * Against confidentiality: classified information leak
    * Against integrity: fake images/videos
    * Against availability: Denial-of-Service (DoS) attacks

## Cryptography

```bash
            encryption                decryption
plain text ----------->  cipher text -----------> plain text
```

* Comes from Greek word meaning "secret"
* Cryptographers invent secret codes to attempt to hide messages from unauthorized observers
* Modern encryption:
    * Algorithms are public.
    * Keys are secret and provide security
    * May be symmetric (secret) or asymmetric (public). More below.
* Cryptographic algorithms goal:
    * Given a key, it should be relatively easy to compute.
    * Without a key, it should be hard to compute (invert).
    * The strength of security often based on the length of a key. The longer the key is the more
      difficult to guess (by brute-force).
* Window of validity
    * The minimum time to compromise a cryptographic algorithm.
    * Problem: it can be shorter than the lifetime of your system.
    * Example
        * SHA-0 was published in 1993.
        * A possible weakness was found in the algorithm and replaced in 1995 with SHA-1.
        * A way to compromise SHA-0 was published in 2004.
        * A way to compromise SHA-1 was published in 2017.
    * A system designer needs to be prepared to update their crypto function, perhaps more than
      once.

## Three Types of Cryptography

* Types
    * Cryptographic hash functions: zero keys
    * Secret-key functions: one key
    * Public-key functions: two keys
* Cryptographic hash functions
    * Suppose we have a cryptographic hash function `h()`.
    * It takes a message `m` of arbitrary length and produces a smaller (short) number `h(m)`.
    * Properties
        * It should be easy to compute `h(m)`.
        * One-way function: given `h(x)`, it should be difficult to find `x`. In other words, the
          reverse of `h()` should be difficult to compute.
        * Weak collision resistance: given `x`, it should be difficult to find `x'` where `h(x') ==
          h(x)`. In other words, given a value and a hash function, it should be difficult to find
          another value that produces the same hash.
        * Strong collision resistance: it should be difficult to find two messages `x` and `x'`
          where `h(x) == h(x)`. In other words, given a hash function, it should be difficult to
          find two values that produce the same hash.
* Private key crypto (or symmetric key crypto)

  ```bash
              encryption                    decryption
              with shared key               with shared key
  plain text ----------------> cipher text ----------------> plain text
  ```

    * One key is shared between encryption and decryption.
    * This assumes that there is a way to shared the secret key in a secure fashion.
    * This was the only type of encryption prior to invention of public-key in 1970's.

* Public key crypto (or asymmetric key crypto)

  ```bash
              encryption                 decryption
              with pub key               with priv key
  plain text -------------> cipher text --------------> plain text
  ```

    * There are two keys.
        * Public key: can be known to anybody, used to encrypt and verify signatures (more below).
        * Private key: should be known only to the owner of the key, used to decrypt and sign
          signatures (more below).
    * For example, if Alice wants to send a secret message to Bob, Alice can use Bob's public key to
      encrypt the message and Bob can use his private key to decrypt the message. Since Bob should
      be the only one who has Bob's private key, Alice and Bob can securely share the message.
    * This does not require having a secure key distribution mechanism.

## Applications of Cryptography

* Storing passwords
    * Password systems don't store plain text passwords.
    * All passwords are hashed and only the hashes are stored. This is for added security. Even if a
      password file gets compromised, plain text passwords are not revealed.
    * When checking a password, check `hash(typed) == stored hash`.
    * In addition, a *salt* is often used when hashing: `hash(input || salt)`.
    * A salt is effectively a random number added to input.
    * It is stored together with the generated hash.
    * This avoids pre-computation of all possible hashes in "rainbow tables" (available for download
      on the Internet). With a salt, pre-computation is not possible.
* Secure digest
    * This is often used to verify a downloaded file (as also discussed in A13).
    * A secure digest is a summary of a message.
    * A fixed-length that characterizes an arbitrary-length message.
    * This is typically produced by a cryptographic hash function, e.g., SHA-256.
* Digital signature
    * A digital signature combines public key crypto and hashing.
    * It verifies a message or a document is an unaltered copy of one produced by the signer.
    * This is not about message encryption, i.e., message itself doesn't need to be secret.
    * There are two parties involved, a signer and a verifier.
        * The signer is the one who sends a message and wants to prove that the message is indeed
          sent by the signer.
        * The verifier is typically the message recipient and verifies that the message was indeed
          sent by the signer.
    * Signer (writing a document, `m`)
        * Computes a message digest: `h(m)` (e.g., using SHA-256)
        * Encrypts the digest with the private key: `enc(h(m))` (e.g., using RSA public key crypto)
            * This is called *signing*. It's because the signer is the only one who can do this
              (assuming that the signer is the only one who has the private key).
        * Sends the message & the signature: `<m, enc(h(m))>`
    * Verifier
        * Receives the message and the signature: `<m, enc(h(m))>`
        * Takes the message `m` and computes a message digest: `h(m)`. Let's call it `h'`.
        * Decrypts the signature with the signer's public key: `dec(enc(h(m)))`. This is `h(m)`.
        * Verifies the equality: `h(m) == h'`. This verifies that the signer has indeed "signed" the
          message as the verifier uses the signer's public key. Again, the signer is the only one
          who can use the private key to encrypt.
* Digital certificate
    * Digital certificates use digital signatures.
    * When you use your browser to go to Instagram, how do you know if you're really communicating
      with Instagram?
    * If you use HTTP, there is no guarantee.
    * However, if you use HTTPS, Instagram will send you its public key as well as something called
      a *digital certificate* that proves that the public key indeed belongs to Instagram.
    * Your OS verifies the authenticity of the digital certificate.
    * After that, your browser uses Instagram's public key to encrypt messages and communicate with
      Instagram. This way, only Instagram can decrypt received messages (assuming that only
      Instagram has its private key).
    * Let's look at how digital certificates work more closely.
* How digital certificates work:
    * A digital certificate is signed by a *digital certificate authority*. VeriSign is a well-known
      one, and we will use the company as an example. Let's also use Instagram once again as an
      example. The following is how digital certificates work.
    * Instagram first goes to VeriSign, gives its public key, and requests a digital certificate.
      VeriSign prepares a document that says "this public key belongs to Instagram" and signs the
      document with VeriSign's own private key. This document and the signature is a digital
      certificate.
    * This digital certificate is essentially saying that "VeriSign proves that the public key on
      this document belongs to Instagram."
    * VeriSign then sends the digital certificate to Instagram.
    * When your browser connects to Instagram, it sends the digital certificate.
    * Now, the digital certificate contains VeriSign's signature. As long as you verify this
      signature, you can be assured (i) that the document was indeed signed by VeriSign and (ii)
      that the public key belongs to Instagram.
    * In order to verify VeriSign's signature, you need VeriSign's public key. (Refer to the
      digital signature discussion above if you need a refresher.)
    * For well-known digital certificate authorities like VeriSign, OS vendors (e.g., Microsoft,
      Apple, etc.) have trust relationships. Thus, their public keys are shipped with an OS. For
      example, Windows, MacOS, Android, iOS, etc. all have public keys of VeriSign.
    * Thus, your OS can verify VeriSign's signature using VeriSign's public key shipped with the OS.
    * If the OS is not compromised, this whole process is secure. If the OS is indeed compromised,
      this whole process is not secure.
* Chain of trust and the root of trust
    * In the above scenario, we rely on the chain of trust as follows.
    * In order to trust the public key sent by Instagram, we need to trust VeriSign's signature.
    * In order to trust VeriSign's signature, we need to trust VeriSign's public key shipped with
      our OS.
    * In order to trust VeriSign's public key, we need to trust that our OS is not compromised.
    * This is a *chain of trust* and as long as the end of the chain (the *root of trust*) is
      trustworthy, all is trustworthy. The root of trust in our scenario is the OS.
