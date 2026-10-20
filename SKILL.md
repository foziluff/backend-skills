---
name: foziluff-backend-style
description: Используй при написании, рефакторинге и ревью бэкенд-кода в стиле foziluff на PHP 8.3 / Laravel 13 (Octane + Swoole, Sanctum, MySQL) — слои Controller → Service → Repository, доменные исключения, FormRequest, Resource, транзакции с lockForUpdate.
---

# foziluff backend style

Скилл описывает не только *как* писать код, но и *почему* он написан именно так.
Источник — реальный проект `electrogo/backend` (зарядные станции для электромобилей:
точки, зарядники, коннекторы, сессии зарядки, баланс, OCPP-шлюз).

Пометки в тексте:
- **[гипотеза]** — причина выведена из кода, автор её явно не подтверждал;
- **[подтверждено]** — автор подтвердил решение лично;
- **legacy** — так в проекте сейчас написано, но новый код так писать не нужно.

---

## Философия

1. **Тонкие края, толстый сервис.** Контроллер только принимает и отдаёт, репозиторий только читает и пишет.
   Все решения («можно ли», «сколько списать», «что делать при ошибке») принимает сервис.
   *Почему:* бизнес-правила лежат в одном месте, их можно вызвать из HTTP, cron и OCPP-хендлера одинаково
   (`StopChargingSessionAction` вызывается из всех трёх).

2. **Берём из БД только то, что отдадим.** Явный `select(Model::DEFAULT_SELECT_COLUMNS)`, eager-load
   с выбором колонок, `preventLazyLoading` вне продакшена, и уже после этого — Resource. **[подтверждено]**
   *Почему:* оптимизация закладывается при написании запроса, а не потом; N+1 ловится сразу в dev.

3. **Целостность — через блокировки, а не через надежду.** Любой сценарий «проверил → действую»
   (лимиты, уникальность, баланс, занятость коннектора) выполняется в `transaction()` с `lockForUpdate`.
   *Почему:* деньги и общие ресурсы (коннектор) под конкурентными запросами. Цена — блокировки и
   чуть более длинные транзакции.

4. **Ошибка — это тип.** Каждая бизнес-ошибка — свой пустой класс исключения, HTTP-статус и
   переведённый текст задаются в одном `match` в `bootstrap/app.php`.
   *Почему:* сервис не знает про HTTP, а коды и тексты ответов управляются централизованно. [гипотеза]

5. **Минимум инфраструктуры — максимум из самого проекта.** Прежде чем тащить Redis/очереди/Elastic,
   решаем средствами стека: MySQL spatial + FULLTEXT ngram, `defer()` вместо очередей, bulk `upsert`,
   `chunkById`, scheduler. **[подтверждено]** («стараться максимально избегать и решать на уровне проекта»)
   *Компромисс:* `defer()` теряет задачу при падении воркера, у FULLTEXT-поиска потолок по качеству.

6. **DRY всегда.** Одинаковый код в двух местах выносится сразу. **[подтверждено]**
   (В проекте есть legacy-копия `validateLanguages()` в `PointService` и `CompanyService` — так не делать.)

7. **Строгий вход, предсказуемый выход.** На входе FormRequest с границами на каждое поле,
   на выходе всегда Resource с явными приведениями типов. **[подтверждено]**
   *Почему:* мобильному клиенту нужен стабильный контракт, а мусорный ввод не должен доходить до БД.

---

## Архитектура в одном абзаце

HTTP-запрос попадает в плоский `routes/api.php` (группы `auth:sanctum` → `role:admin`, отдельно `ocpp` с
`OCPPMiddleware`). Контроллер получает данные из FormRequest (`validated()`), сам достаёт `auth()->user()`/`auth()->id()`
и передаёт их **явными аргументами** в `readonly`-сервис. Сервис открывает `transaction()`, через репозитории
проверяет существование и лимиты с блокировкой, при нарушении бросает доменное исключение, иначе пишет
через репозиторий. Побочные эффекты (FCM, SMS, WebSocket-пуши, команды зарядникам) уходят через
«репозитории-шлюзы» в `defer()` и выполняются после ответа. Контроллер оборачивает результат в Resource.
Исключения превращает в JSON глобальный обработчик в `bootstrap/app.php`.

