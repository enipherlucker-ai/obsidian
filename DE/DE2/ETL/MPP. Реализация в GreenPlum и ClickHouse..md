### 17.1. Что такое MPP (Massively Parallel Processing)

**MPP (Massively Parallel Processing)** — архитектура баз данных, в которой запрос разбивается на подзадачи и выполняется параллельно на множестве независимых узлов (nodes). Каждый узел имеет собственный процессор, оперативную память и диск — это называется **shared-nothing** архитектура.

Ключевые характеристики MPP:
- **Горизонтальное масштабирование (scale-out):** добавление новых узлов линейно увеличивает производительность.
- **Shared-nothing:** каждый узел владеет своей частью данных и ни с кем не делит ресурсы.
- **Координатор (Master):** принимает запрос, строит план, распределяет подзадачи по узлам.
- **Data Distribution:** данные заранее распределены по узлам по определённому ключу (sharding key / distribution key).
- **Data Shuffling:** при JOIN или GROUP BY по ключу, не совпадающему с ключом распределения, данные пересылаются между узлами (redistribute / reshuffle).

### 17.2. Shared-nothing vs Shared-disk vs Shared-everything

| Характеристика | Shared-Everything | Shared-Disk | Shared-Nothing |
|---|---|---|---|
| **CPU** | Общие | Свои у каждого узла | Свои у каждого узла |
| **Память** | Общая | Своя у каждого узла | Своя у каждого узла |
| **Диск** | Общий | Общий (SAN/NAS) | Свой у каждого узла |
| **Масштабирование** | Ограничено (SMP) | Среднее | Лучшее (линейное) |
| **Примеры** | SMP-серверы | Oracle RAC, Exadata | GreenPlum, ClickHouse, Redshift |
| **Bottleneck** | Шина памяти | Сетевой доступ к диску | Сеть (при shuffle) |
| **Сложность** | Низкая | Средняя | Высокая (распределение данных) |

### 17.3. Реализация MPP в GreenPlum

GreenPlum — это MPP-СУБД, построенная поверх **PostgreSQL**. Каждый узел кластера — это модифицированный PostgreSQL-инстанс.

#### 17.3.1. Архитектура GreenPlum

```
┌─────────────────────────────────────────────────────────┐
│                     КЛИЕНТ (psql, JDBC)                 │
│                           │                             │
│                    ┌──────▼──────┐                      │
│                    │   MASTER    │  ← Координатор       │
│                    │  (Catalog,  │    (парсинг, план,   │
│                    │   Planner)  │     диспетчеризация) │
│                    └──────┬──────┘                      │
│              ┌────────────┼────────────┐                │
│              │     Interconnect        │  ← Сеть        │
│       ┌──────▼──────┐          ┌──────▼──────┐         │
│       │ SEGMENT 1   │          │ SEGMENT 2   │         │
│       │ (PostgreSQL │   ...    │ (PostgreSQL │         │
│       │  instance)  │          │  instance)  │         │
│       └─────────────┘          └─────────────┘         │
│                                                         │
│       Каждый сегмент хранит свою часть данных           │
└─────────────────────────────────────────────────────────┘
```

- **Master Node** — точка входа. Не хранит пользовательских данных (только системный каталог). Принимает SQL-запросы, строит план выполнения, отправляет подзадачи на сегменты, собирает результаты.
- **Segment Nodes** — рабочие узлы. Каждый сегмент — это отдельный PostgreSQL-инстанс со своей частью данных. На одном физическом сервере может быть несколько сегментов (обычно по количеству CPU-ядер).
- **Standby Master** — резервный мастер для отказоустойчивости.
- **Segment Mirrors** — зеркала сегментов (реплики) для восстановления при сбое.

#### 17.3.2. Распределение данных (DISTRIBUTED BY)

При создании таблицы обязательно указывается стратегия распределения:

```sql
-- 1) Hash Distribution — строки распределяются по хэшу ключа
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id BIGINT,
    order_date  DATE,
    amount      NUMERIC(15,2)
) DISTRIBUTED BY (customer_id);
-- Все заказы одного клиента попадут на один сегмент.
-- Это позволяет делать JOIN по customer_id без пересылки данных.

-- 2) Random Distribution — строки равномерно, но случайно разбрасываются
CREATE TABLE logs (
    ts      TIMESTAMP,
    message TEXT
) DISTRIBUTED RANDOMLY;
-- Гарантирует равномерность, но любой JOIN потребует redistribute.

-- 3) Replicated Distribution — полная копия таблицы на каждом сегменте
CREATE TABLE countries (
    country_code CHAR(2),
    country_name TEXT
) DISTRIBUTED REPLICATED;
-- Идеально для маленьких справочников — JOIN без пересылки.
```

**Правила выбора ключа распределения:**
1. Столбец с высокой кардинальностью (много уникальных значений) — чтобы данные распределились равномерно.
2. Столбец, по которому часто делается JOIN — чтобы избежать redistribute.
3. Избегать распределения по столбцам с перекосом (data skew) — например, по `status`, где 90% записей имеют значение `'active'`.

#### 17.3.3. Interconnect и Motion-операторы

**Interconnect** — сетевой уровень GreenPlum, отвечающий за пересылку данных между сегментами. Использует UDP (с собственным контролем доставки) для высокой производительности.

**Motion-операторы** — операторы в плане запроса, отвечающие за перемещение данных:

