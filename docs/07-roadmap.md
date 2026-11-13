# 7. Roadmap: the plan, day by day

> **In one minute**
> - MeshLedger is the second half of a 30-working-day plan. The first half (Days 1–14) mostly builds **Gratis**, my Redis-compatible engine. MeshLedger's own days are **15–30**, plus Day 1's setup.
> - **Week 3** (Days 15–21): the plumbing (a shared claim store, 3 replicas, Kafka, a load test), then the offline protocol (signed allowances, spend tags).
> - **Week 4** (Days 22–28): the AI layer.
> - **Days 29–30**: ship.
> - Each day below says what gets built, how it's proven, which Fineract lesson it uses, and how it differs from the original plan text and why.

**Status key:** Done · In progress · Not started. Numbers are added only after they are measured.

## Day 1 (Done): a fresh start

- **Built.**
  - The new public repo, **MeshLedger**.
  - RupeeRelay v1 rebuilt in 16 small commits (wrapper → app skeleton → CI → models → crypto → services → API → dashboard → Dockerfile → README). Every commit builds and passes its tests.
  - Tag `v1-baseline`, then these docs on branch `v2`.
- **Proven.**
  - 3/3 tests pass.
  - A file-by-file comparison against RupeeRelay v1 (with names mapped) shows only the rename.
  - The Day 0 attack script, re-run against MeshLedger, gives the same results, bugs included.
- **Changed from the plan.** The plan assumed the existing RupeeRelay-v2 repo. Instead the project got a new name and a new repo, rebuilt in small steps, so its history tells the story from the start.

## What MeshLedger needs from Gratis

| Gratis day | Feature | MeshLedger uses it on |
|---|---|---|
| 8 | `SET key value NX EX seconds` | 15–18 (claim store) |
| 13 | `WINCR` / `WCOUNT` windowed counters, event-time aware | 24 (sender velocity) |
| 14 | Bloom filter `BF.RESERVE` / `BF.ADD` / `BF.EXISTS` | 26 (blocklist) |

Gratis v1 crashes on `SET … NX EX …` (seen on Day 0). So in Day 18's chart, the "engine v1" column is marked **N/A, crashes on SET NX EX**.

---

## Week 3: plumbing, then the protocol

### Day 15: a claim store you can swap

- **Goal.** The "seen it" list can live in memory or in Redis, and the code doesn't care which.
- **Built.**
  - An `IdempotencyStore` interface, extracted from today's `IdempotencyService` rather than rewritten.
  - An in-memory implementation (the default) and a Redis one (`SET NX EX 86400`), chosen by a property.
  - The exact command the client sends, written down.
  - `docker-compose` with Redis, Postgres and Kafka.
- **Also fixed** (Day 0 holes):
  - Fingerprint the decoded bytes and refuse non-canonical Base64.
  - Unique `(sender, nonce)` in the ledger.
  - A real `REJECTED` outcome, and `UNKNOWN_ACCOUNT` instead of `internal_error`.
- **Proven.**
  - `IdempotencyConcurrencyTest` runs on both stores. The Redis run uses Testcontainers and is skipped without Docker.
  - Then the same test runs against Gratis.
  - Plus a regression test for the re-spelled packet.
