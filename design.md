# Order & Inventory Sync Service: Design

## 1. Summary

A NestJS app with two parts, both on AWS ECS: an API (receives webhooks, serves the ops screen) and a Worker (responsible for running background work). They share one PostgreSQL database. Redis is used to hold the job queues and rate-limit counters as each service has different rate limits. The website's own database is separate from ours.

Stack: NestJS, PostgreSQL, Redis, AWS (ECS Fargate, RDS, ElastiCache, load balancer, CloudWatch). BullMQ is an npm library that keeps its jobs in Redis, so it is not a new service.

The idea in four lines:
- Save every order first, process it in the background.
- Reduce stock with one SQL update that refuses to go below zero. That stops overselling.
- Write every stock change to a history table for the audit trail.
- Anything that must reach a channel or the ERP is saved in a table first and sent from there, in case of failure it can be retried without data loss.

## 2. Assumptions

1. Channels sign their webhooks. Nykaa and Blinkit retry a webhook if we return an error.
2. SKU codes are the same on every channel, in our DB and in Microsoft Business Central.
3. Business Central accepts our order number as the External Document No. and rejects a duplicate.
4. The website DB is separate from ours. We get a limited DB user that can only update stock.
5. Warehouse stock changes (new stock, damage) are entered by ops with a CSV upload. Returns add stock back when ops marks them received.
6. Oversold marketplace orders are cancelled by ops in the seller panel.


## 3. Architecture
*(Please find in the base dir of repo as architecture.png)*

**How data flows**
*(Please find the diagram in the base dir of repo as flow.png)*

Order from a webhook channel (website, Nykaa, Blinkit). For Flipkart, the worker's poll job replaces steps 1 to 4 and saves the orders the same way.

Steps 1 to 4 are fast and never wait on anything else. Step 6 is the only place stock changes. Steps 8 and 9 are separate jobs, so one failing does not stop the other.

1. A webhook (website, Nykaa, Blinkit) reaches the API. The API saves the order as RECEIVED (a duplicate is ignored), adds a job in Redis and returns 200.
2. For Flipkart, the worker's poll job fetches new orders and saves them the same way.
3. The worker picks up the job and runs one database transaction: take the stock, write the history, mark the order CONFIRMED, flag the SKU for sync on all channels, and create an ERP row.
4. A push job per channel sends the latest quantity, inside that channel's rate limit.
5. An ERP job sends the order to MS Business Central from the ERP row.
6. A cancel takes the same path. Stock is added back, the SKU is flagged, and an ERP cancel row is created.
7. A failed step is retried. After 5 tries it shows on the ops screen with a retry button.

**Scheduled jobs.** Stock push does not need a timer, since it runs when stock changes. Only these do:

| Job | Interval | Why |
|---|---|---|
| Flipkart poll | 2 min | Flipkart is polling only |
| Retry sweeper | 1 min | Re-adds jobs for rows that are due again (ERP after Sunday maintenance, or anything lost if Redis restarts) |
| Blinkit CSV | Daily | Full reset, because the Blinkit API times out about 2% of the time |

They are BullMQ repeating jobs in the worker with a fixed job id, so only one runs even with 2 tasks. There is no separate cron service.

## 4. Data model

| Table | Main columns | Notes |
|---|---|---|
| inventory | sku, available | available >= 0 |
| stock_history | sku, change, reason, order_id, created_at | Insert only. Answers "why did stock change" |
| orders | id, channel, external_order_id, status, raw_payload | Unique (channel, external_order_id) |
| order_items | order_id, sku, qty, unit_price | |
| channel_stock_sync | sku, channel, needs_sync, status | One row per SKU and channel |
| erp_sync | order_id, type, status, attempts, next_try_at | Unique (order_id, type) |

Order status: RECEIVED, CONFIRMED, NO_STOCK, CANCELLED.

**Avoiding duplicate orders:** the unique key on (channel, external_order_id) with ON CONFLICT DO NOTHING. It also covers an order that arrives once by webhook and once by polling. A repeated cancel finds the order already cancelled and does nothing.

## 5. Order ingestion

- **Webhooks:** the API checks the signature, saves the order with its raw payload, adds a job and returns 200 quickly. Each channel has an adapter that converts its format into the common order format.
- **Flipkart:** the poll job asks for orders since the newest Flipkart order we have, minus 5 minutes. The overlap catches late updates and the unique key ignores repeats.
- **Out of order (Nykaa):** a cancel for an order we do not have yet is retried after 30 seconds, up to 10 times, then shown on the ops screen. If the cancel is already waiting when the create arrives, the order is saved as CANCELLED and takes no stock.
- **A channel is down:** each channel has its own queue, so it cannot slow the others. Flipkart catches up on the next successful poll.

## 6. Inventory consistency: the last unit

