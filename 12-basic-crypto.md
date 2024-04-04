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

* Comes from Greek word meaning "secret".
* Cryptographers invented secret codes to hide messages from unauthorized observers.
* Modern encryption:
    * Algorithms are public.
    * Keys are secret and provide security.
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
    * Ideally, you should get all three properties for a strong cryptographic hash function.
      However, not all hash functions provide all three properties.
* Private key crypto (or symmetric key crypto)

  ```bash
              encryption                    decryption
              with shared key               with shared key
  plain text ----------------> cipher text ----------------> plain text
  ```

    * One key is shared between encryption and decryption.
    * This assumes that there is a way to share the secret key in a secure fashion.
    * This was the only type of encryption prior to invention of public-key in 1970's.

* Public key crypto (or asymmetric key crypto)

  ```bash
  (This is an example only regarding encryption/decryption. There are other use cases.)

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
    * This is often used to verify a downloaded file.
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
    * When your browser connects to Instagram, Instagram sends the digital certificate.
    * Now, the digital certificate contains VeriSign's signature. As long as you verify this
      signature, you can be assured (i) that the document was indeed signed by VeriSign and (ii)
      that the public key belongs to Instagram.
    * In order to verify VeriSign's signature, you need VeriSign's public key. (Refer to the
      digital signature discussion above if you need a refresher.)
    * For well-known digital certificate authorities like VeriSign, OS vendors (e.g., Microsoft,
      Apple, etc.) have trust relationships. Thus, their public keys are shipped with an OS. For
      example, Windows, MacOS, Android, iOS, etc. all have public keys of VeriSign as well as other
      digital certificate authorities.
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
      trustworthy, all is trustworthy. The *root of trust* in our scenario is the OS.

## Activity: SSH Authentication with Public Key Cryptography

One popular application of public key cryptography is SSH authentication/login. You might have done
this before for other classes, but with our class discussion, you can now understand what is going
on.

* First, log in to a CSIL machine from our VM. Note that if you're not on campus, you need to first
  use the VPN on your host (*not* on our VM) to be able to log in.
* You can use one of the [Linux CPU
  servers](https://www.sfu.ca/computing/about/support/csil/unix/how-to-use-csil-linux-cpu-server.html).
* Create a new ssh public/private key pair. We will use an algorithm called *ed25519*. The command
  is as follows.

  ```bash
  $ ssh-keygen -t ed25519 -C "your email address"
  ```

    * Replace `your email address` with an actual email address of yours. For example,

      ```bash
      $ ssh-keygen -t ed25519 -C "steveyko@sfu.ca"
      ```

    * You can use the default file location by pressing `<Enter>`.
    * Choose a passphrase and remember it. This will be used when you access your private key.
* The above command creates two files under `.ssh/`.
    * One file is `id_ed25519` and the other is `id_ed25519.pub`.
    * `id_ed25519` contains your private key. Try `cat ~/.ssh/id_ed25519` and check the content. It
      should start with `-----BEGIN OPENSSH PRIVATE KEY-----` which indicates that it is a private
      key.
    * `id_ed25519.pub` contains your public key. Try `cat ~/.ssh/id_ed25519.pub` and check the
      content. It should start with `ssh-ed25519`, indicating the algorithm.
* As discussed above, you can share the public key with anybody. However, you need to protect the
  private key as your secret.
* SSH key pairs are typically used for SSH authentication. This is a different way of authenticating
  yourself to a server than using a password.
* The way it works is the following. On the server machine (i.e., the machine that you want to SSH
  in to), you can add your public key to the `authorized_keys` file. This file is located in the
  `$HOME/.ssh/` directory in your home directory (on the server machine). The `authorized_keys` file
  contains a list of public keys that are allowed for public/private key pair authentication. On the
  client side, you need to have your private key under the `$HOME/.ssh/` directory (on the client
  machine), which proves to the server that you are the owner of the public key.
* We will use our CSIL Linux machines to understand this further, but it requires some explanation
  first.
* A special thing about our CSIL Linux machines is that the machines all share the same home
  directory. Thus, no matter which CSIL machine you log in to, you will see the same files in your
  home directory.
* What this means is that if you add your public key to the `$HOME/.ssh/authorized_keys` file from
  one machine, this is all shared. Thus, you can log in to all other CSIL Linux machines from your
  client machine as long as you have your private key on your client machine.
* Now, a CSIL Linux machine doesn't have to be just an SSH server. You can use one CSIL Linux
  machine as an SSH client and log in to another CSIL Linux machine as an SSH server. Thus, you can
  have a private key on an CSIL Linux machine and log in to another CSIL Linux machines.
* However, since all CSIL Linux machines share the same home directory, a private key on one CSIL
  Linux machine is shared with all other CSIL Linux machines as well. Thus, as long as you have your
  private key on one CSIL Linux machine, you can log in from any CSIL Linux machine to any other
  CSIL Linux machine. This is because both the private key and the `authorized_keys` file are
  shared.
* Since you created a public/private key pair above, you only need to add your public key to the
  `authorized_keys` file on a CSIL Linux machine. You can use the following command, which appends
  your public key to the `authorized_keys` file.

  ```bash
  $ cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
  ```

  With this, your private key and your `authorized_keys` file are all shared across all CSIL Linux
  machines. You can now use any of the CSIL Linux machines as an SSH client and log in to any other
  CSIL Linux machines as an SSH server.
* To try out SSH authentication, pick another CSIL Linux CPU server and ssh into it. For example,

  ```bash
  $ ssh -p 24 csil-cpu08.csil.sfu.ca
  ```

* If this is your first time logging in to the server, you need to type `yes` to add the server to
  the known host list.
* After that, it will ask you to type your *passphrase* instead of your password. The passphrase is
  the same passphrase that you entered for your private key.
* Typically, one uses this in conjunction with `ssh-agent` and `ssh-add` to skip your passphrase
  typing for easy authentication.
    * [This short article](https://kb.iu.edu/d/aeww) describes the process of using `ssh-agent` and
      `ssh-add`. In a nutshell, you run `ssh-agent` and then `ssh-add` to keep your private key and
      the passphrase in memory. `ssh-agent` will run in the background and use your private key and
      the passphrase to authenticate all SSH logins. When you're done using `ssh` (or `scp`), you
      need to kill `ssh-agent` so that it removes your private key and the passphrase from memory.

## Activity: Collisions in Cryptographic Hash Functions

Above, we discussed the possibility of collisions in cryptographic hash functions as well as "window
of validity" where a cryptographic hash function is considered secure (i.e., still collision
resistant and/or still difficult to reverse). A well-known example of a popular cryptographic hash
function that is no longer considered secure is [MD5](https://en.wikipedia.org/wiki/MD5), which was
once universal but no longer used in applications that require a secure cryptographic hash function.
There is a good description of this history on
[Wikipedia](https://en.wikipedia.org/wiki/MD5#Collision_vulnerabilities).

It used to be the case that MD5 was the most popular choice for secure digest (discussed above), but
not anymore. For example, if you look at Ubuntu 14.04.6, which is one of the older versions of
Ubuntu Linux, [its download page](https://releases.ubuntu.com/trusty/) shows all the downloadable
files as well as the hashes from various algorithms, including MD5, SHA1, and SHA256. Click the link
and see for yourself. However, if you look at a more recent version, e.g., [Ubuntu 20.04.6's
download page](https://releases.ubuntu.com/20.04.6/), it only shows a hash for SHA256. Also click
the link and see for yourself. This is because MD5 is no longer considered secure. (And actually
SHA1 is no longer considered secure either.)

Let's try a simple example as an activity to examine this further.

The following example comes from [an article on MD5
collisions](https://natmchugh.blogspot.com/2015/02/create-your-own-md5-collisions.html). It shows
two images that generate the same MD5 hash. First take a look at the images and see for yourself how
different they are: the [first image](https://s3-eu-west-1.amazonaws.com/md5collisions/ship.jpg) and
the [second image](https://s3-eu-west-1.amazonaws.com/md5collisions/plane.jpg).

Then run the following commands to download those images and generate a MD5 hash for each.

```bash
$ wget https://s3-eu-west-1.amazonaws.com/md5collisions/ship.jpg
$ wget https://s3-eu-west-1.amazonaws.com/md5collisions/plane.jpg
$ openssl dgst -md5 ship.jpg
$ openssl dgst -md5 plane.jpg
```

The outputs should be identical. What this means is that theoretically, if you used MD5, downloaded
a file from a website, and verified it, it could still be a file different from the file you
intended to download.

The command we used (`openssl`) is a popular command for cryptographic functions including hashing,
encryption, and decryption. `dgst` means that we want to generate a *digest*, which is another term
commonly used to refer to a hash. Let's use a different algorithm and check the output.

```bash
$ openssl dgst -sha256 ship.jpg
$ openssl dgst -sha256 plane.jpg
```

It should give you two different outputs as we're using a different algorithm, SHA256, not MD5, that
doesn't create a collision for the given files.
