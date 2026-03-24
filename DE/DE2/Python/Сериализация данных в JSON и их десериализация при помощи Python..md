### 11.1. Что такое сериализация и десериализация

- **Сериализация** — преобразование структуры данных (объекта) в последовательность байтов или текст для хранения или передачи.
- **Десериализация** — обратный процесс: восстановление структуры данных из сериализованного представления.

### 11.2. Формат JSON

**JSON** (JavaScript Object Notation) — текстовый формат обмена данными, легко читаемый человеком и машиной.

**Поддерживаемые типы:**
- `object` (словарь) `{"key": "value"}`
- `array` (массив) `[1, 2, 3]`
- `string` `"hello"` (всегда двойные кавычки)
- `number` `42`, `3.14` (целые и дробные)
- `boolean` `true`, `false`
- `null`

**Ограничения JSON:**
- Ключи — только строки.
- Нет типов: date/datetime, Decimal, bytes, set, tuple (сериализуется как array).
- Нет комментариев.
- Нет trailing commas.
- Числа: нет `Infinity`, `NaN` (формально).

### 11.3. Модуль `json` — основные функции

| Функция | Направление | Вход → Выход |
|---------|-------------|--------------|
| `json.dumps(obj)` | Сериализация | Python-объект → JSON-строка |
| `json.dump(obj, fp)` | Сериализация | Python-объект → файл |
| `json.loads(s)` | Десериализация | JSON-строка → Python-объект |
| `json.load(fp)` | Десериализация | Файл → Python-объект |

```python
import json

data = {"name": "Иван", "age": 30, "scores": [95, 87, 92]}

# Сериализация в строку
json_str = json.dumps(data, ensure_ascii=False, indent=2)
print(json_str)

# Десериализация из строки
restored = json.loads(json_str)
print(restored["name"])  # Иван

# Запись в файл
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# Чтение из файла
with open("data.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
```

### 11.4. Таблица соответствия типов Python ↔ JSON

| Python | JSON | Примечания |
|--------|------|------------|
| `dict` | object | Ключи приводятся к строкам |
| `list`, `tuple` | array | tuple теряет тип |
| `str` | string | — |
| `int`, `float` | number | — |
| `True` / `False` | true / false | — |
| `None` | null | — |

### 11.5. Важные параметры `json.dumps()`

| Параметр | Описание | Пример |
|----------|----------|--------|
| `indent` | Отступ для pretty-print | `indent=2` |
| `ensure_ascii` | Экранировать не-ASCII символы | `ensure_ascii=False` для кириллицы |
| `sort_keys` | Сортировать ключи | `sort_keys=True` |
| `default` | Функция для несериализуемых типов | см. ниже |
| `separators` | Кастомные разделители | `separators=(',', ':')` — компактный вывод |
| `cls` | Класс-кодировщик | подкласс `json.JSONEncoder` |

### 11.6. Кастомная сериализация несериализуемых типов

```python
import json
from datetime import datetime, date
from decimal import Decimal
from uuid import UUID
from dataclasses import dataclass, asdict

@dataclass
class User:
    name: str
    created: datetime
    balance: Decimal
    id: UUID

# Способ 1: функция default
def custom_default(obj):
    if isinstance(obj, datetime):
        return obj.isoformat()
    if isinstance(obj, date):
        return obj.isoformat()
    if isinstance(obj, Decimal):
        return str(obj)
    if isinstance(obj, UUID):
        return str(obj)
    raise TypeError(f"Object of type {type(obj).__name__} is not JSON serializable")

user = User(
    name="Анна",
    created=datetime(2025, 1, 15, 10, 30),
    balance=Decimal("1234.56"),
    id=UUID("12345678-1234-5678-1234-567812345678"),
)

json_str = json.dumps(asdict(user), default=custom_default, ensure_ascii=False, indent=2)
print(json_str)
```

```python
# Способ 2: наследование от JSONEncoder
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)  # осторожно: потеря точности
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)

json.dumps({"dt": datetime.now(), "vals": {1, 2, 3}}, cls=CustomEncoder)
```

### 11.7. Кастомная десериализация

```python
from datetime import datetime

# object_hook — вызывается для каждого JSON-объекта (dict)
def decode_dates(dct):
    for key, value in dct.items():
        if isinstance(value, str):
            try:
                dct[key] = datetime.fromisoformat(value)
            except ValueError:
                pass
    return dct

data = json.loads('{"created": "2025-01-15T10:30:00", "name": "test"}',
                  object_hook=decode_dates)
print(type(data["created"]))  # <class 'datetime.datetime'>

# object_pairs_hook — получает список пар (key, value), полезен для OrderedDict
from collections import OrderedDict
data = json.loads('{"b": 1, "a": 2}', object_pairs_hook=OrderedDict)
```

### 11.8. JSONL (JSON Lines)

Формат, где каждая строка файла — отдельный JSON-объект. Идеален для потоковой обработки и больших данных.

```python
# Запись JSONL
records = [{"id": 1, "value": "a"}, {"id": 2, "value": "b"}]
with open("data.jsonl", "w") as f:
    for record in records:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")

# Чтение JSONL (потоковое — O(1) по памяти)
with open("data.jsonl", "r") as f:
    for line in f:
        record = json.loads(line)
        process(record)
```

**Преимущества JSONL:**
- Можно читать построчно → не нужно загружать весь файл в память.
- Легко дописывать (`append`).
- Удобен для логов, стриминга (Kafka, Kinesis), ELT-пайплайнов.

### 11.9. Альтернативные библиотеки

| Библиотека | Особенности |
|------------|-------------|
| **orjson** | Написана на Rust. В 3–10× быстрее стандартного `json`. Нативная поддержка `datetime`, `UUID`, `numpy`. Возвращает `bytes`, не `str`. |
| **ujson** | Написана на C. В 2–4× быстрее `json`. API совместим. |
| **rapidjson** | Обёртка над C++ RapidJSON. |
| **simplejson** | Расширенная версия стандартного `json` (больше параметров). |

```python
import orjson
from datetime import datetime

data = {"ts": datetime.now(), "values": [1, 2, 3]}
raw: bytes = orjson.dumps(data)  # автоматически сериализует datetime
parsed = orjson.loads(raw)
```

### 11.10. Сравнение с другими форматами сериализации

| Формат | Текст/Бинарный | Язык | Безопасность | Скорость | Размер |
|--------|----------------|------|--------------|----------|--------|
| **JSON** | Текстовый | Любой | Безопасен | Средняя | Большой |
| **pickle** | Бинарный | Только Python | ⚠️ Опасен (произвольное выполнение кода) | Быстрая | Средний |
| **MessagePack** | Бинарный | Любой | Безопасен | Быстрая | Маленький |
| **Protocol Buffers** | Бинарный | Любой (схема .proto) | Безопасен | Очень быстрая | Маленький |
| **Avro** | Бинарный | Любой (схема) | Безопасен | Быстрая | Маленький |

**pickle** — НИКОГДА не десериализуйте данные из непроверенных источников:

```python
import pickle

# pickle.loads() может выполнить ПРОИЗВОЛЬНЫЙ КОД
# Пример вредоносного payload:
class Evil:
    def __reduce__(self):
        import os
        return (os.system, ("rm -rf /",))
```