Everything for one order runs in a single database transaction. For each item the worker runs UPDATE inventory SET available = available - qty WHERE sku = ? AND available >= qty. If no row is updated, the whole transaction is undone. SKUs are processed in a fixed order so two orders cannot deadlock.

Two channels buy the last unit in the same second:
1. Both workers run the update. Postgres locks the row, so one goes first and the other waits.
2. The first takes the unit, available becomes 0, and it commits.
3. The second now sees 0, so the update changes nothing. Its transaction is undone and the order becomes NO_STOCK.
4. That order is not sent to the ERP. It appears on the ops screen as "cancel on the channel".
5. The SKU is flagged, so every channel is told it is 0.

We cannot sell stock we do not have. What can still happen is a second order arriving while a channel shows old stock for a few seconds. To reduce that, channels are shown available - 2 (a config value).

## 7. Stock push

After a stock change, the SKU is flagged on every channel and one push job per channel is added. The job reads the latest available and sends it. We always send the full quantity, never "minus 1", so a retry is safe and quick changes collapse into one call.

| Channel | Limit | How we stay inside it |
|---|---|---|
| Website | None | Update the quantity in the website DB. Still retried, since it is a different database |
| Nykaa | 5 req/s, 1 SKU per call | Limiter at 4 req/s, about 240 SKUs per minute |
| Flipkart | 10 req/s, 50 SKUs per call | 50 SKUs per call, limiter at 5 req/s. On a 429, wait and slow down |
| Blinkit | 2% timeouts | Limiter at 4 req/s, 5 second timeout, 3 retries, daily CSV as a reset |

On a sale day Nykaa is the slowest. A full refresh of 1,200 SKUs would take about 5 minutes. In practice repeated changes to one SKU become one call, sale orders are mostly for a few SKUs, and the lowest-stock SKUs are sent first because they are the ones that can oversell. At normal load, everything is well under 60 seconds.

## 8. Failure handling

| Case | What happens |
|---|---|
| Temporary error (timeout, 5xx, 429) | Retry with a growing delay (10s, 30s, 2 min, 10 min, 30 min) |
| Fails 5 times, or a permanent error | Marked FAILED, shown on the ops screen. Retry sets it back to pending |
| ERP down on Sunday | The ERP row was saved with the order, so nothing is lost. The sweeper retries every few minutes and sends the backlog when Business Central is back (about 700 orders after a normal 2 hours, cleared in minutes) |
| Same order sent to ERP twice | We send our order number as the External Document No. After a timeout we check Business Central for it before resending |
| Order cancelled before reaching the ERP | Both ERP rows are skipped |
| Blinkit timeouts | Retried. Safe, because we send the full quantity each time |
| Redis loses jobs | Everything is still in Postgres. The sweeper re-adds it within a minute |

## 9. Observability (CloudWatch)

1. Stock sync delay per channel. Alert above 60 seconds.
2. Oldest pending ERP row. Alert above 15 minutes, relaxed during Sunday maintenance.
3. Share of NO_STOCK orders in the last hour. Alert above 0.3%, since the limit is 0.5%.
4. Channel API errors (timeouts, 429s, 5xx) per channel.
5. Failed items older than 30 minutes, and no orders from a channel for a long time.

## 10. Left out of v1

- Second warehouse and the LLM question feature.
- Automatic cancel of oversold orders on marketplaces. Ops does it by hand.
- Return quality checks. Ops marks a return as received.
- Kafka, SQS, Grafana and a separate frontend app. The volume is small and CloudWatch plus plain server-rendered pages are enough.
- SSO and roles for the ops screen.

## 11. Trade-offs

| Decision | Chose | Rejected | Why |
|---|---|---|---|
| Stock check | One SQL update in Postgres | Redis lock or counter | One source of truth. It commits together with the order and history. At 5 orders a second, row locking is no problem |
| Queue | BullMQ on Redis | SQS or Kafka | Redis is already in the stack and BullMQ has retries, delays and rate limits built in |
| Sending to ERP and channels | Save to a table, send from a worker | Call the ERP while processing the order | If the ERP is down, orders are still saved and sent later. The cost is a small delay |
| Scheduled work | BullMQ repeating jobs in the worker | A cron calling the API | No extra service, and a fixed job id stops 2 tasks running the same job |
| Ops screen | Server-rendered pages in NestJS | A React app | React is not in the allowed stack and needs its own build. Less polished |

**Infra cost (rough, check in the AWS calculator):** about 35k per month in total. ECS 6k, RDS Multi-AZ 13k, ElastiCache 5k, load balancer 2.5k, CloudWatch and others 8k.

## 12. AI usage

AI tool was used to help draft this document and get to know more about node services like bullmq, as i have more experience in python could have done using celery but for node required more context.