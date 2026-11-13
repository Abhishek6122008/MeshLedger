# 4. Money safety

> **In one minute**
> - Strangers carry your payment, so we need four defences: they can't **read** it, they can't **change** it, extra **copies** don't pay twice, and old copies can't be **replayed**. v1 has all four.
> - On Day 0 we attacked v1 and found holes. The worst: a bridge can re-spell a packet and get *one* payment settled *twice*. Each hole has a fix day.
> - One problem no code fully solves: someone who is offline can promise the same money to two people. v2 *reduces* that risk and measures it. We never claim it is solved.

## Threats and defences

| Threat | Example | Defence | Status |
|---|---|---|---|
| A stranger reads the payment | Learns that Alice paid Bob ₹500 | Hybrid encryption | Today (v1) |
| A stranger changes the payment | Turns ₹500 into ₹5,000 | AES-GCM tamper seal | Today (v1) |
| Many copies arrive | 3 bridges upload the same packet at once | Claim the fingerprint, plus a unique index in the DB | Today (v1), with a hole (below) |
| An old packet is replayed | A captured packet is resent next week | Nonce + signedAt freshness window | Today (v1) |
| The same bytes, re-spelled | `…A==` vs `…B==` | Fingerprint the decoded bytes, and refuse odd spellings | Planned (Day 15) |
| A crash after the claim | A payment is blocked for 24 h | Claims that expire (a lease) and remember their answer | Planned (Days 15, 17) |
| Two payments race on one balance | A lock conflict is reported as `INVALID` | Retry in a fresh transaction | Planned (Day 17) |
| Offline double-spend | Alice pays Bob and Carol with the same ₹500 | Holds, device signatures, spend tags, risk models | Planned (Days 19–27) |
| The books drift | A bug moves money from nowhere | Double-entry journal + balance check | Planned (Day 16) |

## 1. Strangers can't read it

**Analogy.** A letter goes in a box with a padlock. Only the bank has the key to the padlock.

In practice it's two locks, because the strong lock (RSA) can only lock something small:

1. The phone makes a **fresh AES-256 key** just for this payment.
2. It seals the payment with that key, using **AES-GCM**, which is fast.
3. It locks the AES key with the bank's **RSA public key** (RSA-OAEP).
4. It packs `[locked AES key][IV][sealed payment]` together and sends that.

Only the server's RSA private key can unlock the AES key, so only the server can read the payment. This hybrid pattern is the same one TLS and PGP use.

Code: `crypto/HybridCryptoService.java` (`encrypt` and `decrypt`).

## 2. Strangers can't change it

AES-GCM adds a 16-byte **tag** to the sealed data. Think of a wax seal that breaks if anyone touches the letter. If even one bit changes, `decrypt` throws an error, and the server answers `INVALID / decryption_failed`.

The test `tamperedCiphertextIsRejected` flips one character and checks exactly that.

## 3. Many copies, money moves once

**Analogy.** A ticket collector at a gate. The first time a ticket is shown, it gets torn. A copy of the same ticket shown later is turned away.

- The server makes a **fingerprint**, a SHA-256 hash of the packet, and tries to **claim** it (`IdempotencyService.claim`).
- The claim is `putIfAbsent` on a concurrent map, which is atomic. Exactly one caller wins, however many arrive together.
- Only the winner goes on to decrypt and settle. The others get `DUPLICATE_DROPPED`.
- A backup: the ledger has a **unique index** on the fingerprint, so the database refuses a second row.

The test `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` releases 3 threads at the same instant and checks that 1 settles, 2 are dropped, and Alice is charged once.

**Why fingerprint the sealed packet and not something else?**
- `packetId` is outside the seal, so any phone can rewrite it.
- The nonce is inside the seal. Reading it would mean paying for RSA on every duplicate.

## 4. Old copies can't be replayed

Inside the seal, the sender puts:
- a **nonce**: a random UUID, so two real ₹100 payments to Bob are two different packets;
- **signedAt**: the time it was made.

The server refuses packets older than `upi.mesh.packet-max-age-seconds` (24 h by default) or more than 5 minutes in the future. Nobody can change signedAt without breaking the seal. A replay of the exact same packet is also caught by the claim, as in section 3.

## Holes we found in v1

On Day 0 we sent hand-made packets to the running v1 server over HTTP. The script is `devlog/probes/day-00/` in the workspace. Before tagging `v1-baseline`, the same script was re-run against MeshLedger, with the same results.

### Hole 1: one payment, settled twice

**What happened.** Alice sent Bob ₹100. A bridge uploaded it and it settled. Then the bridge uploaded the *same bytes* written slightly differently, and it **settled again**. Alice went from 5000 to 4800 for one payment.

**Why.** Base64 turns bytes into letters, 6 bits per letter. When the data doesn't fill the last letter, the leftover bits should be zero, but Java's decoder ignores them. So `…A==` and `…B==` can stand for exactly the same bytes, the way "007" and "7" are the same number.
- The server fingerprinted the *spelling*, not the *bytes*. Two spellings gave two fingerprints and two claims.
- The unique index used the same fingerprint, so it didn't catch it either.