- **Fineract lessons.**
  - The claim stores its outcome, and duplicates get the original answer back ([1](06-fineract-lessons.md#1-idempotency-that-remembers-the-answer)).
  - Money rules ([6](06-fineract-lessons.md#6-money-has-rules)).
  - Clear outcomes ([7](06-fineract-lessons.md#7-clear-outcomes-not-internal-error)).
  - The `Clock` bean ([10](06-fineract-lessons.md#10-a-clock-the-system-controls-and-a-close-of-business-job)).
- **Changed from the plan.**
  - The Base64 fix comes *before* the Redis key format is written down, so the documented key is the safe one.
  - Spring Boot 3.3.5 pins an old Testcontainers that probably fails on current Docker. The plan is to override its version in the pom, with no Java change. To be checked on the day.
- **Open question for you:** the property name. The plan says `upimesh.idempotency.store`, while the existing keys use `upi.mesh.*`.

### Day 16: exactly once, across 3 servers

- **Goal.** Three replicas share one ledger, and a duplicate storm still moves money once.
- **Built.**
  - A Postgres profile, because H2 lives inside one JVM and replicas can't share it.
  - One shared RSA key loaded from a keystore file or env var. Today each server makes its own key and can't open packets sealed for another.
  - A single seed step.
  - The simulator as an outside client of `/api/bridge/ingest`.
  - A small load balancer in compose.
- **Proven.**
  - The storm runs on `redis:8` and on Gratis.
  - Then the **books-balance check**: debits = credits, the total of all balances is unchanged, and every balance = opening + credits − debits.
- **Fineract lessons.**
  - The double-entry journal ([3](06-fineract-lessons.md#3-double-entry-every-payment-is-two-equal-lines)).
  - Never delete, reverse ([4](06-fineract-lessons.md#4-never-delete-reverse)).
  - Liquibase plus a registration test ([5](06-fineract-lessons.md#5-liquibase-and-a-check-that-every-migration-is-registered)).
- **Changed from the plan.** Day 0 left "Flyway or validate" open. Now it's Liquibase with `ddl-auto=validate`.

### Day 17: Kafka, and no payment blocked forever

- **Goal.** Put a queue between "received" and "settled". A claim that crashes must not block its payment.
- **Built.**
  - Kafka between `/api/bridge/ingest` and settlement, keyed by sender.
  - An **idempotent consumer**. **[ME]: you write it.** I explain the approach, then review and test it.
  - The claim gets a lease: an `IN_PROGRESS` claim expires.
  - The database's unique key stays the final guarantee, and the claim stays the early filter.
- **Proven.**
  - A test that crashes after the claim and checks the packet still settles later.
  - A test that reproduces two same-sender payments colliding on `@Version`, then the retry fix.
- **Fineract lessons.**
  - The lease and the "try later" state ([1](06-fineract-lessons.md#1-idempotency-that-remembers-the-answer)).
  - Retry on lock conflicts ([2](06-fineract-lessons.md#2-retry-when-two-writes-collide)).
- **Changed from the plan.** Kafka makes settlement asynchronous, so the bridge no longer gets a transaction id straight away. It sits behind a profile, and the default stays synchronous, so today's tests keep their meaning.

### Day 18: a reconnect storm

- **Goal.** See how ingest behaves when hundreds of bridges come online at once.
- **Built.**
  - Configurable phone and bridge counts; the defaults stay 5 and 1.
  - A storm generator.
- **Proven.**
  - Ingest p99 latency and throughput for: in-memory vs `redis:8` vs Gratis v1 (N/A, it crashes) vs Gratis v2.
  - 5 runs each, reporting median and spread, with the exact commands.
  - Outcomes are counted **from the ledger**, not from HTTP replies.

### Day 19: signed allowances and device keys

- **Goal.** A receiver can check offline that a payment is backed by real, reserved money.
- **Built.**
  - A device registry with device key pairs.
  - `/api/device/sync`, which places a **hold** for the offline allowance and returns a server-signed snapshot `{device, allowance, seq, validUntil}`.
  - Payments signed by the device.
  - Offline verification at the receiver.
  - A nightly close-of-business job that releases expired holds.
- **Proven.** Tests for:
  - signature checks;
  - "withdrawable = balance − holds";
  - payments signed outside the hold's validity being refused;
  - hold expiry with a moved `Clock`.
- **Fineract lessons.**
  - Holds ([8](06-fineract-lessons.md#8-a-hold-reserves-money-without-spending-it)).
  - Value date vs booking date and the backdated window ([9](06-fineract-lessons.md#9-value-date-vs-booking-date-and-how-far-back-is-allowed)).
  - The business clock and COB ([10](06-fineract-lessons.md#10-a-clock-the-system-controls-and-a-close-of-business-job)).
- **Changed from the plan.**
  - The plan said "a server-signed balance snapshot". Signing the *balance* promises nothing, because the balance can be spent online a minute later. Signing a *hold* promises the money is still there.
  - v1 has no devices or sync at all, so this day also defines them.
- **Open question for you:** who carries the loss when a payment goes beyond the allowance? The draft is "the receiver, as in v1".

### Day 20: spend tags

- **Goal.** Phones can spot a double-spender when the conflicting payments meet in the mesh.
- **Built.**
  - A clear-text spend tag `{sender pseudonym, snapshot seq, cumulative spent}` that is gossiped with each payment.
  - Simple rule-based offline limits. These are the baseline every model has to beat.
- **Proven.** A simulated double-spend inside one mesh is flagged by the receivers. A cross-basement one is not, and the doc says so.

### Day 21: buffer and review

PR `v2` → `main` with the real numbers from Days 15–20. After your approval, merge without squashing and tag `v2-protocol`.

---

## Week 4: the AI layer

See [The AI layer](05-ai-layer.md) for how each part works.

| Day | Goal | Proven by |
|---|---|---|
| **22** | A fraud simulator: thousands of moving phones; honest users, overspenders, double-spenders and mule rings; fixed seeds; a labelled log; split by ring | Same seed gives the same log; every fraud event has a label |
| **23** | Receiver model: trained in Python, runs in each virtual phone, outputs accept / confirm / refuse | PR-AUC on held-out rings vs the Day 20 rules |
| **24** | Server risk consumer: WINCR velocity keyed by signedAt, fan-in/fan-out, bridge spikes and withholding, repeat overspenders; LightGBM via ONNX Runtime; a case queue | PR-AUC vs rules; the time it adds per payment |
| **25** | AI investigator over MCP with read-only tools; case reports that cite IDs | Right ring members named; **zero invented transaction IDs**. LLM choice: ask on the day (a local Ollama model is fine) |
| **26** | Bloom-filter blocklist built by Gratis, downloaded on sync; a hit means "confirm first" | False-positive rate vs filter size |
| **27** | **Headline:** double-spend losses with vs without the risk layer, against % of honest payments wrongly declined, across thresholds; rules vs model | The curve, with every run's command |
| **28** | Buffer and review. PR + tag `v2-risk` | |

The Day 24 habit from Fineract: any job that walks the ledger uses keyset paging (`id > lastId`) and fails loudly.

---

## Ship

- **Day 29.**
  - READMEs with the charts at the top.
  - An architecture diagram and reproduce-it steps.
  - An updated dashboard.
  - The "Honest limitations" section is kept, and v2 is framed as **measured risk reduction**, never "solved double-spending".
- **Day 30.**
  - Re-run every benchmark from a clean clone.
  - Two 90-second demo scripts.
  - Tag `v2.0`.
  - Resume bullets using only real numbers.

## If we fall behind

These come from the original plan, in this order.

**Cut first:**
1. the Day 26 blocklist;
2. the MCP wrapper (use plain tool calls instead);
3. Gratis sharding;
4. the Day 20 spend-tag gossip.

**Never cut:**
- `v1-baseline` and honest benchmarks;
- exactly-once on Gratis across replicas;
- the Day 27 headline metric.

The Fineract additions are small and live inside their days. If one of them squeezes its day, my proposal is to drop the nightly close-of-business *job* first and run the balance check as a test only. The holds themselves stay, because the Day 19 protocol depends on them.

## Open questions

| Question | Needed by |
|---|---|
| Property name for the claim store | Day 15 |
| Who carries the loss beyond the allowance | Day 19 |
| Model format for the on-phone receiver model | Day 23 |
| Which LLM for the investigator | Day 25 |
