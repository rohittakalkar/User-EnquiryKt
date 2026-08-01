# Utils Programming Guide — Redis, Kafka, RabbitMQ, Consumers, DB, Webhooks

A practical, code-level reference for the shared utility functions across all three repos.
Use this when you want to **build a new feature** on top of the system design already
documented (e.g. "push a new event on Kafka," "add a Redis cache," "write a new consumer
that reads from Postgres"). Every function below is real, copy-pasteable, and cited with its
file path — nothing here is invented.

This doc only covers **utils/shared plumbing**, not business logic. For which controller/
consumer to actually modify for a given feature, see
[`users_domain_product_stories.md`](./users_domain_product_stories.md) and the five
read/write-picture docs.

---

## 1. Redis — cache reads/writes

**Where:** [`users-api-go-production/pkg/config/rediscache.go`](../pkg/config/rediscache.go) (read API) and
[`service-api-go-production/pkg/config/rediscache.go`](../../service-api-go-production/service-api-go-production/pkg/config/rediscache.go) (write API — same pattern).
Also a second, independent Redis client exists in the consumers repo:
[`user-temp-consumers-production/pkg/utils/db.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/db.go) (`InitRedisClient`).

**Client init** — lazy singleton, wrapped in a circuit breaker (`CB_redis`, defined in
[`rediscircuitbreaker.go`](../pkg/config/rediscircuitbreaker.go), trips after 40 consecutive failures):

```go
client, err := config.GetRedis()
```

**Write a string value with TTL:**

```go
err := config.RedisSet("mykey", "myvalue", 5*time.Minute)
```

**Write raw bytes / a struct (auto-marshals structs and maps to JSON, passes strings and
[]byte through untouched):**

```go
err := config.RedisSet_im_cat("mykey", myStructOrMap, 10*time.Minute)
```

**Read a string value** (also returns the connection time in ms, useful for latency
logging to Kibana):

```go
value, err, connMs := config.RedisGet("mykey")
```

**Read raw bytes:**

```go
data, err, connMs := config.RedisGet_Im_CAT("mykey")
```

**Set-based cache (e.g. "which category IDs has this user excluded"):**

```go
err := config.RedisSAdd("myset", "member1")
members, err := config.RedisSMembers("myset")
```

**Consumer-side Redis** (a worker needs its own cache, separate from the API's client):

```go
redisClient, err := utils.InitRedisClient("redis_instance_name") // reads redis.<name>.host/port/password from config
```

**Practical scenario:** caching an expensive read (e.g. `GET /statutory_details`, which
fans out to 3+ downstream services) — call `RedisGet` first, fall back to the real fetch on
a cache miss (`err == nil && value == ""` from `redis.Nil`), then `RedisSet` the result with
a short TTL.

---

## 2. Kafka — produce and consume

**Producing (HTTP-bridged, not a native Kafka client)** — both repos publish to Kafka via
an internal HTTP proxy (`kafka_publish` API / `soa-kafka-pub-api`), not `segmentio/kafka-go`
directly. Two near-duplicate functions exist:

[`user-temp-consumers-production/pkg/utils/functions.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/functions.go):
```go
resp, err := utils.CallKafkaService(dataMap, "some_config_key_for_the_url")
// POSTs dataMap as JSON to the URL at config key `servicenm`,
// header: application/vnd.kafka.json.v2+json
```

[`service-api-go-production/pkg/utils/globalfunctions.go`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go):
```go
resp, err := utils.CallKafkaService(dataMap)
// POSTs to configData.APIList["kafka_publish"]
```

`service-api-go-production/pkg/utils/globalfunctions.go` also has a topic-explicit variant:
```go
status, err := utils.KafkaPublish(message, messageLog, "my_topic_name")
// POSTs {"rservice": "PRODUCT_SERVICE", "data": message, "topic": Topic}
// to http://soa-kafka-pub-api.intermesh.net/kafkapublish (prod) or the dev host
```

**Consuming (native Kafka client, consumer repo only)** — this is where the actual
`segmentio/kafka-go` reader lives:
[`user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go):

```go
func InitializeKafka(queuenm, sub_topic, consumergrp string, dbConnStruct utils.DBConnStruct) {
    kafkaReader = kafka.NewReader(kafka.ReaderConfig{
        Brokers:        utils.Brokers(),
        GroupID:        consumergrp,
        Topic:          sub_topic,
        CommitInterval: CommitInterval,
        StartOffset:    kafka.LastOffset,
    })
    // loop: msg, err = kafkaReader.ReadMessage(ctx)
    // dispatches to KafkaDBFuncMap[queuenm](msg, dbConnStruct, queuenm)
}
```

Every Kafka-driven worker (e.g. `USER_BS_MATCHMAKING`, `USER_BUSINESS_FEED`,
`USER_PERSONALIZATION_ACTIVITY`, `USER_HISTORY`, `USER_FLIPS_CONTENT_SYNC`) is registered
in `KafkaDBFuncMap` in the same file (`IntializeMsgBroker.go`, line ~247) — that map is the
single place to look to see which workers are Kafka-driven vs. RabbitMQ-driven.

**Practical scenario:** to make a new consumer Kafka-driven instead of RabbitMQ-driven —
(1) add its handler function to `KafkaDBFuncMap` with signature
`func(d kafka.Message, dbConnStruct utils.DBConnStruct, queuenm string)`, (2) call
`InitializeKafka(queuenm, topic, group, dbConnStruct)` from its entrypoint instead of
`InitializeRabbitMqv1`, (3) register the queue name in `QueueFuncMap` in
`internal/Router/router.go` as usual.

---

## 3. RabbitMQ — publish and consume

**Publishing — the central function every write controller calls:**
[`service-api-go-production/pkg/utils/rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) (and a near-identical copy in
[`users-api-go-production/pkg/utils/rabbitmq.go`](../pkg/utils/rabbitmq.go) for the read API's own async needs):

```go
data := map[string]interface{}{
    "SERVICENAME": "SUPPLIER_FEEDBACK",   // looked up in an internal service→queue map
    "glusrid":     glusrId,
    // ...whatever fields the target consumer expects
}
result := utils.PushToQueue(glusrId, data)
// resolves queue_name + exchange from a hardcoded SERVICENAME → queue map (see below),
// stamps UNIQUE_LOGGING_ID + TIMESTAMP, then calls RabbitEnqueue()
```

`PushToQueue`'s internal `serviceToQueueMap` (in `rabbitmq.go`) is the master list of every
`SERVICENAME` string and which literal queue it maps to — e.g. `"SUPPLIER_FEEDBACK"` →
`"user.supplierrating.<glusrid % 20>"`, `"GLUSR_UPDATE_SERVICE"` → `"USER_CENTRALIZED_QUEUE"`.
**To wire a new write-path to a new consumer, add an entry here.**

Under the hood, `RabbitEnqueue` doesn't dial RabbitMQ directly — it calls a lightweight
internal HTTP publish API (`/rmq/publish`) with automatic retry (5 attempts) and a
same-datacenter failover URL (`failoverRabbitEnqueTemp`) if the primary fails. There's also
a raw-file fallback: if even the failover fails for specific queues
(`USER_DETAIL_ENCR`, `user.upsert.*`), the message gets appended to a local TD-agent file
for later reprocessing rather than being dropped.

**Lower-level variant if you need explicit control over host/queue/exchange:**
```go
msg := utils.RabbitEnqueueData(data, host, queueName, uniqueID, serviceName, exchange, port)
```

**Consuming — native `streadway/amqp` client, consumer repo:**
[`user-temp-consumers-production/pkg/utils/rabbitmq.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/rabbitmq.go):

```go
channel, notifyClose := utils.ConnectCluster() // dials node1, falls back to node2, retries every 500ms
```

[`user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go):
```go
func InitializeRabbitMqv1(queuenm string, dbConnStruct utils.DBConnStruct) {
    channel, notify = utils.ConnectCluster()
    channel.Consume(queuenm, "", false /*auto-ack*/, false, false, false, nil)
    // loop: on delivery d, dispatch to DatabaseFuncMapv1[queuenm](d, dbConnStruct, queuenm)
    // on channel death (notify), reconnect recursively
}
```

Every RabbitMQ-driven worker is registered in `DatabaseFuncMapv1` in the same file
(line ~336) — mirror of `KafkaDBFuncMap` for the AMQP side. (There's also an older
`DBFuncMap`/`InitializeRabbitMq` pair still present — treat `v1` as the current pattern for
anything new.)

**Practical scenario:** to add a brand-new async side effect from an existing write
controller — (1) add a `SERVICENAME` → queue-name mapping in `PushToQueue`'s
`serviceToQueueMap`, (2) call `utils.PushToQueue(glusrId, map[string]interface{}{"SERVICENAME": "MY_NEW_EVENT", ...})`
from the controller, (3) write a new worker function, register it in `DatabaseFuncMapv1`
and `QueueFuncMap`.

---

## 4. Consumers — worker structure and dispatch

**Where:** [`user-temp-consumers-production/internal/Router/router.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) (entrypoint) +
`internal/Workers/*.go` (one file per queue, ~76 files).

**Deployment model:** one binary, one running instance per queue — the queue name is
passed in as `CONSUMERNAME` (an env var / CLI arg), and `RouteConsumers` looks it up:

```go
func RouteConsumers(consumername string) {
    utils.LoadDefaultConfig(mode, consumername)
    utils.CreateDBStringMap(mode)
    utils.InitiateLog()
    QueueFuncMap[consumername](consumername)  // dispatch to the worker's entrypoint
}
```

**A typical worker's shape** (from [`USER_RATING_AGGREGATE.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go)):
```go
func UserRatingAggregate(queuenm string) {
    defer utils.RecoverPanic(queuenm)
    var dbConnStruct utils.DBConnStruct
    err := utils.GetDatabaseConnectionv1("meshPg", "postgres", &dbConnStruct) // open the DB(s) this worker needs
    if err != nil { return }
    InitializeRabbitMqv1(queuenm, dbConnStruct)  // or InitializeKafka(...) for Kafka-driven ones
}

func workerRatingAggregate(d amqp.Delivery, dbConnStruct utils.DBConnStruct, queuenm string) {
    defer func() {
        if r := recover(); r != nil {
            utils.Mail("PANIC IN "+queuenm, fmt.Sprintf("%+v", r))
            d.Nack(false, false)   // requeue-free nack on panic
        }
    }()
    var message map[string]interface{}
    json.Unmarshal(d.Body, &message)
    // ...validate expected keys, then do the DB work...
    d.Ack(false)  // manual ack only after successful processing
}
```

**Concurrency control:** `GoroutinesConsumers` (a `map[string]int` in
`IntializeMsgBroker.go`) sets a per-queue fan-out factor — e.g. `USER_PERSONALIZATION_ACTIVITY: 20`
spins up to 20 goroutines processing messages concurrently, using `gorc.Gorc` as a
semaphore (`Gorclock.Inc()` / `Gorclock.WaitLow(n)`) so you don't unbounded-goroutine a queue.
Any queue not listed defaults to a fan-out of 1 (`Contains` returns 1 if the key is absent).

**Practical scenario — building a brand-new consumer:** copy the `UserRatingAggregate` /
`workerRatingAggregate` two-function shape, register the entrypoint function in
`QueueFuncMap` (`router.go`) under your new queue name, register the message-handler
function in `DatabaseFuncMapv1` (or `KafkaDBFuncMap` for Kafka) in `IntializeMsgBroker.go`,
and (optionally) add a fan-out entry to `GoroutinesConsumers` if you expect high throughput.

---

## 5. Accessing the DB from a consumer

**Where:** [`user-temp-consumers-production/pkg/utils/db.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/db.go).

**Get/open a connection** (cached by name in a package-level map, so repeated calls are
cheap):

```go
var dbConnStruct utils.DBConnStruct
err := utils.GetDatabaseConnectionv1("meshPg", "postgres", &dbConnStruct)
// dbConnStruct.SqlDbConnSlice[0] is now a live *sql.DB

// or, for a Cassandra-backed worker:
err := utils.GetDatabaseConnectionv1("csl", "cassandra", &dbConnStruct)
// dbConnStruct.CassDbConnSlice[0] is now a live *gocql.Session

// or Redis:
err := utils.GetDatabaseConnectionv1("myredis", "redis", &dbConnStruct)
// dbConnStruct.RedisConnSlice[0]
```
`dbName` (`"meshPg"` above) maps to a config block `postgres.meshPg.host/port/db/username/password`
in the deployed config file — check `data/config.yaml`-equivalent for the consumers repo for
the exact list of registered DB names.

**Older single-connection variant** (still used by some legacy workers):
```go
db := utils.GetDb("meshPg", "postgres")   // returns *sql.DB directly, no struct wrapper
```

**Run a query and get results as `[]map[string]interface{}`** (handles retries, timeouts,
and `[]uint8` → `string` byte-array coercion automatically):

```go
rows, err := utils.GetDataSqlAndReturnArrayContext(
    context.Background(),
    mesh_pg_conn,
    "SELECT * FROM GLUSR_RATING_AGGREGATE WHERE FK_GLUSR_SUPPLIER_ID = $1",
    []interface{}{supplierId},
    2*time.Second,  // per-attempt timeout
    3,              // retry attempts
)
```

**Writes** go through plain `database/sql` (`dbConn.Exec` / `ExecuteQueryRows` helpers found
in `globalfunctions.go`-equivalent files per repo) — there's no separate "write helper" for
consumers; the same `*sql.DB` handle is used for both reads and writes.

**Practical scenario:** a new consumer that needs to read from `mesh` and write an aggregate
back — open the connection once in the entrypoint function (`GetDatabaseConnectionv1`), pass
`dbConnStruct` through to the message handler (as `workerRatingAggregate` does), and reuse
the same `*sql.DB` for both the `SELECT` (via `GetDataSqlAndReturnArrayContext`) and the
`INSERT`/`UPDATE` (via a direct `.Exec` call) — don't re-open a connection per message.

---

## 6. Webhooks

**Important finding:** none of the three repos expose or consume a true **inbound**
webhook (an HTTP endpoint that a third party calls to push an event in). What the codebase
calls a "webhook" is actually an **outbound callback-parameters object** — a small map
(`iilglusrid`, `sentdate`, `umailid`) attached to requests sent *to* an internal mail-sending
microservice, so that service can later report delivery status back asynchronously. See
[`user-temp-consumers-production/pkg/utils/functions.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/functions.go) (search `webhook_params`, used in
mail-sending helpers around lines 965, 1442, 1698).

```go
webhook := map[string]interface{}{
    "iilglusrid": glusrId,
    "sentdate":   sentDateString,
    "umailid":    uniqueMailId,
}
jsonArray["webhook_params"] = webhook
// POSTed to the centralized mail URL (config key centralizedmailurlgcp.url)
```

The closest things to genuine external webhook-style integrations in this domain are the
**Social Reviews** sync ([`SocialReviewContentController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewContentController.go) pulling from Google/Facebook — see
[`ratings_reviews_product_overview.md`](./ratings_reviews_product_overview.md)) and the
**Flips content sync** ([`FlipsContentSync`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go) pushing to `flips_content_write_service` — see
[`ecommerce_social_commerce_read_write_picture.md`](./ecommerce_social_commerce_read_write_picture.md)),
but both are outbound HTTP calls this codebase initiates, not inbound receivers. If a future
feature needs a real inbound webhook (e.g. a payment gateway calling back on transaction
status), there's no existing pattern to copy in these repos — it would need a new public
route + signature verification, built from scratch.

---

## Quick index: file paths

| Concern | File |
|---|---|
| Redis (read API) | [`users-api-go-production/pkg/config/rediscache.go`](../pkg/config/rediscache.go), [`rediscircuitbreaker.go`](../pkg/config/rediscircuitbreaker.go) |
| Redis (write API) | [`service-api-go-production/pkg/config/rediscache.go`](../../service-api-go-production/service-api-go-production/pkg/config/rediscache.go) |
| Redis (consumers) | [`user-temp-consumers-production/pkg/utils/db.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/db.go) (`InitRedisClient`) |
| Kafka produce | [`service-api-go-production/pkg/utils/globalfunctions.go`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go), [`user-temp-consumers-production/pkg/utils/functions.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/functions.go) (`CallKafkaService`, `KafkaPublish`) |
| Kafka consume | [`user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) (`InitializeKafka`, `KafkaDBFuncMap`) |
| RabbitMQ publish | [`service-api-go-production/pkg/utils/rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go), [`users-api-go-production/pkg/utils/rabbitmq.go`](../pkg/utils/rabbitmq.go) (`PushToQueue`, `RabbitEnqueue`) |
| RabbitMQ consume | [`user-temp-consumers-production/pkg/utils/rabbitmq.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/rabbitmq.go) (`ConnectCluster`), [`internal/Workers/IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) (`InitializeRabbitMqv1`, `DatabaseFuncMapv1`) |
| Consumer dispatch/routing | [`user-temp-consumers-production/internal/Router/router.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) (`QueueFuncMap`) |
| DB access from consumers | [`user-temp-consumers-production/pkg/utils/db.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/db.go) (`GetDatabaseConnectionv1`, `GetDataSqlAndReturnArrayContext`) |
| Webhook-shaped code (outbound only) | [`user-temp-consumers-production/pkg/utils/functions.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/functions.go) (`webhook_params` in mail helpers) |

---

## See also

- [`full_read_write_picture.md`](./full_read_write_picture.md) — how these primitives
  compose into the supplier-rating end-to-end flow.
- The four other read/write-picture docs — each names the specific queues/consumers a new
  feature in that story would need to plug into using the functions above.