**Fix (Day 15).**
1. Fingerprint the **decoded bytes**.
2. Refuse non-canonical Base64: decode it, re-encode it, and compare.
3. Make the ledger's final guarantee a unique `(sender, nonce)`. That pair is what actually *is* the payment.

A regression test replays the re-spelled packet.

**What we rejected.** Checking the spelling only at the HTTP edge. Day 17 adds Kafka as a second entry point, and that would reopen the hole.

### Hole 2: "SETTLED" for a payment that failed

Dave has ₹500 and sends ₹10,000. The ledger correctly writes a `REJECTED` row, but the API answers `SETTLED`.

Fix (Day 15): a real `REJECTED` outcome, with a test. All later numbers (Days 18 and 27) count outcomes **from the ledger**, not from HTTP replies.

### Hole 3: a claim that never lets go

If anything fails *after* the claim (for example, an unknown receiver), the claim stays for 24 hours. A corrected retry of the same packet is dropped as a duplicate.

Fix (Days 15 and 17), borrowed from Fineract's idempotency design ([lesson 1](06-fineract-lessons.md)):
- A claim has a state: `IN_PROGRESS` or `DONE`.
- A `DONE` claim stores the answer, so a duplicate gets the *original* outcome and transaction id back.
- An `IN_PROGRESS` claim has a **lease**. If the server crashes, the lease runs out and the packet can be tried again.

### Hole 4: two payments race for one balance (not yet reproduced)

If Alice's two different payments settle at the same moment, both read her balance and then both write it. `@Version` makes the second write fail, which is correct: there is no lost update. But the failure becomes `INVALID / internal_error`, and the claim gets stuck (Hole 3).

So far this comes only from reading the code. Day 17 reproduces it first, then fixes it with a short **retry in a fresh transaction**, which is how Fineract retries commands on lock conflicts ([lesson 2](06-fineract-lessons.md)).

## 5. The books must balance (Planned, Day 16)

**Analogy.** An old-style account book with two columns. Every rupee that leaves one account must arrive in another, so the two columns always add up to the same total.

From Day 16, every payment writes two journal lines (debit the sender, credit the receiver) with the same amount. A check runs after the big tests:
- total debits = total credits;
- the sum of all balances hasn't changed;
- each balance = opening balance + credits − debits.

If a bug ever creates or destroys money, this check fails, even when every individual test passes. The ledger is also **append-only**: a mistake is fixed by a new reversing entry, never by editing or deleting a row. Both ideas come from Fineract ([lesson 3](06-fineract-lessons.md)).

## 6. The hard one: offline double-spend

**The problem.** Alice has ₹500. Offline, in basement A, she pays Bob ₹500. She walks to basement B, where nobody has seen Bob's packet, and pays Carol ₹500. Neither receiver can call the bank. Whichever packet reaches the server first settles, and the other one bounces.

No amount of server code fixes this on its own, because the cheating happens where there is no server. v2 makes it **harder, smaller, and measurable**:

| Layer | Day | What it does | What it can't do |
|---|---|---|---|
| **Hold (offline allowance)** | 19 | On sync, the bank reserves part of Alice's balance and signs a statement: "device X may spend up to ₹A offline until time T". Online spending can't touch the held money. Analogy: a reserved seat that nobody else can take. | Alice can still promise the *same* allowance to two people |
| **Device signatures** | 19 | Each payment is signed by Alice's device key, so the receiver can check offline that it really came from the device in the snapshot | It doesn't stop the device's owner from cheating |
| **Spend tags** | 20 | Each payment carries a clear-text running total for that snapshot. If two payments meet in the mesh with the same snapshot but totals that don't add up, every phone can see the conflict | Two basements that never touch won't see each other's tags |
| **Rules, then a receiver model** | 20, 23 | The receiver decides *accept*, *confirm first* or *refuse*, based on the amount vs the allowance, the snapshot's age, and conflicts it has seen | It is a judgement, so it can be wrong both ways |
| **Server risk layer** | 24–26 | Catches patterns after the fact: velocity, rings, odd bridges. It feeds a blocklist that phones download | It acts after the money has moved |

**The draft rule for who carries the loss (decided on Day 19):** payments inside the signed allowance settle from the hold. Anything beyond it can bounce, just like v1.

**The honest scorecard.** Day 27 measures how much double-spend loss the risk layer removes, and how many honest payments it wrongly blocks along the way. Both numbers get reported, across thresholds, for rules and for the model.

## Not defended yet (and why)

- **Bridge authentication.** `/api/bridge/ingest` is open. The plan only detects bad bridges (Day 24: spikes of bad packets, withheld packets) rather than adding certificates.
- **The PIN.** v1 stores an unsalted SHA-256 of a 4-digit PIN and never checks it. A real deployment checks the PIN at the bank, so we don't build fake PIN security.
- **Key storage.** The server's private key lives in memory. A real deployment keeps it in an HSM or KMS. Day 16 only makes the key shared across replicas, from a keystore file.
- **Metadata privacy.** Strangers can see that a packet exists, and in v2 they can also see the spend tag's pseudonym.
