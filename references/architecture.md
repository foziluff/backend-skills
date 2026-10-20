# Архитектура

## Карта проекта

```
app/
├── Actions/           # операции, общие для нескольких сервисов/точек входа (readonly, ::handle)
├── Enums/             # string-backed enum: статусы, роли, типы
├── Exceptions/        # пустые доменные исключения, по одному на бизнес-ошибку
├── Http/
│   ├── Controllers/   # тонкие, одна строка на метод
│   ├── Middleware/    # RoleMiddleware (role:admin), OCPPMiddleware (общий секрет шлюза)
│   ├── Requests/{Entity}/{Verb}{Entity}Request.php
│   └── Resources/Client/   # выходной контракт + пагинационные обёртки
├── Models/            # Eloquent: fillable, DEFAULT_SELECT_COLUMNS, casts(), accessors, доменная математика
├── Providers/AppServiceProvider.php   # preventLazyLoading, withoutWrapping, локаль
├── Repositories/      # доступ к БД + шлюзы во внешние системы
├── Services/          # бизнес-логика, транзакции
├── Support/Helpers/helpers.php        # глобальные функции: transaction(), full_url(), accept_language()
└── Traits/HasTranslations.php         # полиморфные переводы
bootstrap/app.php      # middleware-алиасы + центральный маппинг исключений → HTTP
routes/api.php         # все маршруты API в одном файле
routes/console.php     # scheduler (замыкания)
config/custom.php      # все проектные env-настройки
lang/{ru,tg,en}/all.php
```

Код сгруппирован **по слоям, а не по фичам**. Внутри `Requests/` есть подпапки по сущностям,
потому что на одну сущность приходится много FormRequest'ов.

*Почему по слоям:* [гипотеза] проект один, сущностей около 13, большинство сценариев затрагивает
несколько сущностей сразу (сессия = коннектор + пользователь + транзакция + уведомление).
При группировке по фичам такие межмодульные зависимости были бы размазаны по папкам.
*Компромисс:* папки `Services/` и `Repositories/` растут линейно с числом сущностей.

## Направление зависимостей

```
Controller ─► Service ─► Repository ─► Model
                 │            ▲
                 └─► Action ──┘
Repository ─► Repository   (допустимо: MobileCommandRepository → UserRepository, ChargingSessionRepository)
Repository ─► Action       (только «технический» Action: SendWebsocketRequestAction)
Repository ─► Resource     (только для сериализации payload WS-пушей: MobileCommandRepository)
```

Запрещено:
- Service → Controller/Request/Resource (сервис не знает про HTTP);
- Repository → Service (репозиторий не принимает решений);
- Model → Repository/Service;
- Controller → Repository напрямую (legacy: `UserController::devToken` обращается к `User::` — так не делать).

---

<a id="controller"></a>
## Controller

**Только HTTP-слой** **[подтверждено]**. **Что делает:** берёт провалидированные данные, достаёт текущего пользователя, вызывает **один** метод
сервиса и оборачивает результат в Resource.

```php
// app/Http/Controllers/ChargingSessionController.php
public function store(StoreChargingSessionRequest $request): ChargingSessionResource
{
    return ChargingSessionResource::make($this->chargingSessionService->create($request->validated(), auth()->id()));
}
```

Правила:
- Наследуется от пустого `abstract class Controller`.
- Зависимость — **один** сервис, через `private readonly` в constructor promotion.
  (У контроллера класс не `readonly`, поэтому `readonly` стоит на свойстве — это **не** нарушение правила «без двойного readonly».)
- Возвращаемый тип — конкретный Resource (`PointResource`, `CursorPaginationResource`, `AnonymousResourceCollection`, `EmptyResource`).
- Скалярные параметры запроса берутся через `$request->integer('point_id')` / `$request->float('radius')` /
  `$request->validated('phone')`, целиком — `$request->validated()`.
- `auth()->user()` / `auth()->id()` вызываются **только здесь** и передаются в сервис аргументом
  (`getById($id, auth()->id())`).
- Path-параметр типизирован (`int $id`, `string $code`), а маршрут ограничен `->whereNumber('id')`.
- Имена методов — REST (`index`, `show`, `store`, `update`, `destroy`) или глагол действия
  (`stop`, `topUp`, `sendCode`, `verifyPhone`, `getSelf`, `updateSelf`).

*Почему:* одно и то же правило вызывается из HTTP, из scheduler (`routes/console.php` → `autoUpdateProgress`)
и из OCPP-обработчика. Если логика живёт в контроллере, её не вызвать без HTTP-запроса.
*Компромисс:* для тривиальных CRUD сервис часто просто проксирует вызов (`SettingService`) — так и задумано.

