#### 15.1. Что такое Apache Kafka

**Apache Kafka** — распределённая платформа потоковой передачи событий (distributed event streaming platform), изначально разработанная в LinkedIn (2011) и переданная в Apache.

Kafka — это **персистентный распределённый лог** (commit log). Ключевые свойства:
- **Высокая пропускная способность**: миллионы сообщений в секунду.
- **Низкая задержка**: единицы миллисекунд.
- **Масштабируемость**: горизонтальная, добавлением брокеров.
- **Отказоустойчивость**: репликация данных между брокерами.
- **Персистентность**: сообщения хранятся на диске (а не только в памяти, как в RabbitMQ).
- **Replay**: консьюмеры могут перечитывать данные за любой прошлый период (в пределах retention).

Типичные сценарии:
- Потоковая передача данных между микросервисами (event-driven architecture).
- Сбор логов и метрик.
- CDC (Change Data Capture) — захват изменений из БД.
- Real-time аналитика.
- Очередь задач (альтернатива RabbitMQ для высоких нагрузок).

---

#### 15.2. Архитектура Kafka

```
┌────────────────────────────────────────────────────────────────┐
│                       Kafka Cluster                           │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐               │
│  │ Broker 0  │   │ Broker 1  │   │ Broker 2  │               │
│  │           │   │           │   │           │               │
│  │ Topic A   │   │ Topic A   │   │ Topic A   │               │
│  │  Part 0   │   │  Part 1   │   │  Part 2   │               │
│  │  (leader) │   │  (leader) │   │  (leader) │               │
│  │           │   │           │   │           │               │
│  │ Topic A   │   │ Topic A   │   │ Topic A   │               │
│  │  Part 1   │   │  Part 0   │   │  Part 0   │               │
│  │ (follower)│   │ (follower)│   │ (follower)│               │
│  └───────────┘   └───────────┘   └───────────┘               │
│                                                                │
│  ┌──────────────────────┐                                      │
│  │ ZooKeeper / KRaft    │  ← координация кластера              │
│  └──────────────────────┘                                      │
└────────────────────────────────────────────────────────────────┘
         ▲                              │
         │                              ▼
  ┌──────────────┐              ┌──────────────┐
  │  Producers   │              │  Consumers   │
  │ (отправка)   │              │ (чтение)     │
  └──────────────┘              └──────────────┘
```

**Broker** — один экземпляр (сервер) Kafka. Кластер состоит из нескольких брокеров. Каждый брокер хранит часть данных (партиции). Брокеры идентифицируются числовым `broker.id`.

**ZooKeeper** (до Kafka 3.3) — внешний сервис для координации кластера: выбор контроллера, хранение метаданных топиков, управление ACL.

**KRaft** (Kafka 3.3+, production-ready с 3.5) — встроенный механизм консенсуса на основе Raft, заменяющий ZooKeeper. Упрощает архитектуру (нет внешних зависимостей), ускоряет выбор контроллера, улучшает масштабируемость метаданных.

---

#### 15.3. Topics (Топики)

**Topic** — именованный поток (категория) сообщений. Аналогия: таблица в БД или каталог в файловой системе.

Каждый топик — это append-only лог: новые сообщения добавляются в конец, существующие не изменяются.

Ключевые настройки топика:

| Параметр | Описание | Пример |
|---|---|---|
| `retention.ms` | Время хранения сообщений (по умолчанию 7 дней = 604800000 мс) | `259200000` (3 дня) |
| `retention.bytes` | Макс. размер лога на партицию (по умолчанию -1 = без ограничения) | `1073741824` (1 GB) |
| `cleanup.policy` | Политика очистки: `delete` или `compact` (или оба) | `compact` |
| `num.partitions` | Количество партиций при создании (по умолчанию 1) | `12` |
| `replication.factor` | Количество реплик каждой партиции (по умолчанию 1) | `3` |
| `min.insync.replicas` | Минимум реплик в ISR для подтверждения записи | `2` |
| `segment.bytes` | Размер одного сегмента лога | `1073741824` |
| `max.message.bytes` | Макс. размер одного сообщения | `1048576` (1 MB) |

**Cleanup Policy — Delete vs Compact:**

- **`delete`** (по умолчанию): Старые сегменты удаляются по достижении `retention.ms` или `retention.bytes`. Подходит для потоков событий (логи, метрики, транзакции).

- **`compact`** (Log Compaction): Kafka сохраняет **только последнюю** запись для каждого ключа. Старые записи с тем же ключом удаляются при компакции. Подходит для хранения текущего состояния (changelog таблицы, конфигурации). Tombstone-запись (значение = `null`) удаляет ключ.

