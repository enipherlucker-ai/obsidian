### 5.1. Что такое ORM

**ORM (Object-Relational Mapping)** — технология, которая связывает реляционную базу данных с объектной моделью языка программирования, позволяя работать с записями в БД как с обычными объектами, а не писать «сырой» SQL.

Ключевое соответствие:

| Реляционная модель | Объектная модель |
|--------------------|------------------|
| Таблица (table) | Класс (class) |
| Строка (row) | Экземпляр объекта (instance) |
| Столбец (column) | Атрибут объекта (attribute) |
| Внешний ключ (FK) | Ссылка на связанный объект / коллекция |

Благодаря этому маппингу разработчик оперирует понятиями предметной области (`User`, `Order`), а ORM генерирует SQL под капотом.

### 5.2. Основные паттерны ORM

#### 5.2.1. Active Record

Объект **сам знает**, как себя сохранить, обновить и удалить. Класс модели содержит и бизнес-логику, и логику доступа к данным.

```python
# Пример: Django ORM (Active Record-подобный)
class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

p = Product(name="Ноутбук", price=59999)
p.save()          # INSERT — объект сам себя сохраняет
p.price = 54999
p.save()          # UPDATE
p.delete()        # DELETE
```

**Плюсы:** простота, быстрая разработка, минимум boilerplate.
**Минусы:** модель «знает» о базе → трудно тестировать в изоляции, бизнес-логика смешивается с инфраструктурным кодом.

#### 5.2.2. Data Mapper

Объект **ничего не знает** о базе данных. Отдельный слой (mapper/repository) отвечает за перенос данных между объектами и таблицами.

```python
# Пример: SQLAlchemy (Data Mapper)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Numeric

class Base(DeclarativeBase):
    pass

class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    price: Mapped[float] = mapped_column(Numeric(10, 2))

# Объект не имеет метода save() — за персистентность отвечает Session
from sqlalchemy.orm import Session

with Session(engine) as session:
    p = Product(name="Ноутбук", price=59999)
    session.add(p)        # запланировать INSERT
    session.commit()      # выполнить
```

**Плюсы:** чистое разделение ответственности, модели легко тестировать, гибкость маппинга.
**Минусы:** больше кода, выше порог входа.

| Критерий | Active Record | Data Mapper |
|----------|--------------|-------------|
| Простота | Высокая | Средняя |
| Разделение ответственности | Низкое | Высокое |
| Тестируемость моделей | Труднее (зависимость от БД) | Легче (POPO) |
| Примеры | Django ORM, Ruby on Rails AR | SQLAlchemy, Hibernate (Java) |
| Когда выбирать | CRUD-приложения, прототипы | Сложная доменная логика |

### 5.3. Unit of Work и Identity Map

**Unit of Work** — паттерн, при котором ORM отслеживает все изменения объектов (новые, изменённые, удалённые) и выполняет их **одной транзакцией** при вызове `commit()`. В SQLAlchemy эту роль играет объект `Session`.

```python
with Session(engine) as session:
    u = session.get(User, 1)
    u.name = "Иван"               # отслеживается автоматически
    session.add(User(name="Олег")) # новый объект
    session.commit()               # один INSERT + один UPDATE в одной транзакции
```

**Identity Map** — кэш внутри Unit of Work, гарантирующий, что для одной и той же строки БД в рамках одной сессии существует ровно один Python-объект. Это предотвращает противоречия:

```python
a = session.get(User, 1)
b = session.get(User, 1)
assert a is b  # True — один и тот же объект, второй запрос к БД не выполняется
```

### 5.4. Загрузка связанных данных: Lazy vs Eager Loading

| Стратегия | Как работает | SQL | Когда использовать |
|-----------|-------------|-----|--------------------|
| **Lazy Loading** | Связанные объекты загружаются при первом обращении к атрибуту | Отдельный `SELECT` по требованию | Связь нужна редко |
| **Eager Loading** | Связанные объекты загружаются сразу (JOIN / subquery) | `JOIN` или дополнительный `SELECT` при основном запросе | Связь нужна почти всегда |

```python
# SQLAlchemy: Eager loading
from sqlalchemy.orm import joinedload

users = session.execute(
    select(User).options(joinedload(User.orders))
).scalars().unique().all()
# Один запрос с JOIN — все заказы загружены сразу
```

### 5.5. Проблема N+1 запросов

Одна из самых частых проблем производительности с ORM.

**Суть:** при Lazy Loading итерация по N объектам, у каждого из которых есть связь, порождает **1** запрос на основную выборку + **N** запросов на загрузку связей.

```python
# N+1 проблема
users = session.execute(select(User)).scalars().all()  # 1 запрос
for user in users:
    print(user.orders)  # N запросов (по одному на каждого пользователя!)
```

**Решения:**
1. `joinedload` / `subqueryload` / `selectinload` — загрузить связь заранее.
2. В Django: `select_related` (JOIN) и `prefetch_related` (отдельный запрос + склейка в Python).

```python
# Django: решение N+1
users = User.objects.select_related("profile").prefetch_related("orders").all()
```

### 5.6. SQLAlchemy: ORM Layer vs Core Layer

SQLAlchemy состоит из **двух уровней**:

| Уровень | Что даёт | Абстракция |
|---------|----------|------------|
| **Core** | SQL Expression Language — программная генерация SQL без маппинга на объекты | Таблицы, столбцы, выражения |
| **ORM** | Маппинг на классы, Session, Unit of Work, Identity Map, связи | Классы-модели, запросы через `select(Model)` |

```python
# ——— Core ———
from sqlalchemy import Table, Column, Integer, String, MetaData, select, create_engine

metadata = MetaData()
users = Table("users", metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(100)),
)
engine = create_engine("sqlite:///app.db")

with engine.connect() as conn:
    result = conn.execute(select(users).where(users.c.name == "Alice"))
    for row in result:
        print(row.name)   # row — это NamedTuple-подобный объект, а не модель

# ——— ORM ———
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

with Session(engine) as session:
    user = session.execute(select(User).where(User.name == "Alice")).scalar_one()
    print(user.name)   # user — экземпляр класса User
```

Core используется, когда нужен полный контроль над SQL без overhead маппинга (ETL-пайплайны, миграции, bulk-операции).

### 5.7. Когда ORM **НЕ** подходит

| Сценарий | Почему ORM плох | Альтернатива |
|----------|----------------|--------------|
| Тяжёлая аналитика (агрегации, оконные функции) | ORM генерирует неоптимальный SQL, сложно выразить сложные запросы | Сырой SQL / SQLAlchemy Core / специализированные OLAP-движки |
| Массовая вставка (bulk insert миллионов строк) | Создание объекта на каждую строку — overhead по памяти и CPU | `COPY` (PostgreSQL), `executemany`, `bulk_save_objects`, pandas `to_sql` |
| Запросы с множественными JOIN и подзапросами | ORM абстракция «протекает», приходится бороться с генератором SQL | Прямой SQL, materialized views |
| Высоконагруженные микросервисы с простым доступом к данным | Overhead ORM не оправдан для `SELECT * WHERE id = ?` | Лёгкие query-builders, `asyncpg` напрямую |

### 5.8. Резюме

ORM экономит время на CRUD-операциях, обеспечивает безопасность типов и переносимость между СУБД. Но разработчик обязан понимать генерируемый SQL (`.statement` в SQLAlchemy, `queryset.query` в Django), контролировать стратегии загрузки и вовремя переключаться на «сырой» SQL там, где абстракция мешает производительности.