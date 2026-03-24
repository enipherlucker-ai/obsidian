#### 16.1. ETL — Extract, Transform, Load

**ETL** — классический подход к интеграции данных, при котором данные:

1. **Extract (Извлечение)** — данные извлекаются из различных источников:
   - Реляционные БД (PostgreSQL, MySQL, Oracle).
   - Файлы (CSV, JSON, XML, Parquet).
   - API (REST, SOAP, GraphQL).
   - Очереди сообщений (Kafka, RabbitMQ).
   - SaaS-системы (Salesforce, HubSpot, Google Analytics).
   - Логи, IoT-устройства.

2. **Transform (Трансформация)** — данные очищаются и трансформируются **до** загрузки в хранилище:
   - Очистка (удаление дубликатов, обработка NULL, нормализация форматов дат/валют).
   - Валидация (проверка типов, допустимых значений, ссылочной целостности).
   - Обогащение (join с справочниками, геокодирование).
   - Агрегация (предварительные вычисления, свёртка).
   - Приведение к целевой схеме (маппинг полей).
   - Трансформации выполняются на **промежуточном сервере** (ETL-сервер, staging area).
   - Инструменты: Informatica PowerCenter, Talend, Apache NiFi, Python-скрипты, Spark.

3. **Load (Загрузка)** — **очищенные и трансформированные** данные загружаются в целевое хранилище (Data Warehouse):
   - Загружаются уже структурированные, валидированные данные.
   - DWH хранит только «готовые» данные.

---

#### 16.2. ELT — Extract, Load, Transform

**ELT** — современный подход, инвертирующий порядок T и L:

1. **Extract (Извлечение)** — аналогично ETL.

2. **Load (Загрузка)** — **сырые** данные загружаются в хранилище **без трансформации** (или с минимальной, например, приведение к формату Parquet):
   - Загружается всё «как есть» — raw layer.
   - Инструменты: Fivetran, Airbyte, Stitch, AWS DMS.

3. **Transform (Трансформация)** — данные трансформируются **внутри** хранилища с использованием его вычислительных ресурсов:
   - SQL-трансформации (CREATE TABLE AS SELECT, INSERT INTO ... SELECT).
   - Инструменты: **dbt**, Dataform, хранимые процедуры, Spark SQL.
   - DWH должно быть достаточно мощным (Snowflake, BigQuery, Redshift, Databricks).

---

#### 16.3. Сравнительная таблица ETL vs ELT

| Критерий | ETL | ELT |
|---|---|---|
| **Порядок** | Extract → Transform → Load | Extract → Load → Transform |
| **Где трансформация** | На промежуточном ETL-сервере | Внутри целевого хранилища (DWH) |
| **Что попадает в DWH** | Очищенные, структурированные данные | Сырые данные (raw) + трансформированные слои |
| **Нагрузка на DWH** | Низкая (только хранение и запросы) | Высокая (хранение + вычисление трансформаций) |
| **Гибкость** | Низкая: изменение трансформации = переделка pipeline | Высокая: сырые данные сохранены, можно пересчитать |
| **Latency** | Выше (данные проходят через ETL-сервер) | Ниже (данные загружаются напрямую) |
| **Масштабируемость** | Ограничена мощностью ETL-сервера | Масштабируется вместе с DWH (cloud auto-scaling) |
| **Объём данных** | Лучше для ограниченных объёмов | Хорошо масштабируется для больших объёмов |
| **Инструменты** | Informatica, Talend, SSIS, Apache NiFi | dbt, Fivetran + dbt, Dataform, Matillion |
| **Хранение сырых данных** | Нет (только трансформированные) | Да (можно пересчитать историю) |
| **Compliance / Privacy** | Проще: чувствительные данные фильтруются ДО загрузки | Сложнее: сырые данные содержат всё, нужна маскировка внутри DWH |
| **Стоимость хранения** | Меньше (хранятся только готовые данные) | Больше (хранятся и сырые, и обработанные) |

---

#### 16.4. Когда использовать ETL

- **Ограниченные вычислительные ресурсы DWH** (on-premise, фиксированный размер кластера).
- **Строгие требования к качеству данных**: данные должны быть проверены ДО попадания в хранилище.
- **Регуляторные требования**: чувствительные данные (PII) должны быть отфильтрованы/маскированы до загрузки.
- **Сложные non-SQL трансформации**: машинное обучение, обработка изображений, NLP, которые нельзя выразить в SQL.
- **Legacy-системы**: традиционные DWH (Teradata, Oracle DW) без elastic scaling.
- **Небольшие объёмы данных**: когда полная ETL-цепочка не является bottleneck-ом.

---

#### 16.5. Когда использовать ELT

- **Мощное облачное DWH** с elastic scaling: Snowflake, Google BigQuery, Amazon Redshift, Databricks.
- **Большие объёмы данных**: когда ETL-сервер становится bottleneck-ом.
- **Необходимость гибкости**: бизнес-требования часто меняются, нужно пересчитывать историю.
- **Data Lake / Lakehouse**: сырые данные загружаются в озеро, затем трансформируются.
- **Self-service BI**: аналитики хотят сами создавать новые модели на основе сырых данных.
- **dbt-driven workflow**: команда использует dbt для управления трансформациями как кодом (SQL + Git + CI/CD + тесты + документация).

---

#### 16.6. Современный гибридный подход: EL + T