```
До компакции:           После компакции:
offset 0: key=A val=1   offset 2: key=A val=3
offset 1: key=B val=2   offset 1: key=B val=2
offset 2: key=A val=3   offset 3: key=C val=4
offset 3: key=C val=4
offset 4: key=B val=5   offset 4: key=B val=5
```

---

#### 15.4. Partitions (Партиции)

**Partition** — единица параллелизма и масштабирования в Kafka. Каждый топик состоит из одной или нескольких партиций.

Свойства партиции:
- **Упорядоченность**: Сообщения внутри одной партиции строго упорядочены по offset-у. Между разными партициями порядок **не гарантируется**.
- **Offset**: Уникальный монотонно возрастающий номер сообщения внутри партиции.
- **Параллелизм**: Каждая партиция может читаться **одним** консьюмером из группы. Больше партиций = больше параллелизм чтения.
- **Распределение по брокерам**: Партиции распределяются по брокерам кластера (round-robin при создании или вручную).

**Partition Key (Ключ маршрутизации):**

Продюсер может указать ключ сообщения. Kafka определяет партицию по формуле:

```
partition = hash(key) % num_partitions
```

Это гарантирует, что все сообщения с одним ключом попадут в одну партицию (и, следовательно, будут упорядочены). Например, все события пользователя `user_123` — в одной партиции.

Если ключ не указан — используется round-robin (или sticky partitioning в новых версиях для батчирования).

**Репликация:**

- **Replication Factor** — количество копий каждой партиции (обычно 3).
- **Leader** — одна реплика, через которую происходит чтение и запись.
- **Follower** — остальные реплики, которые синхронно (или асинхронно) копируют данные с лидера.
- **ISR (In-Sync Replicas)** — множество реплик, которые не отстали от лидера более чем на `replica.lag.time.max.ms`. При записи с `acks=all` Kafka ждёт подтверждения от всех ISR.
- При падении лидера — один из follower-ов в ISR становится новым лидером (автоматически).

---

#### 15.5. Producer (Продюсер)

Продюсер — клиент, отправляющий сообщения в Kafka.

**Ключевые параметры продюсера:**

| Параметр | Описание |
|---|---|
| `acks=0` | Продюсер не ждёт подтверждения от брокера. Максимальная скорость, но возможна потеря данных |
| `acks=1` | Продюсер ждёт подтверждения от **лидера** партиции. Компромисс: данные потеряются, если лидер упадёт до репликации |
| `acks=all` (`-1`) | Продюсер ждёт подтверждения от **всех ISR**. Максимальная надёжность, но выше latency |
| `linger.ms` | Задержка перед отправкой (для формирования батча). По умолчанию 0 (отправка немедленно). Увеличение → больше батч → выше throughput, но выше latency |
| `batch.size` | Максимальный размер батча (байт). По умолчанию 16 KB |
| `compression.type` | Сжатие: `none`, `gzip`, `snappy`, `lz4`, `zstd`. Снижает нагрузку на сеть и диск |
| `buffer.memory` | Размер буфера продюсера (по умолчанию 32 MB). Если буфер полон, `send()` блокируется |
| `retries` | Количество повторных попыток при ошибке (по умолчанию `MAX_INT` в новых версиях) |
| `max.in.flight.requests.per.connection` | Макс. количество неподтверждённых запросов. При >1 возможна переупорядочивание (если не идемпотентный продюсер) |

**Idempotent Producer** (`enable.idempotence=true`):
- Гарантирует exactly-once семантику на уровне отдельной партиции.
- Каждое сообщение получает sequence number; брокер отклоняет дубликаты.
- Автоматически устанавливает `acks=all`, `max.in.flight.requests.per.connection=5`, `retries=MAX_INT`.

**Exactly-Once Semantics (Транзакции):**
- Transactional producer может атомарно записывать в несколько топиков/партиций.
- `initTransactions()` → `beginTransaction()` → `send()` → `commitTransaction()`.
- Используется в Kafka Streams для exactly-once обработки.

---

#### 15.6. Consumer (Консьюмер)

Консьюмер — клиент, читающий сообщения из Kafka.

**Consumer Group (Группа консьюмеров):**
- Каждый консьюмер входит в **группу** (`group.id`).
- Kafka **распределяет партиции** между консьюмерами одной группы: каждая партиция назначается ровно одному консьюмеру.
- Если консьюмеров больше, чем партиций — лишние будут простаивать.
- Разные группы получают **все** сообщения независимо (pub/sub модель).

```
Topic: orders (4 партиции)

Consumer Group A:             Consumer Group B:
┌──────────┐ Part 0,1        ┌──────────┐ Part 0,1,2,3
│Consumer 1│◄────────         │Consumer 1│◄────────
└──────────┘                  └──────────┘
┌──────────┐ Part 2,3
│Consumer 2│◄────────
└──────────┘
```

