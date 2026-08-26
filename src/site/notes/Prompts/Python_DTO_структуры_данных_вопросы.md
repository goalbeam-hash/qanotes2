---
{"dg-publish":true,"permalink":"/prompts/python-dto-struktury-dannyh-voprosy/","dg-note-properties":{}}
---

# Python для AQA: DTO, структуры данных, язык — вопросы и ответы

Добор к файлу `BI.ZONE_подготовка_к_техничке.md`. Здесь то, что спрашивают у автоматизатора на Python сверх pytest: модели данных и DTO, структуры данных и их стоимость, язык в объёме, который реально всплывает в тестовом коде.

Формат: **вопрос → короткий ответ, который произносится вслух → код → что могут доспросить**.

---

## Содержание

- [A. DTO и модели данных](#a)
- [B. Структуры данных](#b)
- [C. Язык: то, что всплывает в тестах](#c)
- [D. ООП под Page Object](#d)
- [E. Даты, время, часовые пояса](#e)
- [F. Практические задачи с решениями](#f)
- [G. Быстрый прогон: 30 коротких вопросов](#g)

---

<a id="a"></a>

# A. DTO и модели данных

## A1. «Что такое DTO и зачем он в автотестах?»

> DTO — объект, который переносит данные между слоями и ничего не делает сам: без бизнес-логики, только поля. В автотестах он решает три задачи.
>
> **Первая — читаемость.** Вместо `resp.json()["data"]["license"]["expires_at"]` пишется `license.expires_at`. Опечатка в строковом ключе ловится только в рантайме, а в атрибуте — сразу подсветит IDE.
>
> **Вторая — валидация контракта.** Если бэкенд переименовал поле или сменил тип, разбор в модель падает сразу и с внятным сообщением, а не через десять строк на непонятном `KeyError`.
>
> **Третья — тестовые данные.** DTO с дефолтами превращается в фабрику: создаю объект, переопределяя только то, что важно для конкретного теста.

## A2. «Чем сделаешь DTO: dict, dataclass, NamedTuple, TypedDict или Pydantic?»

Это любимый вопрос, потому что показывает, понимает ли человек разницу между подсказкой типов и реальной проверкой.

| | Валидация в рантайме | Изменяемый | Когда беру |
|---|---|---|---|
| `dict` | нет | да | быстрый разбор, одноразово |
| `TypedDict` | **нет**, только подсказка для линтера | да | типизировать существующие словари, не меняя код |
| `NamedTuple` | нет | нет | маленький неизменяемый набор полей, можно как ключ словаря |
| `@dataclass` | нет (типы не проверяются!) | да, если не `frozen` | внутренние модели тестов, фабрики данных |
| **Pydantic** | **да** | да | **разбор ответов API — основной выбор** |

> Ключевое, что надо сказать вслух: **`dataclass` и `TypedDict` типы не проверяют.** Аннотация `expires_at: datetime` в датаклассе не помешает положить туда строку — это подсказка для человека и линтера, а не рантайм-контроль. Поэтому для валидации ответов от сервиса я беру Pydantic: он реально приводит и проверяет типы и падает на расхождении контракта.

## A3. Pydantic — как выглядит на практике

```python
from pydantic import BaseModel, Field, field_validator
from datetime import datetime
from enum import Enum


class LicenseStatus(str, Enum):
    ACTIVE = "active"
    EXPIRED = "expired"
    REVOKED = "revoked"


class License(BaseModel):
    id: int
    key: str = Field(min_length=19, max_length=19)
    status: LicenseStatus
    expires_at: datetime
    seats: int = Field(ge=1)
    product: str

    @field_validator("key")
    @classmethod
    def key_format(cls, v: str) -> str:
        if v != v.upper():
            raise ValueError("ключ должен быть в верхнем регистре")
        return v


# разбор ответа
license = License.model_validate(resp.json())
assert license.status is LicenseStatus.ACTIVE

# обратно в словарь для отправки
payload = license.model_dump(mode="json")
```

**Что здесь стоит проговорить:**

- `Enum` для статусов вместо строк — опечатка `"activ"` не пройдёт, а сравнение читается.
- `Field(ge=1)` — ограничения прямо в модели, не в ассертах.
- `model_validate` падает при любом расхождении контракта: лишнее поле не страшно, а отсутствующее или не того типа — сразу.
- `model_dump(mode="json")` — сериализация с приведением `datetime` к строке.

**Могут доспросить:** «а если бэкенд добавит новое поле?» → По умолчанию Pydantic лишние поля игнорирует. Если нужно ловить расширение контракта — `model_config = ConfigDict(extra="forbid")`. В тестах это обоюдоострая штука: строгий режим ловит незадокументированные изменения, но и ломает тесты при безобидном расширении API. Обычно строго — в контрактных тестах, мягко — в функциональных.

## A4. dataclass — для тестовых данных

```python
from dataclasses import dataclass, field, asdict


@dataclass
class InstallationData:
    name: str = "test-installation"
    product: str = "security-cloud"
    version: str = "1.0.0"
    tags: list[str] = field(default_factory=list)   # ❗ не tags: list = []
    online: bool = True


# в тесте переопределяю только значимое
inst = InstallationData(version="2.1.0", online=False)
payload = asdict(inst)
```

**Три вещи, которые здесь проверяют:**

1. **`field(default_factory=list)`.** Если написать `tags: list = []`, список создастся один раз при определении класса и будет общим для всех экземпляров — классическая ловушка мутабельного дефолта.
2. **`frozen=True`** делает объект неизменяемым и хешируемым — можно класть в `set` или использовать как ключ словаря.
3. **Дефолты** — это и есть «фабрика тестовых данных»: тест указывает только то, что важно для него, остальное берётся из осмысленных умолчаний. Это резко повышает читаемость: видно, что именно в этом тесте существенно.

## A5. «Как строишь тестовые данные для сложных объектов?»

> Тремя способами, по возрастанию сложности объекта.
>
> **Дефолты в датаклассе** — когда объект простой. Тест меняет одно-два поля.
>
> **Фабричные функции** — когда нужны осмысленные пресеты: `expired_license()`, `installation_offline()`. Название несёт смысл, и в тесте читается сценарий, а не набор полей.
>
> **Builder** — когда полей много и комбинаций много: `LicenseBuilder().expired().with_seats(5).build()`.
>
> Плюс `Faker` для значений, которые должны быть уникальными и правдоподобными — имена, email, адреса. Уникальность важна при параллельном прогоне: захардкоженное имя приводит к конфликтам между воркерами.

```python
def expired_license(**overrides):
    defaults = dict(
        status="expired",
        expires_at=datetime.now(timezone.utc) - timedelta(days=1),
        seats=1,
    )
    return License(**{**defaults, **overrides})
```

---

<a id="b"></a>

# B. Структуры данных

## B1. «Чем отличаются list, tuple, set, dict?»

| | Упорядочен | Изменяем | Дубликаты | Поиск `in` |
|---|---|---|---|---|
| `list` | да | да | да | **O(n)** |
| `tuple` | да | нет | да | O(n) |
| `set` | нет* | да | нет | **O(1)** в среднем |
| `dict` | да (порядок вставки) | да | ключи уникальны | **O(1)** в среднем |

\* у `set` нет гарантий порядка; у `dict` порядок вставки гарантирован начиная с Python 3.7.

> Практическое следствие для тестов: если проверяю вхождение в большой набор — перевожу в `set`. Проверка `id in list_of_10000` линейная, `id in set_of_10000` — константная. На выборках из API это разница между секундами и миллисекундами.

## B2. «Сложность основных операций»

| Операция | list | dict / set |
|---|---|---|
| доступ по индексу / ключу | O(1) | O(1) |
| `append` / добавление | O(1) амортизированно | O(1) |
| `insert(0, x)` | O(n) | — |
| `in` | O(n) | O(1) |
| удаление по значению | O(n) | O(1) |

**Могут доспросить:** «а если надо часто добавлять в начало?» → `collections.deque` — двусторонняя очередь, `appendleft` за O(1).

## B3. «Что может быть ключом словаря?»

> Только хешируемый объект — то есть неизменяемый: `str`, `int`, `tuple` (если все элементы внутри тоже хешируемы), `frozenset`, `Enum`. Список или словарь ключом быть не могут: если объект изменится, его хеш перестанет соответствовать месту в таблице.

```python
by_key = {("security-cloud", "2.1.0"): installation}   # ✅ tuple как составной ключ
by_key = {["a", "b"]: 1}                               # ❌ TypeError: unhashable type
```

**Могут доспросить:** «а свой класс?» → По умолчанию хешируется по идентичности объекта. Если переопределили `__eq__`, надо переопределить и `__hash__` — иначе объект станет нехешируемым. `@dataclass(frozen=True)` делает это автоматически.

## B4. «Полезное из collections»

```python
from collections import Counter, defaultdict, deque, namedtuple

Counter(i["product"] for i in installations)        # подсчёт: продукт → сколько
Counter(...).most_common(3)                          # топ-3

d = defaultdict(list)                                # словарь со значением по умолчанию
for inst in installations:
    d[inst["product"]].append(inst["name"])          # без проверки на наличие ключа

deque(maxlen=100)                                    # очередь фиксированной длины
```

`Counter` и `defaultdict` — то, что чаще всего выручает на лайвкодинге, когда просят сгруппировать или посчитать.

## B5. «Копирование: в чём разница между поверхностным и глубоким?»

```python
import copy

original = {"name": "inst", "tags": ["prod", "critical"]}

shallow = copy.copy(original)      # или original.copy() / dict(original)
shallow["tags"].append("new")      # ❗ изменит и original — вложенный список тот же объект

deep = copy.deepcopy(original)
deep["tags"].append("new")         # original не тронут
```

> Для тестов это важно, когда есть общий шаблон данных: если базовый payload копировать поверхностно и менять вложенные поля, тесты начнут влиять друг на друга. Либо `deepcopy`, либо неизменяемые дефолты и построение нового объекта.

## B6. «Список или генератор?»

```python
active = [i for i in installations if i["online"]]         # список: весь в памяти
active = (i for i in installations if i["online"])         # генератор: лениво, один проход
```

> Генератор экономит память и хорош для потоковой обработки, но по нему можно пройти **только один раз**. В тестах это ловушка: прошли генератор в ассерте, потом хотите его же посчитать — получаете пусто. Поэтому в тестовом коде я обычно беру список: данных немного, а предсказуемость важнее.

## B7. «Как отсортировать список словарей?»

```python
sorted(installations, key=lambda i: i["name"])
sorted(installations, key=lambda i: (i["product"], -i["seats"]))    # по двум ключам
sorted(installations, key=itemgetter("created_at"), reverse=True)
```

`sorted` возвращает новый список, `list.sort()` меняет на месте и возвращает `None` — на этом ловят: `x = my_list.sort()` даёт `None`.

## B8. «Множества — где применяешь в тестах?»

```python
expected = {"prod", "staging", "dev"}
actual = {e["name"] for e in resp.json()}

assert actual == expected
assert expected - actual == set()          # чего не хватает
assert actual - expected == set()          # что лишнее
```

> Сравнение списков падает с нечитаемым выводом, если отличается только порядок. Множества дают внятную диагностику: разность сразу показывает, чего не хватает и что появилось лишнего. Но только там, где порядок действительно не важен и дубликаты не значимы.

---

<a id="c"></a>

# C. Язык: то, что всплывает в тестах

## C1. Мутабельный аргумент по умолчанию

```python
def add_tag(tag, tags=[]):     # ❌ список один на все вызовы
    tags.append(tag)
    return tags

def add_tag(tag, tags=None):   # ✅
    tags = list(tags or [])
    tags.append(tag)
    return tags
```

Самая частая «ловушка на понимание» в питон-собесах. Причина: значения по умолчанию вычисляются один раз при определении функции, а не при каждом вызове.

## C2. `is` против `==`

> `==` сравнивает значения, `is` — что это один и тот же объект в памяти. В ассертах почти всегда `==`. `is` — только для `None`, `True`, `False` и для членов `Enum`.

```python
assert resp.json()["status"] == "active"     # ✅
assert value is None                          # ✅
assert count is 5                             # ❌ работает случайно, из-за кеша малых чисел
```

## C3. Исключения

```python
try:
    resp = requests.get(url, timeout=10)
    resp.raise_for_status()
except requests.Timeout:
    ...                       # конкретное исключение
except requests.HTTPError as e:
    ...
except Exception:             # ❌ так в тестах не надо — проглотит и падение ассерта
    ...
finally:
    session.close()           # выполнится всегда
```

> Голый `except:` или `except Exception:` в тестовом коде — плохая практика: он проглатывает в том числе `AssertionError`, и тест становится всегда зелёным. Ловлю только то, что действительно ожидаю обработать.

## C4. Контекстные менеджеры

```python
with open("data.json") as f:          # файл закроется даже при исключении
    data = json.load(f)

with pytest.raises(ValueError):
    parse_key("bad")

# свой
from contextlib import contextmanager

@contextmanager
def temporary_license(api, **kwargs):
    lic = api.create_license(**kwargs)
    try:
        yield lic
    finally:
        api.delete_license(lic.id)
```

## C5. Декораторы

> Функция, которая оборачивает другую функцию, не меняя её код. Весь pytest на них построен. Уметь прочитать обязательно; написать простой — желательно.

```python
import functools

def log_step(name):
    def decorator(func):
        @functools.wraps(func)         # сохраняет __name__ и docstring
        def wrapper(*args, **kwargs):
            print(f"--- {name}")
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

`functools.wraps` — деталь, которую любят спрашивать: без неё обёрнутая функция теряет имя, и pytest с отчётами начинают показывать `wrapper` вместо реального имени теста.

## C6. `*args` и `**kwargs`

```python
def create_installation(name, **overrides):
    payload = {"name": name, "product": "security-cloud", "online": True}
    payload.update(overrides)          # тест переопределяет только нужное
    return api.post("/installations", json=payload)

create_installation("inst-1", online=False, version="2.1.0")
```

Это основной приём для фабрик тестовых данных — стоит показать именно в таком виде.

## C7. Распаковка и объединение

```python
merged = {**defaults, **overrides}       # правое побеждает
combined = [*base_tags, *extra_tags]
first, *rest = versions
a, b = b, a
```

## C8. Работа с JSON

```python
import json

data = json.loads(raw_string)                       # строка → объект
raw = json.dumps(data, ensure_ascii=False, indent=2)  # объект → строка

with open("expected.json", encoding="utf-8") as f:
    expected = json.load(f)                          # файл → объект
```

`loads`/`dumps` — со строкой, `load`/`dump` — с файлом. Буква `s` = string. На этом иногда ловят.

## C9. Регулярные выражения

```python
import re

re.fullmatch(r"[A-Z0-9]{4}(-[A-Z0-9]{4}){3}", key)   # ключ целиком
re.search(r"version=(\d+\.\d+\.\d+)", log)            # найти где угодно
m = re.search(r"id=(?P<id>\d+)", text)
m.group("id")
```

`match` — с начала строки, `search` — где угодно, `fullmatch` — вся строка целиком. Для валидации формата почти всегда нужен `fullmatch`, а не `match` — типичная ошибка.

## C10. GIL и параллельность

> GIL не даёт нескольким потокам одновременно выполнять питоновский байткод. Для тестов это значит: потоки не ускорят вычисления, но **отлично работают на ожидании ввода-вывода** — HTTP-запросы, чтение файлов. Поэтому параллельная отправка запросов через `ThreadPoolExecutor` даёт реальный выигрыш, а pytest-xdist использует **процессы**, а не потоки — именно чтобы обойти GIL.

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=10) as pool:
    results = list(pool.map(lambda i: api.get(f"/installations/{i}"), ids))
```

Это же — способ воспроизвести гонку: параллельные одинаковые запросы и проверка инварианта.

---

<a id="d"></a>

# D. ООП под Page Object

## D1. «Наследование или композиция?»

> В Page Object обычно и то и другое. Наследование — от `BasePage`, где живут общие операции: открыть, дождаться загрузки, доступ к драйверу. Композиция — для повторяющихся кусков интерфейса: шапка, боковое меню, модалка. Компонент — отдельный класс, страница держит его как поле.
>
> Правило: наследование — когда «страница **является** частным случаем базовой», композиция — когда «страница **содержит** этот блок». Таблица не является страницей, поэтому она поле, а не родитель.

```python
class BasePage:
    def __init__(self, page):
        self.page = page
        self.header = HeaderComponent(page)      # композиция


class InstallationsPage(BasePage):               # наследование
    def open(self):
        self.page.goto(f"{BASE_URL}/installations")
        return self
```

## D2. «`classmethod`, `staticmethod`, `property`?»

```python
class License:
    def __init__(self, expires_at):
        self.expires_at = expires_at

    @property
    def is_expired(self):                        # обращение как к полю: lic.is_expired
        return self.expires_at < datetime.now(timezone.utc)

    @classmethod
    def from_api(cls, payload):                  # альтернативный конструктор
        return cls(expires_at=parse(payload["expires_at"]))

    @staticmethod
    def format_key(raw):                         # утилита, не нужен ни self, ни cls
        return raw.upper().replace(" ", "-")
```

## D3. «Зачем `__repr__` в тестовых моделях?»

> Прямая практическая польза: при падении ассерта pytest печатает объекты, и без `__repr__` вы увидите `<License object at 0x7f...>`, что бесполезно. С `__repr__` — `License(id=42, status='expired')`, и причина падения видна сразу, без перезапуска с отладчиком.

У `@dataclass` и Pydantic `__repr__` генерируется автоматически — ещё один аргумент в их пользу против голых словарей.

## D4. «`__eq__` и `__hash__`»

> Если переопределяете `__eq__`, надо думать и про `__hash__`: объект с собственным `__eq__` без `__hash__` становится нехешируемым и не ляжет в `set`. Контракт простой: равные объекты обязаны иметь равный хеш. `@dataclass(frozen=True)` делает оба корректно.

---

<a id="e"></a>

# E. Даты, время, часовые пояса

Для их продукта — лицензии с датами истечения — это не абстракция, а прямая зона багов.

```python
from datetime import datetime, timezone, timedelta

datetime.now()                       # ❌ naive — без часового пояса
datetime.now(timezone.utc)           # ✅ aware

# парсинг ISO 8601
datetime.fromisoformat("2026-08-25T15:30:00+00:00")

# сравнение naive и aware бросит TypeError — частая ошибка
```

**Что сказать на вопрос про тестирование лицензий:**

> Первое, что уточняю: истечение считается по UTC или по локальному времени инсталляции. Это самый частый источник расхождения на сутки, особенно когда сервер в UTC, а инсталляция в UTC+10.
>
> В тестах даты никогда не хардкожу — вычисляю относительно текущего момента: `now + 1 день`, `now - 1 секунда`. Хардкод `2026-01-01` протухает и превращается в тест, который однажды сам покраснеет.
>
> И проверяю ровно три точки: последняя секунда до истечения, момент истечения, первая секунда после. Плюс переход через полночь и через границу часового пояса.

---

<a id="f"></a>

# F. Практические задачи с решениями

### F1. Сгруппировать по ключу

**Задача:** из списка инсталляций получить словарь «продукт → список имён».

```python
from collections import defaultdict

def group_by_product(installations):
    result = defaultdict(list)
    for inst in installations:
        result[inst["product"]].append(inst["name"])
    return dict(result)
```

### F2. Найти дубликаты

**Задача:** телеметрия прислала события, найти повторяющиеся id.

```python
from collections import Counter

def find_duplicates(events):
    counts = Counter(e["id"] for e in events)
    return [event_id for event_id, n in counts.items() if n > 1]
```

**Что сказать:** «это ровно та проверка, которая нужна для дедупликации телеметрии — событие пришло дважды, счётчики не должны удвоиться».

### F3. Сравнить два набора и показать разницу

```python
def diff_installations(expected, actual):
    exp = {i["id"] for i in expected}
    act = {i["id"] for i in actual}
    return {
        "missing": sorted(exp - act),
        "unexpected": sorted(act - exp),
    }
```

### F4. Плоский разбор вложенного JSON

**Задача:** достать все версии продуктов из вложенной структуры.

```python
def collect_versions(payload):
    return {
        product["name"]: product["version"]
        for installation in payload["installations"]
        for product in installation.get("products", [])
    }
```

`.get("products", [])` вместо `["products"]` — защита от отсутствующего ключа. На собесе это заметят как аккуратность.

### F5. DTO + разбор ответа + тест

```python
from pydantic import BaseModel
from datetime import datetime


class Installation(BaseModel):
    id: int
    name: str
    product: str
    version: str
    online: bool
    last_seen: datetime


def test_installation_contract(api_client):
    resp = api_client.get("/installations/42")
    assert resp.status_code == 200

    inst = Installation.model_validate(resp.json())   # контракт
    assert inst.online is True                         # бизнес-проверка
    assert inst.version == "2.1.0"
```

**Что подчеркнуть вслух:** «разделяю две проверки — контракт отдельно, бизнес-логика отдельно. Если сломался контракт, я это вижу как ошибку валидации, а не как непонятное падение ассерта».

### F6. Устойчивое сравнение словарей

**Задача:** сравнить ответ с эталоном, игнорируя изменчивые поля.

```python
IGNORED = {"id", "created_at", "updated_at", "trace_id"}

def normalize(payload, ignored=IGNORED):
    return {k: v for k, v in payload.items() if k not in ignored}


def test_response_matches_expected(api_client, expected_payload):
    actual = api_client.get("/installations/42").json()
    assert normalize(actual) == normalize(expected_payload)
```

---

<a id="g"></a>

# G. Быстрый прогон: 30 коротких вопросов

Проверьте себя — на каждый должен быть ответ в одно-два предложения.

**DTO и модели**

1. Чем `dataclass` отличается от Pydantic-модели? → Датакласс типы **не проверяет** в рантайме, Pydantic проверяет и приводит.
2. Что делает `field(default_factory=list)`? → Создаёт новый список на каждый экземпляр вместо общего.
3. Зачем `frozen=True`? → Неизменяемость плюс хешируемость, можно в `set` и как ключ.
4. Чем `TypedDict` отличается от `NamedTuple`? → Первый — это словарь с подсказками типов, второй — неизменяемый кортеж с именами полей.
5. Как в Pydantic запретить лишние поля? → `model_config = ConfigDict(extra="forbid")`.
6. Зачем `Enum` для статусов? → Опечатка ловится сразу, значения в одном месте, сравнение читается.
7. Как сериализовать модель обратно в JSON? → `model_dump(mode="json")`.

**Структуры данных**

8. Сложность `in` для списка и для множества? → O(n) и O(1) в среднем.
9. Что может быть ключом словаря? → Хешируемое, то есть неизменяемое.
10. Гарантируется ли порядок в `dict`? → Да, порядок вставки, с Python 3.7.
11. Разница `copy` и `deepcopy`? → Поверхностная копия делит вложенные объекты, глубокая — нет.
12. `sorted` и `list.sort()`? → Первая возвращает новый список, вторая меняет на месте и возвращает `None`.
13. Когда `deque` вместо `list`? → Когда часто добавляем и удаляем с начала.
14. Что вернёт `Counter(...).most_common(3)`? → Три самых частых элемента с количествами.
15. Чем генератор отличается от списка? → Ленивый, экономит память, проходится один раз.
16. Как найти, чего не хватает в ответе? → Разностью множеств: `expected - actual`.

**Язык**

17. Почему `def f(x=[])` — плохо? → Дефолт вычисляется один раз, список общий для всех вызовов.
18. `is` или `==` для сравнения строк? → `==`.
19. Чем плох `except Exception` в тесте? → Проглотит и `AssertionError`, тест станет всегда зелёным.
20. Что делает `functools.wraps`? → Сохраняет имя и метаданные обёрнутой функции.
21. `json.load` и `json.loads`? → Файл и строка; `s` = string.
22. `re.match`, `search`, `fullmatch`? → С начала, где угодно, вся строка целиком.
23. Что такое GIL и как он влияет на тесты? → Мешает параллелить вычисления в потоках, не мешает ждать ввод-вывод; xdist поэтому на процессах.
24. Как сделать словарь из двух с приоритетом второго? → `{**a, **b}`.
25. Зачем `with`? → Гарантированное освобождение ресурса даже при исключении.

**ООП и практика**

26. Наследование или композиция для компонентов страницы? → Композиция: страница содержит компонент, а не является им.
27. Зачем `__repr__` в модели? → Читаемая диагностика при падении ассерта.
28. Что будет, если переопределить `__eq__` без `__hash__`? → Объект станет нехешируемым.
29. `classmethod` против `staticmethod`? → Первый получает класс и годится для альтернативных конструкторов, второй — просто утилита в неймспейсе класса.
30. Почему нельзя хардкодить даты в тестах? → Протухают; считать относительно текущего момента.

---

## Что сказать, если вопрос не знаете

Не угадывать. Формула:

> «С этим конкретно не сталкивался. Рассуждаю так: …» — и дальше рассуждение вслух.

Инженерное рассуждение вслух оценивается выше, чем случайное попадание в ответ. И заметно выше, чем молчание.
