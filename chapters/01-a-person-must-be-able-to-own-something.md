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

That is a number with roughly 77 digits. For comparison, this is only a **toy** example, not a usable Bitcoin key:

```text
private key d = 42
public key P  = d × G
```

`G` is a publicly agreed starting point on a special mathematical curve. The `×` does not mean ordinary multiplication; it means repeatedly applying the curve’s point-addition rule. Going forward—from `d` and `G` to `P`—is practical for a computer. Reversing it—from `P` back to `d`—is designed to be infeasible.

In a real wallet, the program must choose `d` unpredictably. A person should not choose it from a birthday, a phrase, or a memorable number. The public key is then calculated from that private number.

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

## What this supports—and what it does not

This code supports a modest conclusion: the original client was designed to create and persist its own wallet keys, then recognize transactions controlled by those keys. That is the software foundation for a user-held form of ownership.

It does **not** reveal Nakamoto’s private motives, and it does not prove that this was written before networking, mining, or validation. The early client contains all of those systems. This chapter is a guided reading of one part of it.

## Source trail

- Bitcoin v0.1, `bitcoin0.1/src/main.cpp`: global wallet/key state; `AddKey`; `GenerateNewKey`; `AddToWalletIfMine`.
- Bitcoin v0.1, `bitcoin0.1/src/key.h`: `CKey`, `NID_secp256k1`, `MakeNewKey`, public/private-key serialization, signing, and verification.
- Bitcoin v0.1, `bitcoin0.1/src/db.cpp`: the Berkeley DB storage layer used by the wallet.
- [Satoshi Nakamoto Institute: Bitcoin source-code archive](https://satoshi.nakamotoinstitute.org/code/)

[← Prologue: Autumn 2008]({{ '/chapters/00-autumn-2008/' | relative_url }})