**Partition Assignment Strategies:**

| Стратегия | Описание |
|---|---|
| `RangeAssignor` | Партиции делятся «по диапазону» на каждый топик отдельно. По умолчанию |
| `RoundRobinAssignor` | Партиции всех топиков назначаются по кругу |
| `StickyAssignor` | Минимизирует перемещение партиций при ребалансировке |
| `CooperativeStickyAssignor` | Инкрементальная ребалансировка — не забирает все партиции сразу |

**Offset Management (Управление смещением):**

Offset — позиция последнего прочитанного сообщения в партиции.

- **Автоматический коммит** (`enable.auto.commit=true`): Kafka периодически (каждые `auto.commit.interval.ms`) коммитит текущий offset. Простота, но возможна потеря или дублирование (at-least-once).
- **Ручной коммит** (`enable.auto.commit=false`): Консьюмер явно вызывает `commitSync()` или `commitAsync()` после обработки сообщений. Больше контроля.

**Семантики доставки:**

| Семантика | Описание | Как достигается |
|---|---|---|
| **At-most-once** | Сообщение обрабатывается 0 или 1 раз. Потеря возможна | Коммит offset ДО обработки |
| **At-least-once** | Сообщение обрабатывается 1 или более раз. Дубликаты возможны | Коммит offset ПОСЛЕ обработки. Обработка должна быть идемпотентной |
| **Exactly-once** | Сообщение обрабатывается ровно 1 раз | Kafka transactions + idempotent producer + `isolation.level=read_committed` |

**Rebalancing (Ребалансировка):**

Происходит при:
- Добавлении/удалении консьюмера в группу.
- Падении консьюмера (нет heartbeat в течение `session.timeout.ms`).
- Добавлении новых партиций к топику.

Во время ребалансировки **все** консьюмеры группы прекращают чтение (при eager rebalancing). CooperativeStickyAssignor решает эту проблему, делая ребалансировку инкрементальной.

---

#### 15.7. Примеры кода с kafka-python

**Producer:**

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['broker1:9092', 'broker2:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    acks='all',
    retries=5,
    linger_ms=10,
    batch_size=32768,
    compression_type='gzip',
    enable_idempotence=True,
)

# Отправка сообщения
future = producer.send(
    topic='orders',
    key='user_123',
    value={'order_id': 456, 'amount': 99.90, 'product': 'Widget'},
    headers=[('source', b'web')],
)

# Синхронное ожидание подтверждения
record_metadata = future.get(timeout=10)
print(f"Topic: {record_metadata.topic}, "
      f"Partition: {record_metadata.partition}, "
      f"Offset: {record_metadata.offset}")

producer.flush()
producer.close()
```

**Consumer:**

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['broker1:9092', 'broker2:9092'],
    group_id='order_processing_group',
    value_deserializer=lambda v: json.loads(v.decode('utf-8')),
    key_deserializer=lambda k: k.decode('utf-8') if k else None,
    auto_offset_reset='earliest',  # с начала, если нет сохранённого offset
    enable_auto_commit=False,      # ручной коммит для at-least-once
    max_poll_records=500,
    session_timeout_ms=30000,
)

try:
    for message in consumer:
        print(f"Partition: {message.partition}, "
              f"Offset: {message.offset}, "
              f"Key: {message.key}, "
              f"Value: {message.value}")

        process_order(message.value)

        # Ручной коммит после успешной обработки
        consumer.commit()
except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

---

#### 15.8. Kafka Connect и Kafka Streams

**Kafka Connect** — фреймворк для интеграции Kafka с внешними системами **без написания кода** (декларативно, через JSON-конфигурацию).

- **Source Connector** — читает данные из внешней системы и записывает в Kafka (напр., Debezium для CDC из PostgreSQL/MySQL).
- **Sink Connector** — читает данные из Kafka и записывает во внешнюю систему (напр., Elasticsearch Sink, S3 Sink, JDBC Sink).
- Connector-ы запускаются внутри Connect Worker-ов (standalone или distributed mode).
- Поддержка exactly-once через SMT (Single Message Transforms).

**Kafka Streams** — клиентская Java/Scala-библиотека для **потоковой обработки** данных из Kafka в Kafka. Не требует отдельного кластера (как Flink/Spark Streaming). Поддерживает: фильтрацию, маппинг, агрегацию, join-ы (KStream-KTable, KStream-KStream), оконные операции (tumbling, hopping, sliding, session windows), stateful processing с локальным хранилищем (RocksDB).
