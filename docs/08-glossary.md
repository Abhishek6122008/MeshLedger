# 8. Glossary

Short definitions, grouped by topic. Where a term is explained in depth, the link goes there.

## Payments

| Term | Meaning |
|---|---|
| **UPI** | India's real-time bank payment system. You pay a person's address instead of their account number. |
| **VPA** | Virtual Payment Address, the UPI address itself, like `alice@demo`. |
| **UPI Lite** | A real offline-friendly UPI feature: a small pre-funded wallet on the phone. MeshLedger's hold idea plays the same role. |
| **Settle** | Actually move the money: debit the sender, credit the receiver, write the ledger row. |
| **Ledger** | The permanent record of every money movement. |
| **Overspend** | Trying to pay more than you have. In MeshLedger it ends as `REJECTED`. |
| **Double-spend** | Promising the same money to two people. Offline, nobody can stop it in the moment. See [Money safety](04-money-safety.md#6-the-hard-one-offline-double-spend). |

## The mesh

| Term | Meaning |
|---|---|
| **Mesh** | Phones passing data to nearby phones, with no internet or cell tower in between. |
| **Gossip** | Each phone shares what it holds with the phones near it, and they do the same. |
| **TTL** (time to live) | A hop counter on each packet. Every forward uses one up, and at 0 the packet stops travelling. |
| **Bridge** | Any phone in the mesh that gets internet and uploads the packets it holds. |
| **Packet / envelope** | A `MeshPacket`: a few outer fields phones can read, plus a sealed payment they can't. |
| **Reconnect storm** | Many bridges coming online at once and uploading together (Day 18). |

## Security

| Term | Meaning |
|---|---|
| **Hash / fingerprint** | A short, fixed-size summary of data (SHA-256 here). The same input always gives the same hash, and a different input almost surely gives a different one. |
| **Nonce** | A random one-time value inside each payment, so two real payments never look identical. |
| **Replay** | Sending an old, valid packet again to get it processed twice. |
| **Freshness window** | How old a packet may be and still be accepted (24 h in v1). |
| **RSA-OAEP** | Public-key encryption: anyone can lock with the public key, and only the private key unlocks. Used here to lock a small AES key. |
| **AES-GCM** | Fast encryption that also adds a **tag**, a tamper seal that fails if any bit changes. |
| **Hybrid encryption** | Lock the data with a fresh AES key, then lock that key with RSA. This is how TLS and PGP work too. |
| **Base64** | A way to write bytes as letters. The same bytes can sometimes be written more than one way, which was Day 0's double-settle bug. |
| **Canonical** | The one official way of writing something. We refuse every other spelling. |
| **Device key / signature** | (Day 19) A key pair on each phone. Signing proves a payment came from that device. |
| **Snapshot** | (Day 19) A server-signed statement: "device X may spend up to ₹A offline until time T". |
| **Spend tag** | (Day 20) A clear-text label on a payment: sender pseudonym, snapshot number, running total spent. Lets phones spot conflicts. |
| **Pseudonym** | A stable fake name, so phones can link payments from the same sender without knowing who it is. |

## Correctness under load

| Term | Meaning |
|---|---|
| **Idempotency** | Doing something twice has the same effect as doing it once. |
| **Claim** | The first step of ingest: "I'm handling this fingerprint". Only one caller can win it. |
| **`putIfAbsent` / `SET NX`** | Atomic "store this only if nothing is there yet": the in-memory and Redis versions of a claim. |
| **Lease** | (Day 17) An expiry on a claim, so a crashed server's claim frees itself. |
| **Transaction (DB)** | A group of database writes that all happen or none do. |
| **Optimistic locking** | Each row has a version number. A write fails if the version changed since it was read, so nobody silently overwrites anybody. |
| **Exactly once** | Every payment settles one time, never zero and never two, even with duplicates, retries and crashes. |
| **Replica** | One of several identical copies of the server running at the same time (Day 16). |
| **Kafka** | A durable message queue. Here it sits between "received" and "settled" (Day 17). |
| **p99** | The 99th-percentile latency: 99% of requests are faster than this. |

## Banking ideas from Fineract

| Term | Meaning |
|---|---|
| **Double-entry** | Every movement is written twice, a debit and a credit of the same amount, so the books always balance. |
| **Journal entry** | One of those debit or credit lines. |
| **Reversal** | Undoing a ledger entry by adding an opposite entry, never by deleting. |
| **Hold / lien** | Money reserved on an account. It is still in the balance, but it can't be spent. |
| **Withdrawable balance** | Balance minus holds (and any minimum balance): what you can actually spend. |
| **Value date** | When a payment really happened (`signedAt`). |
| **Booking date** | When the bank recorded it (`settledAt`). Offline payments are always booked after their value date. |
| **Business date** | The date the system *treats* as today, which tests can move. Here it's a `Clock` bean. |
| **Close of business (COB)** | The end-of-day job: expire holds and check the books. |
| **Liquibase** | A tool that applies numbered database changes in order and remembers which ones already ran. |
| **Outbox** | (Not built) Saving an event in the same DB transaction as the change, and sending it later, so no event is lost in a crash. |
| **Maker-checker** | (Not built) One person proposes an action, and a different person must approve it. |

## AI and data

| Term | Meaning |
|---|---|
| **Model** | A function learned from labelled examples that turns facts into a score. |
| **Feature** | One fact the model looks at, such as "payments by this sender in the last 10 minutes". |
| **Label** | The known answer for a training example: fraud or honest. |
| **Seed** | The starting number for a random generator. The same seed gives the same "random" data. |
| **Held-out / test set** | Data kept away from training, used only to judge the model. |
| **Precision / recall** | Of what we flagged, how much was fraud / of all fraud, how much we flagged. |
| **PR-AUC** | Precision and recall across every threshold, as one number. It works on rare events, where accuracy doesn't. |
| **Threshold** | The score cut-off that turns a score into accept, confirm or refuse. |
| **False decline** | An honest payment refused or delayed. |
| **LightGBM** | A fast gradient-boosted-tree model: many small decision trees, each fixing the last one's mistakes. |
| **ONNX / ONNX Runtime** | A standard file format for trained models, and the engine that runs them, here from Java. |
| **LLM** | A large language model, the kind of AI that reads and writes text. |
| **MCP** | Model Context Protocol, a standard way to give an LLM tools to call. |
| **Bloom filter** | A tiny bit array that answers "definitely not seen" or "maybe seen". It never misses, but sometimes says "maybe" wrongly. |
| **WINCR / WCOUNT** | Gratis's windowed counter commands: count events per key over a sliding time window, by event time. |

## Project words

| Term | Meaning |
|---|---|
| **Gratis** | My Redis-compatible engine, built in the same 30-day plan. MeshLedger runs on it. |
| **v1-baseline** | The git tag on the rebuilt v1, the "before" in every before/after comparison. |
| **[ME]** | A plan item I write myself. Claude explains the approach and reviews. |
