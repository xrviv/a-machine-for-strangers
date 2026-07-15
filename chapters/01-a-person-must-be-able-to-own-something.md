---
layout: page
title: "Chapter 1: A Person Must Be Able to Own Something"
permalink: /chapters/01-a-person-must-be-able-to-own-something/
---

> **Fiction notice:** The fictional notebook below is an imagined reconstruction. It does not claim to report Nakamoto’s thoughts, identity, location, or motives.

## A fictional notebook, 2008

*Before a program can move value, I must decide what gives someone the right to move it.*

If Alice says the coins are hers, why should anyone believe her?

The obvious answer is an account. Keep accounts on one server. When Alice pays Bob, the server checks her balance and approves the move.

No. That only rebuilds a bank in a different shape. Someone still owns the list. Someone can refuse a payment, alter a record, or become the point everyone must trust. That cannot be the answer.

The proof must travel with the payment instead of living in an institution.

I need a secret that only Alice holds: a very large random number. I will call it a **private key**. It is not her name. It is not an account number. It is not something she gives away. With it, her software can make a mathematical mark on a payment—a **digital signature**.

There must also be something she can share. A **public key**. Other machines can use it to check her signature. They can establish that the matching private key authorized the payment, but they cannot reconstruct the secret from what they see.

Perhaps the payment simply carries the secret itself. The recipient can see that Alice truly controls it.

In this imagined program, that thought would look like this:

```cpp
payment.privateKey = alice.privateKey;  // Never do this
Send(payment);
```

No. That is not proof of one payment; it is a handover of everything the secret controls. The first recipient could spend Alice’s funds again. A secret used as proof must remain secret.

