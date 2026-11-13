# 6. Lessons from Apache Fineract

> **In one minute**
> - [Apache Fineract](https://github.com/apache/fineract) is an open-source core-banking platform that real banks and lenders run. I contribute to it: global configuration, standing instructions, savings interest posting, Liquibase checks, and the new command framework.
> - Fineract has already solved, at scale, several problems MeshLedger runs into:
>   - "a client retried, don't do it twice";
>   - "the books must balance";
>   - "reserve money without spending it";
>   - "a transaction dated in the past".
> - This doc lists each idea: how Fineract does it (with real file paths), what it means in plain words, and where it lands in MeshLedger.
> - Three groups get **built**: the ledger, idempotency that remembers, and holds as the offline allowance. One group is **explained but not built**. A few things are **deliberately not copied**.

Fineract paths point at `develop` as of commit `18ce75b63b` (2026-10-29). Code excerpts are copied from that commit.

## How each lesson is laid out

Each lesson has four parts:
- **In Fineract**: the real classes and tables, plus my own ticket where I worked on that area.
- **In plain words.**
- **In MeshLedger**: where it goes, and on which day.
- **What I changed or didn't copy.**

---

## Built: group A, idempotency that remembers

### 1. Idempotency that remembers the answer

**In Fineract.**
- A client can send an `Idempotency-Key` header on any write.
  - [`IdempotencyStoreFilter`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/infrastructure/core/filters/IdempotencyStoreFilter.java) reads the key.
  - Every write command is recorded in `m_portfolio_command_source` ([`CommandSource`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/commands/domain/CommandSource.java)), with the key, the result body and the result status code.
  - A unique constraint on `(action_name, entity_name, idempotency_key)` makes the key one-time per action ([changelog part 0061](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/resources/db/changelog/tenant/parts/0061_add_idempotency_key_to_command_source.xml)).
- On a retry with the same key, [`SynchronousCommandProcessingService`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/commands/service/SynchronousCommandProcessingService.java) checks the stored status:

```java
switch (status) {
    case UNDER_PROCESSING -> {
        Class<?> lastExecutionExceptionClass = retryConfigurationAssembler.getLastException();
        if (lastExecutionExceptionClass == null
                || IdempotentCommandProcessUnderProcessingException.class.isAssignableFrom(lastExecutionExceptionClass)) {
            throw new IdempotentCommandProcessUnderProcessingException(wrapper, idempotencyKey);
        }
    }
    case PROCESSED -> throw new IdempotentCommandProcessSucceedException(wrapper, idempotencyKey, command);
    case ERROR -> {
        if (!retry) {
            throw new IdempotentCommandProcessFailedException(wrapper, idempotencyKey, command);
        }
    }
    default -> {
    }
}
```

- `PROCESSED` replays the stored result, with the header `x-served-from-cache`.
- `UNDER_PROCESSING` becomes HTTP **425 Too Early** (`IdempotentCommandExceptionMapper`).

**In plain words.** A coat-check desk that writes down what it handed back. If you show the same ticket again, it doesn't hand over a second coat; it tells you which coat you already got. If your coat is still being fetched, it says "come back in a moment".

**In MeshLedger (Day 15, finished on Day 17).** Today a duplicate gets only `DUPLICATE_DROPPED`, so a bridge that timed out never learns whether the payment worked. The claim becomes a record:

| State | A duplicate gets |
|---|---|
| `IN_PROGRESS` | `IN_PROGRESS`: try again later (Fineract's 425) |
| `DONE` | The **original** outcome and `transactionId` (Fineract's replayed result) |

**What I changed.** Fineract replays an `ERROR` result to the same key. MeshLedger doesn't:
- An `IN_PROGRESS` claim has a **lease** (an expiry time).
- If the server dies mid-payment, the lease runs out and the next copy of the packet can try again.

The reason: a Fineract client can always send a new request with a new key. A mesh packet is the *only* copy of that payment, and it can't be re-signed. This is the plan's "claim-then-crash" fix.

### 2. Retry when two writes collide

**In Fineract.** Each command runs inside a Resilience4j retry ([`RetryConfigurationAssembler`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/commands/configuration/RetryConfigurationAssembler.java)). Every attempt gets its own transaction. From [`application.properties`](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/resources/application.properties):

```properties
fineract.retry.instances.executeCommand.max-attempts=${FINERACT_COMMAND_PROCESSING_RETRY_MAX_ATTEMPTS:3}
fineract.retry.instances.executeCommand.wait-duration=${FINERACT_COMMAND_PROCESSING_RETRY_WAIT_DURATION:1s}
fineract.retry.instances.executeCommand.enable-exponential-backoff=${FINERACT_COMMAND_PROCESSING_RETRY_ENABLE_EXPONENTIAL_BACKOFF:true}
```

Its retry list includes the optimistic-lock exceptions (`jakarta.persistence.OptimisticLockException`, `JpaOptimisticLockingFailureException`, and others).

**In plain words.** Two cashiers update the same account card at once. The second one notices the card changed under them, so they re-read it and try again, instead of giving up or writing over the first cashier's work.

**In MeshLedger (Day 17).** `Account` already has `@Version`, so a collision is *detected*. But today it ends as `INVALID / internal_error` with a stuck claim. Day 17 first reproduces that with a test. Then `settle` gets a short, capped retry, each attempt in a fresh transaction. If it still fails, the claim is released so the packet can come back.

---

## Built: group B, a ledger like a bank's

### 3. Double-entry: every payment is two equal lines

**In Fineract.**
- GL accounts have five types ([`GLAccountType`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/accounting/glaccount/domain/GLAccountType.java)): ASSET, LIABILITY, EQUITY, INCOME and EXPENSE.
- Journal lines are DEBIT or CREDIT ([`JournalEntryType`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/accounting/journalentry/domain/JournalEntryType.java)), stored in `acc_gl_journal_entry` ([`JournalEntry`](https://github.com/apache/fineract/blob/develop/fineract-accounting/src/main/java/org/apache/fineract/accounting/journalentry/domain/JournalEntry.java)).
- [`AccountingProcessorHelperImpl`](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/java/org/apache/fineract/accounting/journalentry/service/AccountingProcessorHelperImpl.java) always writes a matching debit and credit for the same amount and the same `transaction_id`.
- Manual journal entries are rejected if debits and credits don't add up to the same total.

My work on savings interest posting (FINERACT-2705: cash-based GL accounts in the posting job) is where I learned how a business event turns into balanced GL postings.

**In plain words.** An old account book with two columns. Every rupee that leaves one account arrives in another, so both columns always add up to the same total. If they don't, something is wrong, even if each line looks fine on its own.

**In MeshLedger (Day 16).**
- A new `journal_entries` table. Every settlement writes a DEBIT line for the sender and a CREDIT line for the receiver, with one `transaction_id`.
- A **books-balance check** runs after the 3-replica duplicate storm:
  - total debits = total credits;
  - the sum of all balances hasn't changed;
  - every balance = opening + credits − debits.
- That check is the proof behind "exactly once across 3 replicas". Counting HTTP replies isn't enough; the books themselves have to balance.

**What I didn't copy.** Fineract's full chart of accounts (GL account tree, product-to-GL mappings, accounting rules). MeshLedger has user accounts only. One flat pair of lines per payment is enough to prove money is neither created nor lost.

### 4. Never delete; reverse

**In Fineract.**
- A journal entry is never deleted. `revertJournalEntry` in [`JournalEntryWritePlatformServiceImpl`](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/java/org/apache/fineract/accounting/journalentry/service/JournalEntryWritePlatformServiceImpl.java) writes an opposite entry, then marks the original `reversed` and links it to the reversal.
- Savings transactions work the same way ([`SavingsAccountTransaction`](https://github.com/apache/fineract/blob/develop/fineract-savings/src/main/java/org/apache/fineract/portfolio/savings/domain/SavingsAccountTransaction.java)):

```java
public static SavingsAccountTransaction reversal(SavingsAccountTransaction accountTransaction) {
    SavingsAccountTransaction sat = copyTransaction(accountTransaction);
    sat.reversed = false;
    sat.setReversalTransaction(true);
    sat.originalTxnId = accountTransaction.getId();
    return sat;
}
```

**In plain words.** In a bank's book, you never use an eraser. If a line is wrong, you add a new line that cancels it and point it at the old one. Anyone can see what happened and when.

**In MeshLedger (Day 16, as a rule).**
- `transactions` and `journal_entries` are append-only.
- A correction, such as undoing a payment that turns out to be fraud, is a new reversing pair linked by `reversal_of`.
- On Postgres, the app's database user gets INSERT and SELECT on these tables, but no UPDATE or DELETE.

### 5. Liquibase, and a check that every migration is registered

**In Fineract.**
- The schema is built by Liquibase, not Flyway. [`db.changelog-master.xml`](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/resources/db/changelog/db.changelog-master.xml) includes numbered `parts/NNNN_*.xml` files in order.
- A part that exists but isn't included silently never runs. My FINERACT-2903 added [`scripts/check-liquibase-changelog-registration.sh`](https://github.com/apache/fineract/blob/develop/scripts/check-liquibase-changelog-registration.sh) and a CI job, which catch unregistered parts, duplicate includes and missing files.

**In plain words.** Database changes are numbered recipe cards, with an index card listing which ones to follow. A card that isn't on the index might as well not exist, so a robot checks the index.

**In MeshLedger (Day 16).**
- The Postgres profile uses Liquibase with `ddl-auto=validate`. Hibernate checks the schema but never changes it.
- A small JUnit test fails if any file under `db/changelog/parts/` isn't included by the master file.
- This replaces Day 0's open choice of "Flyway or validate".

### 6. Money has rules

**In Fineract.**
- [`Money`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/organisation/monetary/domain/Money.java) is immutable. It carries a currency ([`MonetaryCurrency`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/organisation/monetary/domain/MonetaryCurrency.java)), and rounds to the currency's decimal places with a configured rounding mode.
- Mixing currencies throws an error.
- It never uses `double`.

**In plain words.** Rupees have exactly two decimal places. "₹0.001" isn't a real amount, and the code should say so instead of quietly rounding it.

**In MeshLedger (Day 15).**
- Before settling, an amount must have at most 2 decimal places, be greater than 0, and be under a configured per-payment limit.
- Whether v1 currently *accepts* something like `0.001` is **not yet known**. Day 15 checks it with a test before fixing anything.

### 7. Clear outcomes, not "internal error"

**In Fineract.** My early fixes were all of one kind:
- FINERACT-2671 and FINERACT-2673 made "not found" return 404 instead of 500.
- FINERACT-2694 made missing parameters return a 400 validation error instead of a 500 null-pointer error.

The rule I took away: **a business problem deserves its own answer**, not a generic crash.

**In MeshLedger (Day 15).**
- An overspend returns `REJECTED`, not `SETTLED` (Day 0 found the API saying `SETTLED`).
- An unknown account returns `INVALID / UNKNOWN_ACCOUNT`, not `internal_error: …`.
- These cases also release the claim, because they are final answers, not crashes.

---

## Built: group C, the offline allowance as a hold

### 8. A hold reserves money without spending it

**In Fineract.**
- A savings account can have money **on hold**. `AMOUNT_HOLD` and `AMOUNT_RELEASE` are their own transaction types ([`SavingsAccountTransactionType`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/savings/SavingsAccountTransactionType.java)).
- A hold doesn't change the balance. It changes what you're allowed to **withdraw** ([`SavingsAccount`](https://github.com/apache/fineract/blob/develop/fineract-savings/src/main/java/org/apache/fineract/portfolio/savings/domain/SavingsAccount.java)):

```java
public BigDecimal getWithdrawableBalance() {
    return getAccountBalance().subtract(minRequiredBalanceDerived(getCurrency()).getAmount()).subtract(this.getOnHoldFunds())
            .subtract(this.getSavingsHoldAmount());
}
```

My FINERACT-2400 (partial standing-instruction transfers) uses exactly this method. When a scheduled transfer can't be covered in full, the job moves only what is withdrawable. That's how I learned the difference between "balance" and "what you can actually spend".

**In plain words.** A reserved seat. It's still in the theatre and still counted, but nobody else can sit in it.

**In MeshLedger (Days 19–20).**
- When a phone syncs, the server places a **hold** for its offline allowance and signs a snapshot: `{device, allowance, seq, validUntil}`.
- Online spending sees *withdrawable = balance − active holds*, so it can't spend the reserved part.
- Offline payments signed against that snapshot settle **from the hold**.
- Whatever is left is released on the next sync, or when the hold expires.
- This gives receivers something real to check offline. It is the same idea as UPI Lite's pre-funded wallet, built with a bank's hold.

**What I changed.** Fineract records holds as hold and release transactions on the account, plus a running "on hold" total. MeshLedger gives **each hold its own row**, with its device, its signed snapshot, the amount used so far, and its own expiry. A hold here is a promise to one phone, not just a reserved amount.

### 9. Value date vs booking date, and how far back is allowed

**In Fineract.**
- Transactions have a business date, which can be earlier than the day they're entered.
- My FINERACT-1950 added the global config `disallow-backdated-transactions` ([changelog part 0251](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/resources/db/changelog/tenant/parts/0251_add_disallow_backdated_transactions_configuration.xml)). It is enforced in one shared place, [`BackdatedTransactionValidationService`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/infrastructure/configuration/service/BackdatedTransactionValidationService.java):

```java
public void validateTransactionDate(final LocalDate transactionDate) {
    if (transactionDate == null || !configurationDomainService.isBackdatedTransactionsDisallowed()) {
        return;
    }
    final LocalDate businessDate = DateUtils.getBusinessLocalDate();
    final Long toleranceDays = configurationDomainService.retrieveBackdatedTransactionsToleranceDays();
    final LocalDate earliestAllowedDate = businessDate.minusDays(toleranceDays != null && toleranceDays > 0 ? toleranceDays : 0);
    if (transactionDate.isBefore(earliestAllowedDate)) {
        throw new GeneralPlatformDomainRuleException("error.msg.transaction.backdated.not.allowed",
                "Backdated transactions are disallowed", transactionDate, earliestAllowedDate);
    }
}
```

**In plain words.** Every offline payment is backdated by design. It *happened* when Alice signed it, but the bank only *books* it when a bridge uploads it, possibly hours later. Banks call these the **value date** and the **booking date**.

**In MeshLedger (Day 19).**
- `signedAt` is the value date and `settledAt` is the booking date. Both are stored on the journal lines.
- Today's flat 24-hour freshness window becomes: **a payment must be signed while its hold was valid**.
- Like FINERACT-1950, the check lives in one validator, not scattered across callers.

### 10. A clock the system controls, and a close-of-business job

**In Fineract.**
- Code asks [`DateUtils.getBusinessLocalDate()`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/infrastructure/core/service/DateUtils.java) for "today", not the wall clock. The business date is stored in `m_business_date` ([`BusinessDate`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/infrastructure/businessdate/domain/BusinessDate.java)).
- A close-of-business (COB) process then moves accounts forward day by day.

**In plain words.** A clock you are allowed to wind. Tests and simulations can say "it is now 3 days later" without waiting 3 days.

**In MeshLedger (Day 15 for the clock, Day 19 for the job).**
- One `java.time.Clock` bean replaces every `Instant.now()`. Tests use a fixed or moving clock, and the Day 22 fraud simulator can run days of activity in minutes.
- A nightly `@Scheduled` close-of-business job releases expired holds and runs the books-balance check.

**What I didn't copy.** Spring Batch, partitioned workers and account locking. Those exist because Fineract processes millions of loans. A single scheduled method is enough here.

---

## Explained, not built: group D, ops guardrails

These are good ideas that didn't make the cut for the 30 days. They're written down so it's clear they were considered.

- **Transactional outbox.**
  - In Fineract: business events are saved to `m_external_event` ([`ExternalEvent`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/infrastructure/event/external/repository/domain/ExternalEvent.java)) *in the same database transaction* as the change.
  - A job ([`SendAsynchronousEventsTasklet`](https://github.com/apache/fineract/blob/develop/fineract-core/src/main/java/org/apache/fineract/infrastructure/event/external/jobs/SendAsynchronousEventsTasklet.java)) then sends them to Kafka or JMS. This avoids "saved but never announced" after a crash.
  - MeshLedger instead publishes settled events directly and accepts that gap. The risk layer can rebuild its features from the ledger.
- **Command audit trail.** Every write is recorded in `m_portfolio_command_source`: who, what, when, and the result. MeshLedger's ledger plus its logs cover the demo.
- **Maker-checker.**
  - A command can be parked until a *different* user approves it ([`MakercheckersApiResource`](https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/java/org/apache/fineract/commands/api/MakercheckersApiResource.java)).
  - MeshLedger's AI investigator is read-only anyway: it can't act, so there is nothing to approve. If case actions are added later, maker-checker is the first thing to add.

## Habits I kept (they cost nothing)

- **Keyset paging.** Any job that walks the ledger pages with `WHERE id > :lastId ORDER BY id`, never `OFFSET`. My FINERACT-2915 fixed a job whose paging skipped loans.
- **Jobs fail loudly.** A background job that hits an error marks itself failed instead of logging and carrying on. This is what my FINERACT-2916 patch does for the interest jobs.
- **Bound SQL parameters.** Never build SQL by gluing strings together.

## Deliberately not copied

| Fineract has | Why MeshLedger doesn't |
|---|---|
| **Partial transfers** (my FINERACT-2400 `allow_partial_transfer`) | Right for a scheduled sweep from savings. Wrong for a payment someone already accepted: a ₹500 payment either settles in full or not at all. |
| **The full command framework** (legacy `CommandWrapper`, and the new typed `fineract-command`; I worked on its migration rules in FINERACT-2941/2942) | Fineract has 416 distinct `@CommandType` handlers on `develop`, which is where a command bus pays for itself. MeshLedger has one write path, so it would only be ceremony. |
| **Multi-tenancy** | One demo bank. |
| **GL chart of accounts** | See lesson 3. |