```
routes/api.php ─► Middleware (auth:sanctum, role:admin, OCPPMiddleware)
      │
      ▼
Controller ── FormRequest::validated()  (валидация формы и типов, без обращений к БД)
      │        auth()->user() / auth()->id()  → передаются аргументом
      ▼
Service (readonly) ── transaction(), бизнес-проверки, throw XxxException
      │   └──► Action (readonly, ::handle) — операция, которую делят несколько сервисов
      ▼
Repository (readonly) ── Eloquent / raw SQL / внешние шлюзы (HTTP→WS, FCM, SMS) через defer()
      ▼
Model ── fillable, DEFAULT_SELECT_COLUMNS, casts(), accessors, доменная математика (static)

Выход: Model ─► Resource (Http/Resources/...) ─► JSON без обёртки
Ошибки: XxxException ─► bootstrap/app.php match ─► {"message": __('all.key')} + статус
```

Подробно: [references/architecture.md](references/architecture.md).

---

## Главные правила

### Слои
- Контроллер: одна строка на метод — вызвать сервис и вернуть Resource. Никаких `if`, запросов и Eloquent.
  → [architecture.md](references/architecture.md#controller)
- Сервис ничего не знает про HTTP: никаких `request()`, `auth()`, `abort()`, `response()`. Пользователь приходит аргументом.
  → [architecture.md](references/architecture.md#service)
- Eloquent и `DB::` используются **только** в репозиториях (и в миграциях/сидерах). Raw SQL — только в репозитории.
  → [data-layer.md](references/data-layer.md#repository)
- Внешние системы (WS-шлюз, Firebase, SMS) скрыты за репозиторием-шлюзом, сеть — через `defer()`.
  → [architecture.md](references/architecture.md#repository-gateway)
- Action — для операции, которую вызывают ≥2 сервиса/точки входа. Метод всегда `handle`. **[подтверждено]**
  → [conventions.md](references/conventions.md#actions)
- DI — только constructor promotion в `readonly class`, свойства `private` **без** повторного `readonly`. **[подтверждено]**

### Документация (Swagger генерируется из исходников)
- CI выполняет `php artisan swagger`: генератор читает **код**, а не аннотации. Поэтому: FormRequest — типом параметра,
  `rules()` без обращений к запросу/маршруту/пользователю, явный токен типа в каждом правиле, вызов сервиса как
  `$this->xxxService->method(...)`, строка маппинга `$e instanceof X => [Response::HTTP_*, ...]` на одной строке.
  → [architecture.md](references/architecture.md#swagger)

### Данные
- Репозиторий возвращает `?Model` / `bool` / коллекцию / пагинатор и **никогда не бросает** доменных исключений.
  Сервис превращает `null`/`false` в `XxxNotFoundException`.
- Существование связанных сущностей проверяет сервис через `existsById($id, true)` внутри `transaction()`,
  а не правило `exists:` в FormRequest. [гипотеза: защита от гонок + доменный текст ошибки]
- Каждое чтение — с явным `select(...DEFAULT_SELECT_COLUMNS)`, eager-load — с замыканием, где тоже выбраны колонки.
- Деньги: в БД `decimal(36, 2)`, в модели каст `float`, после **каждой** арифметики `round(..., 2)`.
- Лимиты и бизнес-константы — типизированные константы модели (`public const int MAX_COUNT = 200;`), без магических чисел.
  (legacy: `>= 100` в `ChargerService::create`)
- Массовые обновления — `chunkById` + `upsert`, при сбое пачки — откат на поштучную обработку.
  → [data-layer.md](references/data-layer.md)

### Валидация
- Один FormRequest на действие: `Requests/{Entity}/{Verb}{Entity}Request`. Правила массивом, у каждого числа/строки
  явные `min`/`max`, enum — через `new Enum(X::class)`.
- **Политика остановки** **[подтверждено]**:
  - многополевые **формы** (store/update в админке, редактирование профиля) — **без** `$stopOnFirstFailure` и **без**
    `bail`: фронт сразу получает все ошибки, и пользователю не приходится исправлять их по одной;
  - всё остальное (фильтры, query-параметры, одно-двухполевые действия вроде `send-code`, `verify-phone`,
    старта сессии) — `protected $stopOnFirstFailure = true;` + `'bail'` первым в каждом правиле.
  → [data-layer.md](references/data-layer.md#validation)

### Ответы API
- **Всегда Resource**, в том числе для админки и пустых ответов (`EmptyResource::make()`). **[подтверждено]**
  `response()->json($model)` в существующих контроллерах — legacy.
- `JsonResource::withoutWrapping()` — объект отдаётся без `data`; списки — `{data, next_cursor}` (`CursorPaginationResource`)
  или `{data, next_page_url}` (`LengthAwarePaginationResource` поверх `simplePaginate`).
- Пагинация по умолчанию — cursor по 20.
  → [error-handling.md](references/error-handling.md#api-contract)

### Ошибки и логи
- Новая бизнес-ошибка = класс `App\Exceptions\XxxException extends RuntimeException {}` + строка в `match`
  в `bootstrap/app.php` + ключ в `lang/{ru,tg,en}/all.php`. Все три шага обязательны.
- Денежные потоки и сессии зарядки логируются в канал `charging` с контекстом (`user_id`, `session_id`, `exception`),
  потом исключение пробрасывается дальше (log-and-rethrow). [гипотеза]
  → [error-handling.md](references/error-handling.md)

### Стиль
- `readonly class`, constructor promotion, типы у всех параметров и возвратов, `const int|array` с типом,
  `casts()` как метод с `#[\Override]`, ранний return, `throw new XxxException;` без скобок.
- Сравнение enum — по `->value`: `$session->status->value === ChargingSessionStatus::CHARGING->value`. **[подтверждено]**
- Комментариев почти нет; PHPDoc — только `@property`, `@mixin` и generics (`Collection<int, User>`) для Larastan.
  → [conventions.md](references/conventions.md)

### Тесты
- PHPUnit, Feature-тесты на каждый эндпоинт (happy path + каждое исключение из сервиса + границы валидации),
  Unit — на чистую доменную математику моделей. MySQL test-БД (spatial/FULLTEXT в SQLite не работают).
  `Http::fake()` + `Http::preventStrayRequests()`.
  → [testing.md](references/testing.md)

### Git
- Коммит = одно логическое изменение с осмысленным сообщением (не `changes`). **[подтверждено]**
- **Никаких следов ИИ в git:** без `Co-Authored-By: Claude…`, без «Generated with…», без упоминаний ассистента
  в сообщениях, ветках, PR и коде. **[подтверждено]**
  → [conventions.md](references/conventions.md#git)

---

## Чек-лист перед сдачей кода

**Слои**
- [ ] Контроллер: только `validated()`/`integer()`/`float()` → сервис → `Resource::make()/collection()`.
- [ ] Сервис не вызывает `auth()`, `request()`, `abort()`, `response()`; пользователь/ID пришли аргументом.
- [ ] Нет Eloquent/`DB::` вне `app/Repositories` (и миграций/сидеров).
- [ ] Повторяющаяся логика вынесена в private-метод/Action, а не скопирована (DRY).

**Данные**
- [ ] Каждый запрос — с `select(...DEFAULT_SELECT_COLUMNS)`, связи — через `with([... fn (Relation $q) => $q->select(...)])`.
- [ ] Новая модель: `$fillable`, `DEFAULT_SELECT_COLUMNS`, `$hidden` (timestamps, `deleted_at`, служебное), `casts()`.
- [ ] «Проверил → записал» — внутри `transaction()` с `lockForUpdate`.
- [ ] Деньги округлены `round(..., 2)`; колонки денег `decimal(36, 2)`.
- [ ] Лимиты — константы модели.
- [ ] Миграция: `foreignId()->constrained()->cascadeOnDelete()`, `softDeletes()` для бизнес-сущностей, индексы под реальные `where`.

**Контракт**
- [ ] FormRequest выбран по политике `bail`/`stopOnFirstFailure` (форма или нет); у каждого поля границы.
- [ ] Ответ — Resource с явными `(int)/(float)/(bool)`; пустой ответ — `EmptyResource::make()`.
- [ ] Маршрут: kebab-case во множественном числе, `->whereNumber('id')`, правильная группа (public / `auth:sanctum` / `role:admin`).
- [ ] Новые исключения: класс + `match` в `bootstrap/app.php` + ключ в `lang/ru`, `lang/tg`, `lang/en`.
- [ ] Swagger: `rules()` не читает запрос, у каждого правила есть тип, строка `match` в формате `$e instanceof X => [Response::HTTP_*, …]`.

**Качество**
- [ ] `php vendor/bin/pint` (или `php vendor/bin/pint <path>`) — без изменений.
- [ ] `php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon` — 0 ошибок.
- [ ] `php artisan test` — зелёный; на новый эндпоинт есть Feature-тест.
- [ ] Коммит логичный, без следов ИИ.

---

## Чего никогда не делать

| ❌ Нельзя | Почему |
|---|---|
| Логика, `if`, запросы, `Model::` в контроллере | Правило будет недоступно из cron/OCPP, его придётся дублировать |
| `auth()`/`request()`/`abort()` в сервисе | Сервис вызывается без HTTP (scheduler, OCPP-сообщения) |
| `response()->json($model)` | Модель утекает наружу целиком, контракт меняется вместе со схемой |
| `Model::all()`, `->get()` без `select` | Лишние колонки и переводы; противоречит принципу «бери только нужное» |
| Ленивая загрузка связей | В dev упадёт (`preventLazyLoading`), в prod даст N+1 |
| `exists:` в FormRequest вместо проверки в сервисе | Проверка вне транзакции/блокировки, не доменный текст [гипотеза] |
| `throw new HttpException`/`abort()` для бизнес-ошибок | Только доменные классы + централизованный `match` |
| Исключение без строки в `match` и без перевода | Пользователь получит 500 вместо понятного ответа |
| Проверка баланса/лимита без `lockForUpdate` | Гонка двух запросов → двойное списание / превышение лимита |
| Арифметика денег без `round(..., 2)` | Накопление ошибок float на каждом интервале |
| Копировать метод в два сервиса | DRY всегда [подтверждено] |
| Новые очереди/Redis/Elastic «на всякий случай» | Сначала решаем средствами проекта [подтверждено] |
| `readonly class` + `private readonly` одновременно | Избыточно [подтверждено] |
| Action с методом `execute` | Эталон — `handle` [подтверждено] |
| Маршруты для разработки, обходы OTP, секреты (`firebase.json`, ключи) в репозитории | Утечка доступа в прод (CI деплоит `main` как есть) |
| `Co-Authored-By`/«Generated with» в коммитах | Никаких следов ИИ в git [подтверждено] |
| Магические числа | Константа модели с говорящим именем |

---

## Навигация по references

| Задача | Читай |
|---|---|
| Куда положить новый класс, кто от кого зависит, Service vs Action vs Repository | [references/architecture.md](references/architecture.md) |
| Как назвать файл/метод/маршрут/колонку/ключ перевода, форматирование, git | [references/conventions.md](references/conventions.md) |
| Новая бизнес-ошибка, HTTP-статус, формат ответа, пагинация, логирование | [references/error-handling.md](references/error-handling.md) |
| Миграция, модель, запрос, транзакция, блокировки, валидация, переводы, деньги | [references/data-layer.md](references/data-layer.md) |
| Написать тест, настроить тестовую среду | [references/testing.md](references/testing.md) |
| Посмотреть эталонный код с разбором и анти-примерами | [references/examples.md](references/examples.md) |
| Создать новый CRUD/эндпоинт с нуля | [templates/README.md](templates/README.md) |

## Порядок работы над новой фичей

1. Миграция → модель (+ enum, если есть статус) → factory.
2. Исключения (`XxxNotFoundException` и бизнес-ошибки) → `match` в `bootstrap/app.php` → ключи в трёх `lang`.
3. Репозиторий (только чтение/запись, `lockForUpdate`-флаги).
4. Сервис (транзакции, проверки, исключения).
5. FormRequest'ы (политика `bail`) → Resource → контроллер → маршруты.
6. Feature-тесты → `pint` → `phpstan` → `php artisan test` → логичный коммит без следов ИИ.

Шаблоны всех файлов — в [templates/](templates/README.md).
