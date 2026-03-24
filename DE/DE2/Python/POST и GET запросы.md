#### 2.1 Что такое HTTP и зачем нужны методы

HTTP (HyperText Transfer Protocol) — протокол прикладного уровня для обмена данными между клиентом и сервером. Каждый HTTP-запрос состоит из: строки запроса (метод + URL), заголовков (headers), тела (body — опционально). HTTP-методы указывают серверу, какое действие клиент хочет выполнить. Два основных — GET и POST.

#### 2.2 GET-запрос

**Назначение:** получение данных (ресурса) с сервера.

Характеристики:
- Параметры передаются в URL через query string: `?key1=value1&key2=value2`.
- Тело запроса пустое (по спецификации допускается, но на практике не используется).
- **Идемпотентный** — повторный вызов даёт тот же результат, не меняет состояние сервера.
- **Безопасный (safe)** — не изменяет данные на сервере.
- Кэшируется браузерами, прокси-серверами, CDN.
- Данные видны в URL → логи, история браузера, закладки. Нельзя передавать пароли и конфиденциальные данные.
- Длина URL ограничена (обычно 2048–8192 символов в зависимости от браузера/сервера).

Пример на Python (requests):
```python
import requests

response = requests.get(
    'https://api.example.com/users',
    params={'page': 1, 'limit': 10},
    headers={'Authorization': 'Bearer <token>'}
)
print(response.status_code)  # 200
data = response.json()       # десериализация JSON из тела ответа
```

Пример через urllib (стандартная библиотека):
```python
from urllib.request import urlopen, Request
from urllib.parse import urlencode
import json

params = urlencode({'q': 'python', 'page': 1})
url = f"https://api.example.com/search?{params}"
req = Request(url, headers={'Accept': 'application/json'})
with urlopen(req) as resp:
    data = json.loads(resp.read().decode())
```

#### 2.3 POST-запрос

**Назначение:** отправка данных на сервер для создания/изменения ресурсов.

Характеристики:
- Данные передаются в **теле запроса** (body), а не в URL.
- **Не идемпотентный** — повторный вызов может создать ещё одну запись.
- **Не безопасный** — предполагает изменение состояния сервера.
- Не кэшируется по умолчанию.
- Размер тела не ограничен URL-лимитами (ограничения задаёт сервер).
- Поддерживает разные Content-Type: `application/json`, `application/x-www-form-urlencoded`, `multipart/form-data`.

Примеры:
```python
import requests

# JSON
response = requests.post(
    'https://api.example.com/users',
    json={'name': 'Иван', 'email': 'ivan@example.com'},
    headers={'Authorization': 'Bearer <token>'}
)

# Form data
response = requests.post(
    'https://api.example.com/login',
    data={'username': 'user', 'password': 'pass'}
)

# File upload (multipart/form-data)
with open('report.csv', 'rb') as f:
    response = requests.post(
        'https://api.example.com/upload',
        files={'file': ('report.csv', f, 'text/csv')},
        data={'description': 'Monthly report'}
    )
```

#### 2.4 Сравнительная таблица

| Характеристика | GET | POST |
|----------------|-----|------|
| Назначение | Чтение данных | Создание / изменение данных |
| Данные | В URL (query string) | В теле запроса (body) |
| Идемпотентность | Да | Нет |
| Кэширование | Да | Нет |
| Размер данных | Ограничен длиной URL | Ограничен настройками сервера |
| Видимость данных | В URL, логах, истории | Скрыты в теле |
| Закладки/повтор | Легко | Браузер спросит подтверждение |

#### 2.5 Другие HTTP-методы (сопутствующий вопрос)

| Метод | Назначение | Идемпотентный? |
|-------|-----------|----------------|
| PUT | Полная замена ресурса | Да |
| PATCH | Частичное обновление ресурса | Нет (по спецификации) |
| DELETE | Удаление ресурса | Да |
| HEAD | Как GET, но без тела ответа (только заголовки) | Да |
| OPTIONS | Узнать поддерживаемые методы | Да |

#### 2.6 Что такое REST (сопутствующий вопрос)

REST (Representational State Transfer) — архитектурный стиль API, где ресурсы адресуются через URL, а действия определяются HTTP-методами:
- `GET /users` — список пользователей
- `GET /users/42` — конкретный пользователь
- `POST /users` — создать пользователя
- `PUT /users/42` — обновить пользователя
- `DELETE /users/42` — удалить пользователя

#### 2.7 Коды ответов HTTP (сопутствующий вопрос)

| Группа | Значение | Примеры |
|--------|----------|---------|
| 1xx | Информационные | 100 Continue |
| 2xx | Успех | 200 OK, 201 Created, 204 No Content |
| 3xx | Перенаправление | 301 Moved Permanently, 302 Found |
| 4xx | Ошибка клиента | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Ошибка сервера | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

#### 2.8 Сессии в requests (сопутствующий вопрос)

`requests.Session()` сохраняет cookies, headers и connection pool между запросами:
```python
session = requests.Session()
session.headers.update({'Authorization': 'Bearer <token>'})
session.get('https://api.example.com/data1')
session.get('https://api.example.com/data2')  # токен передаётся автоматически
```