---

<a id="service"></a>
## Service

**Что делает:** вся бизнес-логика — проверки, лимиты, расчёты, транзакции, порядок побочных эффектов.

```php
readonly class ChargerService
{
    public function __construct(
        private ChargerRepository $chargerRepository,
        private PointRepository $pointRepository,
    ) {}
```

Правила:
- `readonly class`, зависимости — `private` без повторного `readonly`. **[подтверждено]**
  (legacy с двойным readonly: `UserService`, `NotificationService`, `PaymentService`, `TransactionService`.)
- Зависит от репозиториев своей и **соседних** сущностей и от Action'ов. Другие сервисы не внедряет
  (в проекте нет ни одного Service → Service). [гипотеза: так нет циклов, а общий код выносится в Action]
- Публичный метод = сценарий: `getById`, `create`, `update`, `delete`, `cursorPaginate`, `getInRadius`, `stop`, `topUp`.
- `getById()` **бросает** `XxxNotFoundException`, если репозиторий вернул `null`.
  Этот же `getById()` переиспользуется внутри `update()` (`ChargerService::update`, `ConnectorService::update`).
- Проверки вынесены в private-методы с говорящими именами:
  `getAndValidateConnectorForStart()`, `validateActiveSessionsLimit()`, `validateUserBalanceForStart()`.
- Модифицирующие сценарии обёрнуты в `transaction(function () use (...) { ... })`; внутри все проверки идут
  с `lockForUpdate` (`existsById($id, true)`, `countByPointId($id, true)`).
- Ошибки — только доменные исключения (`throw new ConnectorBusyException;`).
- Ownership-проверка выдаёт `NotFound`, а не 403:
  `if (! $session || $session->user_id !== $authUserId) throw new ChargingSessionNotFoundException;`
  [гипотеза: не раскрываем, что чужой ресурс существует]

---

<a id="action"></a>
## Action

Используется, когда операцию вызывают **несколько** сервисов или точек входа.

| Action | Кто вызывает | Вид |
|---|---|---|
| `StopChargingSessionAction` | `ChargingSessionService::stop`, `ChargingSessionService::autoUpdateProgress`, `OCPPService::handleStatusNotification/handleStopTransaction` | инстанс с DI |
| `SendWebsocketRequestAction` | `MobileCommandRepository`, `OCPPCommandRepository` | static, технический |
| `StoreImageAction` / `DeleteImageAction` | `ConnectorTypeService` | static, технический |

Правила:
- `readonly class`, единственный публичный метод — **`handle`**. **[подтверждено]**
  (legacy: `execute` в `StopChargingSessionAction` и `SendWebsocketRequestAction`.)
- Если Action нужны репозитории — это инстанс с DI (`public function handle(...)`).
  Если это чистая техническая утилита без зависимостей — `public static function handle(...)`.
- Бизнес-Action (`StopChargingSessionAction`) сам владеет своей транзакцией, логированием и побочными эффектами,
  потому что его вызывают из разных контекстов.

*Почему не сервис:* сервис привязан к одной сущности/контроллеру; если бы `OCPPService` внедрял
`ChargingSessionService`, появилась бы связка между сервисами. Action — нейтральная общая точка. [гипотеза]

---

<a id="repository"></a>
## Repository (БД)

```php
readonly class ChargerRepository
{
    public function findById(int $id, bool $lockForUpdate = false): ?Charger
    {
        $query = Charger::where('id', $id)
            ->select(Charger::DEFAULT_SELECT_COLUMNS);

        if ($lockForUpdate) {
            $query->lockForUpdate();
        }

        return $query->first();
    }
```

