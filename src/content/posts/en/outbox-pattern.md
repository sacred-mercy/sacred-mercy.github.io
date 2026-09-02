---
title: "The Outbox Pattern: Reliable Events in distributed system"
description: "Why writing to your database and publishing to a message broker in the same request is unsafe, and how the transactional outbox pattern fixes it."
pubDate: 2026-09-02
categories: [Architecture]
tags: [distributed-systems, microservices, messaging, databases, patterns, cdc]
---

You upsert an order to Database, then publish an `OrderPlaced` event to Kafka. Two systems, one request handler, no shared transaction. Most of the time it works but the order upsert and event publish are not **Atomic**. That "most of the time" is the whole problem. 

## 1. The Dual-Write Problem

Here is the code almost everyone writes first:

```js
await db.orders.insert(order);          // (1) commits
await broker.publish('OrderPlaced', e); // (2) may fail
```

There are two ways this breaks:

- **(2) fails after (1) commits.** The order exists, but nobody downstream knows about the change. No confirmation email, no inventory reservation. The data is silently inconsistent.
- **Reverse the order** and publish first, and now (1) can fail after the event is out. Downstream services react to an order that does not exist. This is worse — you cannot un-send a message.

Wrapping both in a database transaction does not help, because the broker is not part of it. Retrying does not help either: a retry of (2) can succeed *and* the original can have succeeded, so you get duplicates. A retry of the whole handler can double-insert the order.

The root cause is that you are attempting an atomic write across two systems that share no commit protocol.

## 2. The Pattern

Write the event into the *same database*, in the *same transaction*, as the business data. A separate process reads those rows and publishes them.

```sql
CREATE TABLE outbox (
  id           BIGSERIAL PRIMARY KEY,
  aggregate_id TEXT        NOT NULL,
  event_type   TEXT        NOT NULL,
  payload      JSONB       NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at TIMESTAMPTZ
);

CREATE INDEX outbox_unpublished_idx
  ON outbox (id) WHERE published_at IS NULL;
```

The write path becomes a single local transaction:

```js
await db.tx(async (t) => {
  const order = await t.orders.insert(orderData);
  await t.outbox.insert({
    aggregate_id: order.id,
    event_type: 'OrderPlaced',
    payload: toEvent(order),
  });
});
```

Either both rows are there or neither is. There is no window where the order exists without its pending event, and the request handler never talks to the broker at all — so broker downtime no longer fails user-facing writes.

The partial index matters: the unpublished set stays small even when the table is large, so the poll query touches a handful of pages instead of scanning history.

## 3. Getting Events Out: The Polling Publisher

A worker claims a batch of unpublished rows, publishes them, marks them done:

```sql
-- claim
UPDATE outbox SET published_at = now()
WHERE id IN (
  SELECT id FROM outbox
  WHERE published_at IS NULL
  ORDER BY id
  FOR UPDATE SKIP LOCKED
  LIMIT 100
)
RETURNING *;
```

`FOR UPDATE SKIP LOCKED` is what makes this safe to run on several workers at once — each grabs a disjoint batch instead of blocking on the same rows. Simple, portable, and adds one poll interval of latency.

Note the ordering trap: if you mark rows published *before* the broker acknowledges, a crash loses the event. Mark them after, or claim with a `locked_until` lease and only set `published_at` post-ack. Losing an event is much worse than sending it twice.

This is the whole publisher: a claim query, a publish loop, and an interval. Portable across every database, no extra infrastructure, and the cost is one poll interval of latency plus a steady trickle of queries against the primary.

## 4. Do We Need an Outbox Table in Every Database?

Follow the pattern above across a dozen services and you have a dozen outbox tables, a dozen publisher workers to deploy and monitor, and a dozen retention jobs. That is a lot of duplicated plumbing for what is really one concern: *get committed changes out of this database and onto the broker*.

Change Data Capture answers it differently. Every database already writes an ordered, durable log of committed changes so replicas can catch up — Postgres calls it the WAL, MySQL the binlog, MongoDB has the oplog behind change streams. **Debezium** reads that log and turns each committed row change into an event on the broker. If the database already has a log of everything that happened, you do not need the application to write a second one.

### How Debezium Attaches to the Database

For Postgres, Debezium registers as a logical replication client — the same interface a read replica uses:

```sql
ALTER SYSTEM SET wal_level = logical;   -- restart required
CREATE PUBLICATION dbz_pub FOR TABLE public.orders, public.outbox;
```

Then it runs as a Kafka Connect connector pointed at the database:

```json
{
  "name": "orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "orders-db",
    "database.dbname": "orders",
    "plugin.name": "pgoutput",
    "slot.name": "dbz_orders",
    "publication.name": "dbz_pub",
    "topic.prefix": "orders",
    "table.include.list": "public.orders,public.outbox"
  }
}
```

On startup it takes a consistent snapshot of the included tables, then switches to streaming: it opens a replication slot, decodes WAL records as they are written, and emits one change event per committed row — with `before` and `after` images and the operation (`c`, `u`, `d`) — to a topic per table. No polling, no queries against your tables, latency in the low milliseconds. The slot is what makes it reliable: Postgres will not recycle WAL segments the slot has not acknowledged, so a connector that restarts resumes exactly where it stopped.

That same slot is the operational catch. **A connector that stays down retains WAL forever, and the primary's disk fills.** Alert on `pg_replication_slots` lag, and set `max_slot_wal_keep_size` so the database sheds the slot rather than dying — you lose the stream and have to re-snapshot, which is strictly better than an unplanned outage on the primary.

