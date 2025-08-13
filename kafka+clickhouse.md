Perfect! Let’s explain the **full project end-to-end** using **Confluent Kafka + ClickHouse**, including **how data is produced, topics assigned, how it is consumed**, and **how to retrieve it**, with **settings.py configuration** for all Kafka and ClickHouse parameters. I’ll cover both **direct insertion** and **Materialized View** approaches.

---

# **PART 1: Direct ClickHouse Insertion**

## **1️⃣ Configuration in Django (`settings.py`)**

```python
# settings.py

KAFKA_SETTINGS = {
    "bootstrap_servers": "localhost:9092",
    "client_id": "django-campaign-producer",
    "group_id": "clickhouse-writer",
    "topics": {
        "event_tracking": "campaign.event_tracking",
        "audience_ready": "campaign.audience_ready",
        "template_ready": "campaign.template_ready"
    }
}

CLICKHOUSE_SETTINGS = {
    "host": "localhost",
    "username": "default",
    "password": "",
    "database": "campaigns"
}
```

* **Centralized configs** make producers/consumers reusable.

---

## **2️⃣ Step 1: Kafka Producer (Django)**

```python
# campaign/kafka_producer.py
import json
from confluent_kafka import Producer
from django.conf import settings

producer = Producer({
    'bootstrap.servers': settings.KAFKA_SETTINGS['bootstrap_servers'],
    'client.id': settings.KAFKA_SETTINGS['client_id']
})

def publish_event(topic_key: str, data: dict):
    topic = settings.KAFKA_SETTINGS['topics'][topic_key]
    producer.produce(topic=topic, value=json.dumps(data))
    producer.flush()
```

**Usage in Django campaign logic:**

```python
from campaign.kafka_producer import publish_event
from datetime import datetime

event = {
    "campaign_id": "123e4567-e89b-12d3-a456-426614174000",
    "workspace_id": "987e6543-e21b-32d1-a654-426614174999",
    "email": "user@example.com",
    "event_type": "delivered",
    "event_time": datetime.utcnow().isoformat(),
    "metadata": {"subject": "Welcome Email"}
}

publish_event("event_tracking", event)
```

* Each **type of event** (audience ready, template ready, email sent) is assigned a **dedicated topic**.
* Events are serialized as JSON.

---

## **3️⃣ Step 2: Kafka Topic Assignment**

| Event Type     | Kafka Topic               |
| -------------- | ------------------------- |
| Audience ready | `campaign.audience_ready` |
| Template ready | `campaign.template_ready` |
| Email sent     | `campaign.event_tracking` |

* Each topic allows **parallel consumers** and **replayable events**.

---

## **4️⃣ Step 3: ClickHouse Table (Direct Insertion)**

```sql
CREATE TABLE email_events
(
    campaign_id UUID,
    workspace_id UUID,
    email String,
    event_type Enum8('delivered'=1,'open'=2,'click'=3,'bounce'=4,'unsubscribe'=5),
    event_time DateTime,
    metadata String DEFAULT ''
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (campaign_id, event_time);
```

* Partitioned by day, ordered by campaign\_id + event\_time.
* Optimized for **high-volume inserts and queries**.

---

## **5️⃣ Step 4: Kafka Consumer → ClickHouse**

```python
# campaign/clickhouse_consumer.py
import json
from confluent_kafka import Consumer
from clickhouse_connect import Client
from django.conf import settings

ch = Client(
    host=settings.CLICKHOUSE_SETTINGS['host'],
    username=settings.CLICKHOUSE_SETTINGS['username'],
    password=settings.CLICKHOUSE_SETTINGS['password'],
    database=settings.CLICKHOUSE_SETTINGS['database']
)

consumer = Consumer({
    'bootstrap.servers': settings.KAFKA_SETTINGS['bootstrap_servers'],
    'group.id': settings.KAFKA_SETTINGS['group_id'],
    'auto.offset.reset': 'earliest'
})

consumer.subscribe([settings.KAFKA_SETTINGS['topics']['event_tracking']])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        print("Consumer error:", msg.error())
        continue

    event = json.loads(msg.value().decode('utf-8'))

    # Direct insertion into ClickHouse
    ch.insert('email_events', [{
        'campaign_id': event['campaign_id'],
        'workspace_id': event['workspace_id'],
        'email': event['email'],
        'event_type': event['event_type'],
        'event_time': event['event_time'],
        'metadata': json.dumps(event.get('metadata', {}))
    }])
```

---

## **6️⃣ Step 5: Data Retrieval**

```sql
-- Retrieve all opens for a campaign
SELECT email, event_time
FROM email_events
WHERE campaign_id = '123e4567-e89b-12d3-a456-426614174000'
  AND event_type = 'open'
ORDER BY event_time DESC;
```

* Flexible query on **raw events**.
* Good for **ad-hoc analytics** or detailed logs.

---

# **PART 2: Materialized View Using Kafka Table**

Now, the consumer is **ClickHouse itself**, no Python required for insertion.

---

## **1️⃣ Step 1: Kafka Table in ClickHouse**

```sql
CREATE TABLE queue
(
    timestamp UInt64,
    level String,
    message String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'localhost:9092',
    kafka_topic_list = 'campaign.event_tracking',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow';
```

* `queue` reads **events directly from Kafka**.
* ClickHouse automatically polls the topic.

---

## **2️⃣ Step 2: Aggregated Table**

```sql
CREATE TABLE daily
(
    day Date,
    level String,
    total UInt64
)
ENGINE = SummingMergeTree(day, (day, level), 8192);
```

* Stores **aggregated counts per day and event type**.

---

## **3️⃣ Step 3: Materialized View**

```sql
CREATE MATERIALIZED VIEW consumer
TO daily
AS
SELECT
    toDate(toDateTime(timestamp)) AS day,
    level,
    count() AS total
FROM queue
GROUP BY day, level;
```

* Automatically inserts aggregated counts into `daily` table.
* Near **real-time analytics**, no Python consumer needed.

---

## **4️⃣ Step 4: Data Retrieval**

```sql
-- Aggregated metrics per level (event type)
SELECT level, sum(total) AS total_count
FROM daily
GROUP BY level;
```

* Returns **fast, dashboard-ready aggregates**.
* For raw events, query `queue` table directly.

---

## **5️⃣ Step 5: How the Flow Works End-to-End**

1. **Django Producer** sends JSON events to `campaign.event_tracking`.
2. **Kafka Topic** persists events.
3. **ClickHouse Kafka Table (`queue`)** reads messages automatically.
4. **Materialized View (`consumer`) → `daily`** aggregates counts per day + event type.
5. **Retrieve aggregated metrics** with a simple query.

* No Python consumer is needed for insertion.
* Very **low-latency analytics**, perfect for dashboards.

---

# ✅ **Summary Table: Direct vs MV Approach**

| Feature           | Direct Insertion          | Kafka Table + Materialized View      |
| ----------------- | ------------------------- | ------------------------------------ |
| Consumer          | Python                    | Built-in ClickHouse                  |
| Aggregation       | Manual SQL queries        | Automatic via MV                     |
| Dashboard-ready   | Needs aggregation queries | Ready out-of-the-box                 |
| Raw events access | `email_events` table      | `queue` table                        |
| Complexity        | Moderate                  | Simple once Kafka + ClickHouse setup |

---

If you want, I can **draw a single end-to-end diagram** showing:

* **Django → Kafka → Consumer → ClickHouse (direct vs MV)**
* **Topics, tables, and retrieval paths**

This will make it extremely clear for both developers and stakeholders.

Do you want me to create that diagram next?