| Motion-оператор | Описание | Когда используется |
|---|---|---|
| **Redistribute Motion** | Перехеширование данных по новому ключу | JOIN по столбцу ≠ distribution key |
| **Broadcast Motion** | Отправка полной копии таблицы на все сегменты | JOIN маленькой таблицы с большой |
| **Gather Motion** | Сбор данных со всех сегментов на Master | Финальный результат, ORDER BY |

Пример плана с Motion:

```sql
EXPLAIN SELECT o.order_id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- Если orders DISTRIBUTED BY (customer_id) и customers DISTRIBUTED BY (customer_id):
--   → Hash Join (локальный, без Motion!)
-- Если customers DISTRIBUTED BY (customer_id), а orders DISTRIBUTED BY (order_id):
--   → Redistribute Motion (orders по customer_id) → Hash Join
-- Если customers — маленькая таблица:
--   → Broadcast Motion (customers) → Hash Join
```

#### 17.3.4. Slices и параллелизм запросов

**Slice** — это единица параллелизма в плане запроса GreenPlum. Каждый Motion-оператор разделяет план на слайсы. Каждый слайс выполняется независимо на всех сегментах.

```
Slice 0: Gather Motion (на Master)
  └── Slice 1: Hash Join
        ├── Seq Scan on orders (локально на каждом сегменте)
        └── Slice 2: Redistribute Motion
              └── Seq Scan on customers (локально)
```

Каждый сегмент запускает `gang` — набор процессов (по одному на слайс). Таким образом, если у вас 8 сегментов и 3 слайса в запросе, одновременно работают 24 процесса.

### 17.4. Реализация MPP в ClickHouse

ClickHouse — это **колоночная OLAP СУБД**, ориентированная на аналитические запросы. Изначально разработана в Яндексе для аналитики Яндекс.Метрики.

#### 17.4.1. Архитектура ClickHouse-кластера

ClickHouse не имеет выделенного мастер-узла. Каждый узел — полноценный сервер ClickHouse. Кластер описывается в конфигурации:

```xml
<clickhouse>
    <remote_servers>
        <my_cluster>
            <shard>
                <replica>
                    <host>ch-node-1</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-node-2</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>ch-node-3</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-node-4</host>
                    <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>
</clickhouse>
```

Здесь 2 шарда, каждый с 2 репликами.

#### 17.4.2. Distributed-таблица и шардирование

```sql
-- Локальная таблица на каждом узле
CREATE TABLE events_local ON CLUSTER my_cluster (
    event_date  Date,
    user_id     UInt64,
    event_type  String,
    payload     String
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (user_id, event_date);

-- Распределённая таблица — «виртуальная» обёртка
CREATE TABLE events_distributed ON CLUSTER my_cluster AS events_local
ENGINE = Distributed(my_cluster, default, events_local, sipHash64(user_id));
-- sipHash64(user_id) — sharding key, определяет, на какой шард попадёт строка
```

**Как работает INSERT в Distributed-таблицу:**
1. Клиент отправляет INSERT в `events_distributed` на любой узел.
2. Этот узел по sharding key вычисляет целевой шард для каждой строки.
3. Строки пересылаются на нужные шарды и вставляются в `events_local`.
4. Альтернатива: INSERT напрямую в `events_local` (рекомендуется для production — контроль над балансировкой).

**Как работает SELECT из Distributed-таблицы:**
1. Узел-координатор (тот, куда пришёл запрос) отправляет запрос на все шарды.
2. Каждый шард выполняет запрос на своей `events_local`.
3. Координатор получает частичные результаты и объединяет их (merge агрегатов, сортировка, LIMIT).

#### 17.4.3. Локальный параллелизм

В отличие от GreenPlum, где параллелизм достигается за счёт множества сегментов (процессов), ClickHouse использует **многопоточность внутри одного узла**:

- Настройка `max_threads` (по умолчанию = количество CPU-ядер) определяет, сколько потоков параллельно обрабатывают один запрос.
- Данные из таблицы разбиваются на **гранулы** (по умолчанию 8192 строки), и потоки обрабатывают гранулы независимо.
- Это даёт линейное ускорение на одном узле при увеличении ядер.

### 17.5. Сравнение GreenPlum и ClickHouse

| Критерий | GreenPlum | ClickHouse |
|---|---|---|
| **Базовая СУБД** | PostgreSQL | Собственная разработка (Яндекс) |
| **Хранение** | Строковое (row-oriented) + append-optimized columnar | Колоночное (column-oriented) |
| **Координатор** | Выделенный Master-узел | Любой узел может быть координатором |
| **Параллелизм** | Много процессов (сегментов) | Многопоточность внутри узла |
| **Распределение данных** | DISTRIBUTED BY (hash/random/replicated) | Distributed engine + sharding key |
| **SQL-совместимость** | Полная PostgreSQL (JOIN, подзапросы, CTE, оконные) | Своё расширение SQL, есть ограничения JOIN |
| **Лучше для** | Сложные JOIN, ad-hoc аналитика, DWH | Агрегации над большими объёмами, timeseries, логи |
| **UPDATE/DELETE** | Полноценная поддержка | Мутации (асинхронные, тяжёлые) |
| **Транзакции** | ACID (от PostgreSQL) | Нет полноценных транзакций |
| **Сжатие** | Стандартное (zlib, zstd для AO) | Агрессивное колоночное (LZ4, ZSTD, Delta, DoubleDelta, Gorilla) |
| **Скорость на простых агрегатах** | Быстро, но уступает ClickHouse | Сверхбыстро (колоночная + SIMD + векторизация) |