### Two Ways to Use It

**Stream the business tables directly.** No outbox table anywhere. Debezium publishes `orders` row changes and downstream services consume those. Zero application code — but what you are publishing is now your table schema. Consumers couple to column names, a rename becomes a breaking change for other teams, and you cannot express anything that is not a row: no domain event like `OrderCancelled` with a reason, no event that spans three tables, no way to omit a column you would rather not broadcast. It works well for replication-shaped problems (feeding a search index, a warehouse, a cache) and poorly as a public domain-event contract.

**Keep the outbox table, delete the publisher.** You still write the event you actually mean, inside the business transaction, so the payload stays a deliberate contract. But nothing polls it — Debezium tails it, and the [outbox event router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html) SMT reshapes the row into a proper message:

<!--```json
"transforms": "outbox",
"transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
"transforms.outbox.route.by.field": "aggregate_type",
"transforms.outbox.table.field.event.key": "aggregate_id",
"transforms.outbox.table.field.event.payload": "payload"
```-->

<!--The row goes out on a topic chosen by `aggregate_type` (so `Order` rows land on `outbox.event.Order`), keyed by `aggregate_id`, with `payload` unwrapped as the message body — the envelope columns never reach consumers. One connector per database replaces a worker per service, and you get millisecond latency instead of a poll interval.-->

There is a neat consequence: the outbox rows only ever need to *exist in the WAL*, not in the table. Debezium sees the `INSERT` in the log regardless of what happens next, so you can insert and delete the row in the same transaction and keep the table permanently empty — no retention job at all. (Debezium ignores the matching delete by default; if it doesn't, filter `op` in a transform.)

### How Debezium Preserves Order

The guarantee comes from the log itself. The WAL is a total order of commits, Debezium reads it single-threaded per connector, and it emits events in that order — LSN by LSN, transaction by transaction, with a row's events in the order they committed. It never reorders and never interleaves two transactions.

What can lose that order is Kafka, not Debezium. Ordering in Kafka holds *within a partition*, so:

- Keyed by `aggregate_id`, every event for one order hashes to the same partition and arrives in commit order. This is the guarantee you almost always want, and the router gives it to you by default.
- Across different orders, events land on different partitions and may interleave. Total global ordering needs a single partition, which caps throughput at one consumer — rarely worth it.

Restart behaviour is where duplicates come from. Debezium checkpoints its LSN to the Connect offsets topic periodically, not per event, so a crash replays everything after the last flushed offset. Order is preserved on replay; uniqueness is not. Which brings us back to the same requirement as the polling worker.

## 5. At-Least-Once, So Consumers Must Be Idempotent

Whichever publisher you pick, the guarantee is **at-least-once**. A worker can publish and die before recording success; CDC can replay from an older offset. Duplicates are not an edge case, they are the contract.

So give every event a stable id and make consumers deduplicate:

```js
async function handle(event) {
  const fresh = await db.processed.insertIfAbsent(event.id);
  if (!fresh) return;            // already handled
  await applyBusinessLogic(event);
}
```

<!--Do the dedupe insert and the side effect in one transaction where you can. If the side effect is an external call, make it idempotent too — pass the event id as an idempotency key.-->
<!---->
<!--Ordering needs the same scrutiny on the polling side. One worker claiming in `id` order preserves it; several workers with `SKIP LOCKED` do not — batch two can publish before batch one finishes. Key by `aggregate_id` there too, and treat per-entity order as the only ordering you actually have.-->

## 6. Housekeeping

The outbox is a queue, not a log. Delete or partition-drop published rows on a schedule — a nightly `DELETE ... WHERE published_at < now() - interval '7 days'` is usually enough. Left alone, the table grows forever, autovacuum falls behind on the dead tuples, and the poll query slows down even with the partial index.

On MongoDB you can skip the cron job entirely and let a **TTL index** do it:

```js
db.outbox.createIndex(
  { publishedAt: 1 },
  { expireAfterSeconds: 604800 }   // 7 days after publish
);
```

A background thread sweeps roughly every 60 seconds and drops documents whose `publishedAt` is older than the window. The detail that makes this safe: TTL only expires documents where the indexed field is an actual date, so unpublished events — where `publishedAt` is missing or `null` — are never touched, however long they sit there. Retention and correctness come from the same index, with nothing to schedule. Set the window from your consumers' worst-case replay need, not from disk pressure; once a document is gone, so is the ability to re-publish it.

Two things worth alerting on:

- **Oldest unpublished row age.** This is your real end-to-end lag. A rising number means the publisher is stuck or the broker is rejecting writes.
- **Unpublished row count.** Catches the case where the publisher is running but not keeping up with write volume.

## 7. When To Skip It

The outbox is not free — an extra table, a worker to operate, idempotent consumers, a retention job. Skip it when:

- The event is genuinely best-effort (cache warm-up, non-critical metrics).
- The consumer can derive what it needs by reading your database or polling an API.
- The broker *is* your database — if you are already using a Postgres-backed job queue in the same transaction, you have the pattern for free.

Reach for it when a lost event means a customer never gets charged, an order never ships, or two services disagree about what happened. That is most of the interesting cases.

---

The idea generalizes past messaging: any time you must do a local commit and a remote effect together, commit the *intent* locally and let a separate process carry it out with retries. The outbox is that idea applied to events.