![Fictional illustration of an imagined programmer rejecting the idea of sharing Alice's private key with a payment.]({{ '/assets/images/content/ch01-nakamoto-private-key.png' | relative_url }})

*Fictional illustration: the private key must remain secret; a payment should carry proof of authorization, not the secret itself.*

The better construction is plain: Alice keeps the private key. The program uses it only to sign this specific payment.

```cpp
payment.signature = Sign(payment, alice.privateKey);
```

Everyone receives the payment and the signature, but not her private key. The network checks the signature against her public key. If it verifies, it has evidence that the holder of the secret authorized *this* payment—without learning who that holder is.

That is enough. I do not need to identify a person or ask an account manager. I need a rule every machine can check: **a valid signature authorizes a spend from the key that controls it.**

Now key creation is not a technical detail. The client must make the secret, preserve it after restart, share the public side when needed, and recognize incoming value controlled by its own keys.

## Evidence from the client

The following is source evidence, not part of the fictional notebook.

### First: the mathematical picture

A real Bitcoin private key begins as one integer—one whole number—chosen from an unimaginably large allowed range. On the `secp256k1` elliptic curve, that range is approximately:

```text
1 ≤ private key < 1.16 × 10^77
```

**Private number:** the secret whole number a wallet chooses unpredictably
from this allowed range. In elliptic-curve mathematics it is also called a
*private scalar*. A wallet uses it to create signatures; it must never be
shared. When this private number is stored and used as the secret half of a
key pair, we call it a **private key**.

That is a number with roughly 77 digits. For comparison, this is only a **toy** example, not a usable Bitcoin key:

```text
private key d = 42
public key P  = d × G
```

`G` is a publicly agreed starting point on a special mathematical curve. The `×` does not mean ordinary multiplication; it means repeatedly applying the curve’s point-addition rule. Going forward—from `d` and `G` to `P`—is practical for a computer. Reversing it—from `P` back to `d`—is designed to be infeasible.

In a real wallet, the program must choose `d` unpredictably. A person should not choose it from a birthday, a phrase, or a memorable number. The public key is then calculated from that private number.

### Try it: generate a real key pair

A private key is **chosen at random**, not calculated from someone’s public
key. This exercise performs the same kind of `secp256k1` key-pair generation
as the original client: it chooses a real 32-byte private number locally in
the browser and derives its real public key.

> **Safety:** The generated private number is real. Anyone who learns it can
> spend bitcoin sent to its matching public key or address. The page does not
> save or send it, but do not fund, reuse, or rely on a key displayed in a
> browser lesson.

<section aria-labelledby="toy-secret-heading" style="border: 1px solid #d1d5db; border-radius: 0.5rem; padding: 1rem; margin: 1.5rem 0;">
  <h4 id="toy-secret-heading">Generate a v0.1-style key pair</h4>
  <p id="toy-secret-help"><small>This runs locally using the browser’s cryptographic random-number source. It displays a real private scalar and matching public key for study.</small></p>
  <button id="toy-secret-button" type="button" aria-describedby="toy-secret-help" style="padding: 0.4rem 0.7rem;">Generate real key pair</button>
  <output id="toy-secret-output" aria-live="polite" style="display: block; background: #f6f8fa; border-radius: 0.25rem; padding: 0.75rem; margin-top: 0.75rem; white-space: pre-wrap; word-break: break-all;">Select “Generate real key pair” to trace the process.</output>
  <p id="toy-secret-explanation" aria-live="polite" style="color: #b42318; font-weight: 600;"></p>
</section>

<script>
  (function () {
    const button = document.getElementById('toy-secret-button');
    const output = document.getElementById('toy-secret-output');
    const explanation = document.getElementById('toy-secret-explanation');
    const field = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2Fn;
    const order = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141n;
    const generator = {
      x: 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798n,
      y: 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8n
    };

    function mod(value) {
      return ((value % field) + field) % field;
    }

    function inverse(value) {
      let previousRemainder = field;
      let remainder = mod(value);
      let previousCoefficient = 0n;
      let coefficient = 1n;

      while (remainder !== 0n) {
        const quotient = previousRemainder / remainder;
        [previousRemainder, remainder] = [remainder, previousRemainder - quotient * remainder];
        [previousCoefficient, coefficient] = [coefficient, previousCoefficient - quotient * coefficient];
      }
      if (previousRemainder !== 1n) throw new Error('Point has no inverse.');
      return mod(previousCoefficient);
    }

    function addPoints(first, second) {
      if (!first) return second;
      if (!second) return first;
      if (first.x === second.x && mod(first.y + second.y) === 0) return null;

      const slope = first.x === second.x && first.y === second.y
        ? mod(3n * first.x * first.x * inverse(2n * first.y))
        : mod((second.y - first.y) * inverse(second.x - first.x));
      const x = mod(slope * slope - first.x - second.x);
      return { x: x, y: mod(slope * (first.x - x) - first.y) };
    }

    function multiplyPoint(scalar) {
      let result = null;
      let addend = generator;
      let remaining = scalar;
      let additions = 0;
      let doublings = 0;

      while (remaining > 0n) {
        if (remaining % 2n === 1n) {
          result = addPoints(result, addend);
          additions += 1;
        }
        remaining = remaining / 2n;
        if (remaining > 0n) {
          addend = addPoints(addend, addend);
          doublings += 1;
        }
      }
      return { point: result, additions: additions, doublings: doublings };
    }

    function toHex(value) {
      return value.toString(16).padStart(64, '0');
    }

    function bytesToNumber(bytes) {
      let value = 0n;
      for (const byte of bytes) value = (value << 8n) | BigInt(byte);
      return value;
    }

    function randomPrivateNumber() {
      const bytes = new Uint8Array(32);
      let value;
      do {
        window.crypto.getRandomValues(bytes);
        value = bytesToNumber(bytes);
      } while (value === 0n || value >= order);
      return value;
    }

    function generateKeyPair() {
      if (!window.crypto || !window.crypto.getRandomValues) {
        output.textContent = 'This browser does not provide a secure random-number source.';
        explanation.textContent = '';
        return;
      }

      const secret = randomPrivateNumber();
      const derivation = multiplyPoint(secret);
      const publicPoint = derivation.point;
      const privateHex = toHex(secret);
      const publicHex = '04' + toHex(publicPoint.x) + toHex(publicPoint.y);
      const bits = secret.toString(2);
      const oneBits = bits.split('1').length - 1;

      output.textContent =
        '1. CKey() selects secp256k1.\n' +
        '   Curve: y² = x³ + 7 (mod p), where p is a 256-bit prime.\n\n' +
        '2. MakeNewKey() asks for randomness.\n' +
        '   Real private number d (32 bytes): ' + privateHex + '\n\n' +
        '3. Derive the public key: P = d × G.\n' +
        '   d has ' + bits.length + ' binary digits; the calculation used ' + derivation.doublings + ' doublings and ' + derivation.additions + ' point additions.\n' +
        '   Public point x: ' + toHex(publicPoint.x) + '\n' +
        '   Public point y: ' + toHex(publicPoint.y) + '\n\n' +
        '4. AddKey() would store the private side.\n' +
        '   This demonstration saves nothing; v0.1 stored OpenSSL’s DER private-key encoding in wallet.dat.\n\n' +
        '5. GetPubKey() returns the public side.\n' +
        '   Historical uncompressed SEC public key (65 bytes): ' + publicHex;
      explanation.textContent =
        'What just happened mathematically: this is scalar multiplication, not ordinary multiplication. The program writes d in binary. For every binary digit it doubles a curve point; whenever the digit is 1, it adds that point into the answer. Point addition uses high-school-style coordinate formulas, but arithmetic is done “mod p”: after each operation, take the remainder after division by p. The slope is (y₂ − y₁) ÷ (x₂ − x₁) mod p; division means multiplying by the one number that reverses multiplication modulo p. From that slope λ, the next point is x₃ = λ² − x₁ − x₂ mod p and y₃ = λ(x₁ − x₃) − y₁ mod p. Going forward is practical; recovering d from only P is the elliptic-curve discrete-logarithm problem, believed infeasible at Bitcoin’s key size. Keep the displayed private number private and never fund it.';
    }

    button.addEventListener('click', generateKeyPair);
  }());
</script>

The button follows the same conceptual path as the original client, using a
real `secp256k1` arithmetic at every stage:

1. `CKey()` selects `secp256k1`, as the original client does.
2. `MakeNewKey()` obtains random bytes and selects `d`, the private number.
   The original delegates this to OpenSSL; the exercise uses the browser’s
   cryptographic random-number source and rejects values outside the curve’s
   allowed range.
3. Scalar multiplication derives `P = d × G` on the actual curve and returns
   the 65-byte uncompressed public-key encoding used by the early client.
4. `AddKey()` would persist the private side and `GetPubKey()` returns the
   shareable public side. The exercise saves nothing.

### Try it in a terminal: make the client’s steps tangible

The browser exercise shows the result. This short terminal lab lets the reader
perform the same broad sequence on their own machine: select `secp256k1`, ask
OpenSSL to generate a key pair, then derive and inspect only its public half.
It uses a modern OpenSSL command-line interface, not the exact 2009 build, but
the curve and key-pair operation are the same kind used by the v0.1 client.

> **Safety:** Run these commands in a private local terminal. Do not paste the
> private PEM file, its contents, or any generated secret into chat, a website,
> source control, or a wallet. This is a study key: do not fund or reuse it.

**1. Make a private workspace, then limit new-file permissions.**

```bash
mkdir -p ~/bitcoin-key-lab
cd ~/bitcoin-key-lab
umask 077
```

`umask 077` asks the shell to create new files readable only by your user. It
does not create a key yet; it prepares the place where OpenSSL will write one.

**2. Select the real Bitcoin curve and generate the key pair.**

```bash
openssl ecparam -name secp256k1 -genkey -noout -out chapter-01-private.pem
stat -c '%A %n' chapter-01-private.pem
```

The first command corresponds most closely to the original client’s
`CKey()` plus `key.MakeNewKey()`: it selects `secp256k1`, obtains cryptographic
randomness, chooses a valid private number, and calculates its matching public
point. The second command should report restrictive permissions such as
`-rw-------`. It reports metadata only; it does not reveal the secret.

**3. Derive the public half and inspect it.**

```bash
openssl ec -in chapter-01-private.pem -pubout -out chapter-01-public.pem
openssl ec -pubin -in chapter-01-public.pem -text -noout
```

The first command derives and writes a shareable public-key file. The second
prints that public key’s curve name and coordinates. This is the terminal
analogue of `GetPubKey()`: it exposes information other people may verify
without exposing the private number.

**4. Compare that with the original client’s storage step.**

The terminal lab keeps the private material in a local PEM file only long
enough to derive its public half. In v0.1, `AddKey(key)` instead passed
`key.GetPubKey()` and OpenSSL’s DER-encoded `key.GetPrivKey()` to
`CWalletDB().WriteKey(...)`, which stored the key in the wallet database.

**5. Remove the study files when finished.**

```bash
rm -f chapter-01-private.pem chapter-01-public.pem
rmdir ~/bitcoin-key-lab
```

This removes the lab files from the normal filesystem view; it is not a claim
of forensic secure erasure on every drive type.

### The v0.1 key-generation path

The private number is not visibly constructed in `main.cpp`. The client delegates that cryptographic job to OpenSSL through its `CKey` class. First, the `CKey` constructor selects Bitcoin’s curve:

```cpp
pkey = EC_KEY_new_by_curve_name(NID_secp256k1);
```

Then `MakeNewKey()` asks OpenSSL to generate the new key pair:

```cpp
void MakeNewKey()
{
    if (!EC_KEY_generate_key(pkey))
        throw key_error("CKey::MakeNewKey() : EC_KEY_generate_key failed");
}
```

In plain language: OpenSSL chooses an allowed unpredictable private number and derives the matching public key on `secp256k1`.

`main.cpp` uses that lower-level machinery in this short function:

```cpp
vector<unsigned char> GenerateNewKey()
{
    CKey key;
    key.MakeNewKey();
    if (!AddKey(key))
        throw runtime_error("GenerateNewKey() : AddKey failed\n");
    return key.GetPubKey();
}
```

Walk it line by line:

1. `CKey key;` creates a key container configured for `secp256k1`.
2. `key.MakeNewKey();` delegates generation of the private number and matching public key to OpenSSL.
3. `AddKey(key)` saves the wallet’s key information; failure stops the operation.
4. `key.GetPubKey()` returns the shareable public key. It does not return the private key.

The original v0.1 client keeps separate collections for wallet transactions, private keys, and public-key information near the beginning of `main.cpp`:

```cpp
map<uint256, CWalletTx> mapWallet;
map<vector<unsigned char>, CPrivKey> mapKeys;
map<uint160, vector<unsigned char> > mapPubKeys;
```

Its key-generation function is deliberately direct:

```cpp
CKey key;
key.MakeNewKey();
if (!AddKey(key))
    throw runtime_error("GenerateNewKey() : AddKey failed\n");
return key.GetPubKey();
```

`AddKey` then records the key information in memory and asks the wallet database to persist it:

```cpp
mapKeys[key.GetPubKey()] = key.GetPrivKey();
mapPubKeys[Hash160(key.GetPubKey())] = key.GetPubKey();
return CWalletDB().WriteKey(key.GetPubKey(), key.GetPrivKey());
```

When the client encounters a transaction, it has a compact test for whether that transaction belongs in its wallet:

```cpp
if (tx.IsMine() || mapWallet.count(tx.GetHash()))
```

## Did Nakamoto invent this key-generation method?

No—not the underlying cryptography. The surviving source cannot tell us how
Nakamoto learned it or whether he independently worked through its mathematics.
It does show that the client used an existing OpenSSL interface:
`EC_KEY_generate_key()` on the already named `NID_secp256k1` curve.

The method had a long public history before Bitcoin. Victor Miller proposed
using elliptic curves for cryptography in 1985, and Neal Koblitz published an
independent elliptic-curve cryptosystem proposal in 1987. ECDSA was already
part of public signature-standard work by 2000. The exact family of parameters
Bitcoin uses—`p`, the curve equation, base point `G`, order `n`, and cofactor
`h`—was published as `secp256k1` in the Standards for Efficient Cryptography
Group’s 2000 SEC 2 document.

So this part of the client is best understood as integration, not invention:
select a published curve, let a mature cryptographic library generate a secret
scalar and matching public point, then connect that key pair to Bitcoin’s
transaction, wallet, database, and peer-to-peer rules. That conclusion does
not settle which broader parts of Bitcoin’s design Nakamoto originated; it
only separates established public-key cryptography from the way this client
applied it.

## What this supports—and what it does not

This code supports a modest conclusion: the original client was designed to create and persist its own wallet keys, then recognize transactions controlled by those keys. That is the software foundation for a user-held form of ownership.

It does **not** reveal Nakamoto’s private motives, and it does not prove that this was written before networking, mining, or validation. The early client contains all of those systems. This chapter is a guided reading of one part of it.

## Source trail

- Bitcoin v0.1, `bitcoin0.1/src/main.cpp`: global wallet/key state; `AddKey`; `GenerateNewKey`; `AddToWalletIfMine`.
- Bitcoin v0.1, `bitcoin0.1/src/key.h`: `CKey`, `NID_secp256k1`, `MakeNewKey`, public/private-key serialization, signing, and verification.
- Bitcoin v0.1, `bitcoin0.1/src/db.cpp`: the Berkeley DB storage layer used by the wallet.
- Victor Miller, [“Use of Elliptic Curves in Cryptography” (1985)](https://doi.org/10.1007/3-540-39799-X_31).
- Neal Koblitz, [“Elliptic Curve Cryptosystems” (1987)](https://doi.org/10.1090/S0025-5718-1987-0866109-5).
- [NIST FIPS 186-2, *Digital Signature Standard* (2000)](https://csrc.nist.gov/pubs/fips/186-2/final).
- [SECG, *SEC 2: Recommended Elliptic Curve Domain Parameters*, version 1.0 (2000)](https://www.secg.org/SEC2-Ver-1.0.pdf).
- [Satoshi Nakamoto Institute: Bitcoin source-code archive](https://satoshi.nakamotoinstitute.org/code/)

[← Prologue: Autumn 2008]({{ '/chapters/00-autumn-2008/' | relative_url }})