Наиболее распространённый паттерн в современных data-стеках:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Sources    │────▶│   EL Tool    │────▶│  Cloud DWH   │
│ (DB, API,    │     │ (Fivetran,   │     │ (Snowflake,  │
│  SaaS, files)│     │  Airbyte)    │     │  BigQuery)   │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                           ┌──────▼───────┐
                                           │     dbt      │
                                           │ (Transform)  │
                                           └──────┬───────┘
                                                  │
                                           ┌──────▼───────┐
                                           │   BI Tools   │
                                           │ (Looker,     │
                                           │  Tableau,    │
                                           │  Metabase)   │
                                           └──────────────┘
```

- **EL (Extract + Load)**: Инструмент (Fivetran, Airbyte, Meltano) извлекает данные из источников и загружает «как есть» в DWH (raw-слой).
- **T (Transform)**: dbt выполняет SQL-трансформации **внутри** DWH, создавая clean/mart-слои.

---

#### 16.7. Что такое dbt (data build tool)

**dbt** — open-source фреймворк для SQL-трансформаций в DWH. Подход «transformations as code».

Ключевые возможности:
- **Models (Модели)** — SQL-файлы (SELECT-запросы), которые dbt материализует как таблицы или views в DWH.
- **Materializations** — способ создания модели: `view`, `table`, `incremental`, `ephemeral`.
- **Tests** — встроенные тесты качества: `unique`, `not_null`, `accepted_values`, `relationships`, custom SQL-тесты.
- **Documentation** — автогенерация документации из YAML-описаний с lineage graph.
- **Lineage (Data Lineage)** — визуализация зависимостей между моделями (какая модель из какой строится).
- **Jinja + Macros** — шаблонизация SQL, переиспользуемые макросы.
- **Seeds** — CSV-файлы, загружаемые как таблицы (для справочников).
- **Snapshots** — SCD Type 2 (slowly changing dimensions).
- **Packages** — переиспользуемые библиотеки (dbt_utils, dbt_expectations).
- **CI/CD** — `dbt build` в CI-пайплайне: компиляция, выполнение, тесты.

Пример модели dbt:

```sql
-- models/marts/orders_summary.sql

{{ config(materialized='incremental', unique_key='order_date') }}

SELECT
    DATE(order_timestamp) AS order_date,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM {{ ref('stg_orders') }}    -- ссылка на staging-модель
WHERE status = 'completed'
{% if is_incremental() %}
    AND order_timestamp > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
GROUP BY 1
```

---

#### 16.8. Medallion Architecture (Медальонная архитектура)

Популярный паттерн организации данных в Data Lakehouse (Databricks), применим и к DWH:

| Слой | Описание | Аналогия |
|---|---|---|
| **Bronze (Raw)** | Сырые данные «как есть» из источников. Минимальная обработка: добавление метаданных (timestamp загрузки, источник). Все типы — строки. Хранит полную историю | Staging |
| **Silver (Cleaned)** | Очищенные, дедуплицированные данные с приведёнными типами. Валидированные, с join-ами справочников. Бизнес-сущности | ODS (Operational Data Store) |
| **Gold (Aggregated)** | Бизнес-агрегаты, готовые для потребления: витрины (marts), KPI, отчёты. Оптимизированы для BI-инструментов | Data Mart |

```
Bronze (raw)   →   Silver (clean)   →   Gold (mart)
───────────────────────────────────────────────────
stg_orders         int_orders            fct_daily_revenue
stg_customers      int_customers         dim_customers
stg_products       int_products          fct_product_sales
```

---

#### 16.9. Data Lake vs Data Warehouse vs Data Lakehouse

| Критерий | Data Lake | Data Warehouse | Data Lakehouse |
|---|---|---|---|
| **Хранение** | Файлы в объектном хранилище (S3, GCS, ADLS) | Структурированные данные в специализированной СУБД | Файлы с ACID-транзакциями (Delta Lake, Iceberg, Hudi) |
| **Формат** | Любой: CSV, JSON, Parquet, Avro, изображения, видео | Только структурированные (таблицы) | Parquet/ORC с метаданными (Delta, Iceberg) |
| **Schema** | Schema-on-read (схема при чтении) | Schema-on-write (схема при записи) | Schema-on-write + schema evolution |
| **Пользователи** | Data Engineers, Data Scientists | Business Analysts, BI | Все вышеперечисленные |
| **ACID** | Нет (без дополнительных инструментов) | Да | Да (Delta Lake, Iceberg) |
| **Стоимость** | Низкая (объектное хранилище) | Высокая (compute + storage) | Средняя (дешёвое хранилище + elastic compute) |
| **Примеры** | S3 + Athena, HDFS + Hive | Snowflake, BigQuery, Redshift | Databricks (Delta Lake), Dremio (Iceberg) |
| **Проблемы** | «Data Swamp» (болото данных), нет governance | Дорого, жёсткая схема | Относительно новая технология |

**Data Lakehouse** — попытка объединить лучшее из обоих миров: дешёвое хранение Data Lake (S3/GCS) + ACID-транзакции, schema enforcement и производительность Data Warehouse. Реализуется через open table formats: **Delta Lake** (Databricks), **Apache Iceberg** (Netflix, теперь широко принят), **Apache Hudi** (Uber).

Ключевые возможности Lakehouse:
- ACID-транзакции на объектном хранилище.
- Time travel (версионирование данных, откат к прошлым snapshot-ам).
- Schema evolution (безопасное изменение схемы).
- Unified batch + streaming.
- Прямой доступ из BI-инструментов (Tableau, Power BI).
