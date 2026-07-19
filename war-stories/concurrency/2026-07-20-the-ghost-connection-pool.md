# 2026-07-20 - The Ghost Connection Pool

> **Stack:** Java 21 | Spring Boot 3.2 | PostgreSQL 15 | HikariCP (default config)
> **Outage Duration:** 47 minutes
> **Impact:** 100% of order-history API requests timing out, cascading into the checkout flow; ~40k users affected

---

## 🌩️ The Calm

It was a Tuesday. Traffic was normal — about 800 requests per minute against our order service, a standard Spring Boot monolith with a HikariCP connection pool sized at the default **10 connections** against a PostgreSQL 15 instance that could comfortably handle 200.

That morning's deploy included a "harmless" data-cleanup feature: an admin endpoint to purge orphaned order records older than seven years. A compliance ask. The developer tested it against staging with a few thousand rows. It ran in two seconds.

Production had **2.3 million** orphaned rows.

At 14:03 UTC, someone from compliance clicked the button. By 14:05, every single endpoint in the service — even ones that didn't touch the `orders` table — was returning `504 Gateway Timeout`. The service wasn't crashed. It wasn't erroring. It was just... frozen. A ghost.

---

## 💥 The Trigger

```java
@RestController
@RequestMapping("/admin/orders")
public class OrderCleanupController {

    private final OrderCleanupService cleanupService;

    public OrderCleanupController(OrderCleanupService cleanupService) {
        this.cleanupService = cleanupService;
    }

    @PostMapping("/purge-orphans")
    @Transactional  // <-- THE BOMB
    public ResponseEntity<String> purgeOrphanedOrders() {
        long deleted = cleanupService.deleteOrphanedOrders();
        return ResponseEntity.ok("Purged " + deleted + " orphaned orders.");
    }
}
```

```java
@Service
public class OrderCleanupService {

    private final JdbcTemplate jdbcTemplate;

    public OrderCleanupService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public long deleteOrphanedOrders() {
        // Deletes ALL orphaned rows in ONE statement — inside the caller's transaction.
        return jdbcTemplate.update("""
            DELETE FROM orders o
            WHERE NOT EXISTS (
                SELECT 1 FROM customers c WHERE c.id = o.customer_id
            )
            AND o.created_at < NOW() - INTERVAL '7 years'
            """);
    }
}
```

One `@Transactional` annotation on the controller method meant **one database connection** was checked out of HikariCP for the entire duration of a 2.3-million-row `DELETE`. PostgreSQL had to build a massive snapshot, hold locks, and generate gigabytes of WAL. The delete ran for minutes.

Ten pool connections, one giant transaction... so where did the *other nine* connections go? They were grabbed by incoming HTTP requests — normal order-history reads — which then piled up *behind the row locks and table-level contention* the mega-delete created. And once all ten connections were checked out and waiting, every subsequent request queued on `HikariDataSource.getConnection()` until it hit the default 30-second `connectionTimeout`.

---

## 🕵️ The Hunt

**Step 1: Is the app even alive?**

CPU was low. Memory was fine. No OOM, no crash loop in the logs. Just this, repeating:

```
SQLTransientConnectionException: HikariPool-1 - Connection is not available,
request timed out after 30000ms.
```

Classic pool exhaustion. But the pool metrics showed no leaked connections being *closed* improperly — connections were checked out and simply never returned. So what was holding them?

**Step 2: Thread dump — where are the threads parked?**

```bash
jstack <pid> > threads.txt
grep -c "getConnection" threads.txt
```

Dozens of HTTP worker threads, all stacked up in one of two places:

```
"http-nio-8080-exec-47" #212 prio=5 WAITING
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:197)
    ...

"http-nio-8080-exec-12" #177 prio=5 TIMED_WAITING
    at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:376)
    ...
```

Some threads were *waiting for a connection*. Others *had* a connection and were stuck inside a query. The database was the bottleneck. Time to ask PostgreSQL directly.

**Step 3: What is the database actually doing right now?**