- **Максимально тупой** **[подтверждено]**: только операции с данными, ни одного бизнес-решения
  (полная таблица «можно / нельзя» — в [data-layer.md](data-layer.md#repository)).
- `readonly class` без интерфейса. Внедряется по конкретному классу.
  *Почему без интерфейсов:* [гипотеза] одна реализация, одна БД; интерфейс дал бы только лишний файл.
  *Компромисс:* в тестах подменять приходится через `$this->mock(ConcreteRepository::class)`, а не через биндинг.
- Возвращает `?Model`, `bool`, `int`, `Collection`, `CursorPaginator`/`Paginator`. **Не бросает** доменные исключения.
- Подробные правила запросов — в [data-layer.md](data-layer.md#repository).

<a id="repository-gateway"></a>
## Repository как шлюз во внешнюю систему

«Репозиторий» в этом проекте — **любой источник или приёмник данных**, не только БД: [гипотеза]

| Репозиторий | Что скрывает |
|---|---|
| `OCPPCommandRepository` | HTTP-команды Node-шлюзу → зарядникам (`RemoteStartTransaction`, `RemoteStopTransaction`, `ChangeConfiguration`) |
| `MobileCommandRepository` | WebSocket-пуши мобильному приложению через Node-шлюз (`ActiveSessions`, `UserUpdated`) |
| `NotificationRepository` | таблица `notifications` + FCM (`FirebasePush`) + SMS-шлюз |
| `PaymentRepository` | способы оплаты (пока заглушка с TODO) |

Правила шлюзов:
- Шлюз — **только транспорт**: получает готовые данные (заголовок, текст, получатель, payload) и доставляет их.
  Что отправить, кому и на каком языке — решает сервис. (legacy: `NotificationRepository` сам собирает тексты
  через `__()` и выбирает перевод/топик — при доработке перенести это в `NotificationService`.)
- Имена событий, действий протокола и статусов ответа — enum, а не строки
  (legacy: `'UserUpdated'`, `'ActiveSessions'`, `'RemoteStartTransaction'`, `'Accepted'` строками в `MobileCommandRepository` и `OCPPCommandRepository`).
- Сеть — внутри `defer(function () use (...) { ... })`, чтобы не задерживать ответ и не держать транзакцию открытой.
  Исключение — когда ответ нужен прямо сейчас (`remoteStartTransaction` возвращает `bool`, иначе сессию не создать).
- Данные для отложенного вызова **сериализуются до `defer`**
  (`$userData = UserResource::make($user)->resolve();` в `MobileCommandRepository::pushUserUpdated`):
  внутри `defer` нет ленивых загрузок, и уходит снимок состояния на момент вызова.
- Ошибка внешней системы не роняет бизнес-операцию: `SendWebsocketRequestAction` логирует её и возвращает `null`.

*Почему `defer()` вместо очередей:* проект работает на Octane/Swoole, `defer` выполняется после отправки ответа
в том же воркере, отдельный queue-воркер не нужен. Job-классы (`SendFirebasePushJob`, `SendSmsJob`, `SendMobileCommandJob` и др.)
были в проекте и удалены 2026-08-17 в пользу `defer`. **[подтверждено принципом «минимум инфраструктуры»]**
*Компромисс:* нет ретраев и персистентности — при падении воркера задача теряется.
Если понадобится гарантированная доставка (платежи), только тогда возвращаемся к очередям.

---

## Model

Модель — это схема данных и доменная математика, без обращений к другим слоям:
- `ChargingSession::calculateCost()`, `calculateIntervalWatts()`, `getEstimatedWattsDelivered()`, `getIntervalCost()` —
  чистые расчёты, которые используют сервис, Action и accessor `amount`;
- вычисляемые статусы через `Attribute` + `$appends` (`Connector::status`, `Charger::is_online`);
- бизнес-константы (`MAX_ACTIVE_SESSIONS_PER_USER`, `PROGRESS_INTERVAL_SECONDS`, `MAX_RADIUS`, `MAX_COUNT`).

Детали — в [data-layer.md](data-layer.md#model).

---

## Внешний контур

```
Мобильное приложение ──HTTP──► Laravel API (Octane/Swoole)
          ▲                         │  POST {WS_SERVER_URL}/broadcast, /command/{remote_code}
          │ WebSocket               ▼  заголовок Ws-Secret
          └──────────────── Node WS/OCPP gateway (отдельный репозиторий) ◄──OCPP──► Зарядники
                                    │
                                    └──► POST /api/ocpp/message, GET /api/ocpp/chargers/{code}  (OCPPMiddleware)
```

- Laravel не держит WebSocket-соединения: этим занимается Node-шлюз (до 2026-08-30 лежал в этом репо как `ws-server/`, `ykc-server/`).
- Входящие OCPP-сообщения разбирает `OCPPService::handleMessage` через `match ($action)` по хендлерам `handleXxx`.

## Фоновые процессы

```php
// routes/console.php
Schedule::call(function (ChargingSessionService $chargingSessionService) {
    $chargingSessionService->autoUpdateProgress();
})->everyMinute()
    ->name('charging:update-progress')
    ->withoutOverlapping();
```

- Scheduler — замыкание, которое вызывает **метод сервиса** (Command-класс `UpdateChargingProgressCommand` удалён 2026-08-15).
- `withoutOverlapping()` обязателен для задач, которые могут не уложиться в интервал.
- Обработка пачками `chunkById(500)`; пачка обрабатывается одной транзакцией (bulk upsert), при сбое —
  поштучно (`processSessionsBulk` → `processSessionsIndividually`).

<a id="swagger"></a>
## Автогенерация документации: почему форма кода важна

CI после деплоя выполняет `php artisan swagger` (пакет автора `foziluff/swagger`) → `public/api-docs.json`, `public/docs.html`.
Генератор **не использует аннотации**: он читает маршруты, рефлексию и **исходный код регулярками**. Поэтому часть
стилевых правил — не вкус, а условие того, что документация будет верной:

| Что делает генератор | Какое правило из этого следует |
|---|---|
| Ищет FormRequest по **типу параметра** метода контроллера (рефлексия) | Входные данные — только через FormRequest в сигнатуре, никаких `$request->validate()` |
| Создаёт FormRequest через `newInstanceWithoutConstructor()` и вызывает `rules()` | `rules()` **не должен** зависеть от запроса, маршрута, пользователя (`$this->route()`, `$this->user()`, `$this->input()`) — иначе генерация упадёт или будет неверной |
| Правила разбирает как **массив** (или строку через `\|`), тип угадывает по строковым токенам `integer` / `numeric` / `boolean` / `array` / `image` / `file` / `mimes` | В каждом правиле есть явный токен типа; правила пишутся массивом |
| `min:`/`max:` → `minimum`/`maximum` (`minLength`/`maxLength`), `required` → required, `new Enum(X::class)` и `in:` → enum | Явные границы и enum-правила сразу попадают в документацию |
| Если в FormRequest есть метод `example()`, его результат становится примером запроса | Можно добавлять `public function example(): array` для сложных тел (в проекте пока не используется) |
| Коды ответов: ищет `throw new X` в теле метода контроллера **и в теле публичного метода сервиса**, который вызван как `$this->{prop}->{method}(` — **на один уровень**; статус берёт из `bootstrap/app.php` | Контроллер вызывает сервис именно как `$this->xxxService->method(...)` через типизированное свойство. Исключения, брошенные в private-хелперах, Action'ах или во вложенном `getById()`, в документацию **не попадут** |
| Маппинг читается регуляркой `$e instanceof Class => [Response::HTTP_CONST` | Строка в `match` — строго в этом виде, на одной строке, статус — константой `Response::HTTP_*` |
| `auth*`-middleware → `security: bearerAuth`; `throttle*` → `429`; FormRequest → `422` | Доступ и лимиты задаются middleware маршрута, а не кодом в контроллере |
| Тег — по имени контроллера, summary — по имени метода | Говорящие имена методов контроллера (`getByCompany`, `topUp`) |

Следствие для новых сервисов [гипотеза — подтвердить]: если важно, чтобы код ответа попал в документацию,
`throw new` должен стоять в теле публичного метода, вызываемого из контроллера (как в `ChargerService::create`).
Проверки, вынесенные в private-методы (`ChargingSessionService::create` → `getAndValidateConnectorForStart`),
остаются хорошим стилем, но их коды генератор не увидит.

## Сводная таблица решений

| Решение | Проблема, которую решает | Компромисс | Статус |
|---|---|---|---|
| Controller → Service → Repository | Логика переиспользуется из HTTP, cron, OCPP | Тонкие сервисы-прокси для CRUD | паттерн |
| Без интерфейсов репозиториев | Меньше кода при единственной реализации | Тесты мокают конкретный класс | [гипотеза] |
| Пустые доменные исключения + `match` | Сервис не знает HTTP, тексты и коды в одном месте | Длинный `match` в `bootstrap/app.php` | паттерн |
| `existsById(..., lock)` в сервисе | Гонки, доменные тексты ошибок | Лишний запрос | [гипотеза] |
| `DEFAULT_SELECT_COLUMNS` везде | Производительность + работа `HasTranslations` | Константу нужно поддерживать при изменении схемы | подтверждено |
| `defer()` вместо очередей | Нет воркера и Redis, ответ не ждёт сети | Нет ретраев | подтверждено |
| MySQL spatial/FULLTEXT вместо PostGIS/Elastic | Одна БД | Потолок качества поиска | подтверждено |
| Bulk upsert + поштучный fallback | Ежеминутный cron на тысячах сессий | Сложный код | паттерн |
| Репозиторий максимально тупой | Поведение меняется только в сервисе; запрос можно переписать без риска | Больше кода в сервисах | подтверждено |
| Enum в коде, колонка `string` | Новое значение — один `case`, без миграции; нет сырых строк | БД не валидирует значения | подтверждено |
| Ownership → 404 | Не раскрывать существование чужих данных | Менее информативно | [гипотеза] |
| Один токен на пользователя (`deleteTokens` при входе) | Одно устройство — одна сессия | Нельзя войти с двух устройств | [гипотеза] |