```sql
SELECT pid, state, wait_event_type, wait_event,
       NOW() - query_start AS age,
       LEFT(query, 120) AS query
FROM pg_stat_activity
WHERE datname = 'orders_db'
  AND state <> 'idle'
ORDER BY query_start;
```

And there it was — the smoking gun:

```
 pid   | state  | wait_event_type | wait_event | age      | query
-------+--------+-----------------+------------+----------+---------------------------------------
 48112 | active |                 |            | 00:14:32 | DELETE FROM orders o WHERE NOT EXISTS...
 48133 | active | Lock            | tuple      | 00:11:05 | SELECT * FROM orders WHERE customer_id =...
 48134 | active | Lock            | tuple      | 00:10:48 | SELECT * FROM orders WHERE customer_id =...
```

**One delete, running for 14 minutes**, and a growing crowd of ordinary `SELECT`s blocked behind its locks. Every blocked select held a Hikari connection. Every held connection starved the pool. Every starved request became a 504. The ghost had a name.

**Step 4: The "aha"**

The `@Transactional` on the controller was doing more than wrapping the delete — it made the *entire HTTP request* one transaction, so even if the developer had added batching inside the service, it would all still commit as one monstrous unit. The annotation was added "for safety." It provided the exact opposite.

**Step 5: Stop the bleeding**

```sql
SELECT pg_cancel_backend(48112);
```

Fourteen minutes of work rolled back (that took another three minutes), the locks released, the queue drained, and the service recovered on its own. No restart needed.

---

## 🔧 The Cure

**1. Remove `@Transactional` from the controller.** Transactions belong in the service layer, scoped as narrowly as possible.

**2. Chunk the delete.** Never let one transaction touch millions of rows. Each batch gets its own short transaction, holds locks for milliseconds, and yields between rounds:

```java
@Service
public class OrderCleanupService {

    private static final int BATCH_SIZE = 5_000;
    private final TransactionTemplate transactionTemplate;
    private final JdbcTemplate jdbcTemplate;

    public OrderCleanupService(PlatformTransactionManager txManager,
                               JdbcTemplate jdbcTemplate) {
        this.transactionTemplate = new TransactionTemplate(txManager);
        this.jdbcTemplate = jdbcTemplate;
    }

    public long deleteOrphanedOrders() {
        long totalDeleted = 0;
        int deletedInRound;

        do {
            // Each batch runs in its OWN short transaction.
            deletedInRound = transactionTemplate.execute(status ->
                jdbcTemplate.update("""
                    DELETE FROM orders
                    WHERE id IN (
                        SELECT o.id FROM orders o
                        WHERE NOT EXISTS (
                            SELECT 1 FROM customers c
                            WHERE c.id = o.customer_id
                        )
                        AND o.created_at < NOW() - INTERVAL '7 years'
                        LIMIT ?
                    )
                    """, BATCH_SIZE)
            );

            totalDeleted += deletedInRound;

            // Yield between batches so normal traffic can breathe.
            if (deletedInRound == BATCH_SIZE) {
                Thread.sleep(100);
            }
        } while (deletedInRound == BATCH_SIZE);

        return totalDeleted;
    }
}
```

**3. Add a safety rail to HikariCP** so a runaway query can't hold a connection forever:

```properties
spring.datasource.hikari.max-lifetime=600000
# Kill any connection checked out longer than 30s (logs the offending stack trace):
spring.datasource.hikari.leak-detection-threshold=30000
```

The `leak-detection-threshold` alone would have turned this 47-minute ghost hunt into a 5-minute log read — it prints the exact stack trace of whoever is holding the connection hostage.

The full purge of 2.3 million rows re-ran that night in ~500 chunked batches. Total time: 6 minutes. Lock impact on production traffic: zero.

---

## 📜 The Golden Rule

> **Never let a single transaction touch an unbounded number of rows — chunk destructive operations into small, short-lived transactions, and let the pool breathe between them.**

---

*Names and details sanitized to protect the guilty. The pain was real.*
