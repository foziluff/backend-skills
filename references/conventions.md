# Конвенции: именование, структура, стиль

## Размещение новых файлов

| Что | Куда | Имя |
|---|---|---|
| Контроллер | `app/Http/Controllers/` | `{Entity}Controller` (`ChargingSessionController`) |
| FormRequest | `app/Http/Requests/{Entity}/` | `{Verb}{Entity}Request`: `Store`, `Update`, `Index`, `Search`, `Get…` (`StoreChargerRequest`, `IndexPointRequest`) |
| Resource | `app/Http/Resources/Client/` | `{Entity}Resource`; составные — по смыслу (`ActiveSessionsResource`, `TokenResource`, `ContactsResource`) |
| Сервис | `app/Services/` | `{Entity}Service` |
| Репозиторий | `app/Repositories/` | `{Entity}Repository`; шлюз — `{Target}CommandRepository` (`OCPPCommandRepository`, `MobileCommandRepository`) |
| Action | `app/Actions/` | `{Verb}{Object}Action` (`StopChargingSessionAction`, `StoreImageAction`) |
| Модель | `app/Models/` | единственное число (`Connector`) |
| Enum | `app/Enums/` | `{Entity}{Aspect}` (`ChargingSessionStatus`, `TransactionSubType`) |
| Исключение | `app/Exceptions/` | `{Entity}{Problem}Exception`: `NotFound`, `AlreadyExists`, `LimitReached`, `Busy`, `Offline`, `Rejected` |
| Трейт | `app/Traits/` | `Has{Feature}` |
| Глобальный хелпер | `app/Support/Helpers/helpers.php` | snake_case, обёрнут в `if (! function_exists(...))` |
| Конфиг проекта | `config/custom.php` | snake_case-ключи |
| Переводы | `lang/{ru,tg,en}/all.php` | snake_case-ключи; ключ ошибки = имя исключения без `Exception` (`ConnectorBusyException` → `connector_busy`) |

Аббревиатуры в именах классов пишутся заглавными: `OCPPService`, `OCPPController`, `OCPPMiddleware`,
`OCPPCommandRepository`, `OCPPProtocolType`. (Коммит 2026-08-17 переименовал `OcppCommandFailedException` → `OCPPCommandFailedException`.)

## Именование методов

### Репозиторий (возвращает данные, не бросает)

| Префикс | Возврат | Пример |
|---|---|---|
| `findBy{X}` | `?Model` | `findById(int $id, bool $lockForUpdate = false)`, `findByRemoteCode`, `findByPhone`, `findByChargerAndNumber` |
| `getAll…` / `get{X}By{Y}` | `Collection` | `getAll()`, `getAllByPointId`, `getActiveSessionsByUserId`, `getInRadius` |
| `getLocked{X}By{Y}` | `Collection` с `lockForUpdate` | `getLockedActiveSessionsByIds`, `getLockedUsersByIdsKeyed` |
| `existsBy{X}` | `bool` | `existsById($id, $lockForUpdate)`, `existsByRemoteCode`, `existsByCode` |
| `countBy{X}` | `int` | `countByPointId`, `countActiveSessionsByUserId`, `countByCodes` |
| `has{X}` | `bool` | `hasActiveSessionsByUserId`, `hasUnread` |
| `cursorPaginate[By{X}]` | `CursorPaginator` | `cursorPaginate()`, `cursorPaginateByChargerId`, `cursorPaginateByUserId` |
| `create` / `update` / `softDelete` | `Model` / `?Model` / `bool` | `update(int $id, array $data): ?Charger` |
| `upsert{X}` / `insert` | `void` | `upsertProgress`, `upsertBalances` |
| `chunk{X}` | `void` + callback | `chunkActiveSessions(500, $cb)`, `chunkStalePendingSessions` |
| глаголы шлюзов | `void`/`bool` | `remoteStartTransaction`, `pushActiveSessions`, `pushUserUpdated`, `sendSms`, `sendFirebasePush` |

### Сервис (сценарий, бросает доменные исключения)

| Метод | Смысл |
|---|---|
| `getById(int $id, …): Model` | найти или `throw XxxNotFoundException` (в отличие от репозиторного `findById`, который возвращает `null`) |
| `create(array $data[, int $authUserId]): Model` | создать с проверками в транзакции |
| `update(int $id, array $data): Model` | обновить; если сущности нет — NotFound |
| `delete(int $id): void` | `softDelete`; если `false` — NotFound |
| `cursorPaginate…`, `getAll…`, `getInRadius`, `search` | чтение |
| private `getAndValidate{X}For{Scenario}()` | достать сущность и проверить её пригодность для сценария (возвращает её) |
| private `validate{Rule}()` | только проверка, `void`, бросает исключение |
| private `handle{Case}()` | ветка обработки (`handleValidActiveSession`, `handleBootNotification`) |

## Маршруты

```php
Route::get('charging-sessions/{id}', [ChargingSessionController::class, 'show'])->whereNumber('id');
Route::post('charging-sessions/{id}/stop', [ChargingSessionController::class, 'stop'])->whereNumber('id');
Route::patch('languages/{code}', [LanguageController::class, 'update'])->where('code', '[a-zA-Z]{2}');
```

- URL — kebab-case, ресурс во множественном числе: `charging-sessions`, `connector-types`, `payment-methods`.
- CRUD: `GET` (list/show), `POST` (create), `PATCH` (частичное обновление, **не** `PUT`), `DELETE`.
- Действия над ресурсом — `POST {resource}/{id}/{verb}` (`/stop`) или `POST {resource}/{verb}` (`user/top-up`).
- «Свой» ресурс — без id: `GET|PATCH|DELETE user`.
- Каждый path-параметр ограничен: `->whereNumber('id')` или `->where('code', regex)`.
- Группы по уровню доступа, сверху вниз: публичные → `auth:sanctum` → вложенная `role:` + `Role::ADMIN->value` → `ocpp` (`OCPPMiddleware`, `prefix('ocpp')`).
- Rate limit на чувствительных публичных маршрутах: `->middleware('throttle:1,1')` (send-code), `throttle:4,1` (verify-phone).
- Версионирования (`/v1`) нет. [гипотеза: один клиент, обратная совместимость через Resource; минимальные версии клиентов задаются в `android-version`/`ios-version`]

## БД и env

- Таблицы — snake_case во множественном числе (`charging_sessions`, `connector_types`); колонки — snake_case.
- Булевы флаги — `is_*` (`is_connected`, `is_online`); моменты времени — `*_at` (`started_at`, `last_heartbeat_at`).
- Внешние ключи — `{entity}_id`; статус/тип — `string`-колонка + backed enum.
- Env-переменные — UPPER_SNAKE (`WS_SERVER_URL`, `APP_DEFAULT_LANGUAGE`). **Читаются только в `config/*.php`**,
  в коде — `config('custom.ws_server_url')`. *Почему:* при `config:cache` `env()` вне конфигов возвращает `null`.
- Новые проектные настройки добавляются в `config/custom.php`, а в `.env.example` — в блок `#CUSTOM#…`.

<a id="enum"></a>
## Enum и запрет сырых строк **[подтверждено]**

Принцип автора: **максимально избегать сырых строк в коде.** Любое значение из фиксированного набора — enum,
любой повторяющийся технический идентификатор — константа. Набор значений живёт в коде, а не в БД
(колонка `string`, без `$table->enum()`), чтобы новое значение добавлялось одним `case` без миграции.

| ❌ Сырая строка (legacy в проекте) | ✅ Как надо |
|---|---|
| `$user->role !== $role` (`RoleMiddleware`, `role` без каста) | каст `'role' => Role::class` в `User`, сравнение `$user->role?->value !== $role` |
| `match ($action) { 'BootNotification' => ... }` (`OCPPService`) | `match (OCPPAction::tryFrom($action)) { OCPPAction::BOOT_NOTIFICATION => ... }` |
| `in_array($status, ['Preparing', 'Charging', ...])` (`OCPPService::handleStatusNotification`) | enum `OCPPConnectorStatus` с методом `isPluggedIn(): bool` |
| `$measurand === 'Energy.Active.Import.Register'` (`OCPPService::handleMeterValues`) | enum `OCPPMeasurand::ENERGY_ACTIVE_IMPORT_REGISTER` |
| `['status' => 'Accepted']`, `'Invalid'` (`OCPPService`, `OCPPCommandRepository`) | enum `OCPPResponseStatus::ACCEPTED->value` |
| `'event' => 'UserUpdated'`, `'ActiveSessions'` (`MobileCommandRepository`) | enum `MobileEvent::USER_UPDATED->value` |
| `Log::channel('charging')` в 8 местах | константа/enum канала (например, `LogChannel::CHARGING->value`) |
| `$phone == '999999999'` (`UserService::sendCode`) | не нужно вовсе (это dev-обход) или константа из конфига |

(Имена enum'ов в правой колонке — предложение; таких классов в проекте пока нет.)

Не считаются «сырыми строками»: имена колонок и связей в запросах, ключи переводов (`'all.connector_busy'`),
ключи массивов ответа/конфига, тексты логов.

### Устройство enum

```php
enum ChargingSessionStatus: string
{
    case PENDING = 'pending';
    case CHARGING = 'charging';
    case COMPLETED = 'completed';
    case CANCELLED = 'cancelled';

    /**
     * @return array<int, self>
     */
    public static function activeStatuses(): array
    {
        return [self::PENDING, self::CHARGING];
    }
}
```

- Только string-backed. Кейсы — UPPER_CASE, значения — lowercase snake.
- Группы значений — static-метод enum'а (`activeStatuses()`), а не массив, раскопированный по коду.
- В модели — каст (`'status' => ChargingSessionStatus::class`).
- **Сравнение — по `->value`** **[подтверждено]**:
  ```php
  if ($session->status->value === ChargingSessionStatus::CHARGING->value) { ... }
  ```
  [гипотеза о причине: те же значения лежат в массивах для `upsert`/`insert`
  (`'status' => ChargingSessionStatus::COMPLETED->value`), и сравнение строк выглядит одинаково везде.]
- В массивах для `upsert`/`insert`/`DB::table` — всегда `->value` (bulk-операции не применяют касты).
- В `in_array` с enum-объектами — `in_array($status, ChargingSessionStatus::activeStatuses())` (это допустимо: сравниваются объекты одного enum).

<a id="actions"></a>
## Actions

```php
readonly class {{Verb}}{{Object}}Action
{
    public function __construct(private SomeRepository $someRepository) {}

    public function handle(...): Result { ... }
}
```

- Единственный публичный метод — **`handle`**. **[подтверждено]**
- Static `handle` — только для утилит без зависимостей (`StoreImageAction::handle($file, 'connector_types')`).

## Стиль кода

Форматирование — **Laravel Pint, пресет по умолчанию (`laravel`)**, отдельного `pint.json` нет.
Статический анализ — **Larastan, level 5**:

```bash
php vendor/bin/pint            # весь проект
php vendor/bin/pint app/Services/ChargerService.php   # один файл
php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon
```

Из этого следует (и последовательно видно в коде):
- `! $x` с пробелом, `fn () =>`, конкатенация `'a'.$b` без пробелов, trailing comma в многострочных списках.
- `throw new XxxException;` — без скобок.
- `use`-импорты отсортированы; функции — `use function Illuminate\Support\defer;`.

Соглашения сверх Pint:
- **Классы:** `readonly class` для сервисов, репозиториев и Action'ов; constructor promotion; пустое тело `{}`.
- **Типы:** у всех параметров и возвратов. Nullable — `?Type`. Типизированные константы: `public const int`, `public const array`, `const null UPDATED_AT = null;`.
- **`#[\Override]`** — на переопределённых методах фреймворка (`casts()`, `register()`).
- **Ранний return / guard clause** вместо вложенных `if`:
  ```php
  if (! isset($data['translations'])) {
      return;
  }
  ```
- **Многострочное условие** — каждое подвыражение на своей строке, оператор в конце строки:
  ```php
  if (
      isset($data['remote_code']) &&
      $data['remote_code'] !== $charger->remote_code &&
      $this->chargerRepository->existsByRemoteCode($data['remote_code'], true)
  ) {
  ```
- **Именованные аргументы** — только где позиционные непонятны (миграции: `geometry('coordinates', subtype: 'point', srid: 4326)`).
- **Комментарии:** почти нет. Пишем только `// TODO` с причиной. Код объясняет себя именами методов
  (`getAndValidateConnectorForStart`).
- **PHPDoc:** только для статического анализатора:
  - `@property` на моделях — для appends, кастов и связей (`@property ConnectorStatus $status`, `@property Charger|null $charger`);
  - `@mixin Model` + `@property` на Resource;
  - generics: `@param Collection<int, ChargingSession> $sessions`, `@return array<int, self>`;
  - `/** @var Translation $item */` внутри замыканий.
- **null:** репозиторий возвращает `null` → сервис бросает исключение. Внутри кода — `?->` и `??`
  (`$user?->balance`, `$this->watts ?? $connectorWatts`).
- **Массивы:** входные данные — `array $data` из `validated()` (DTO нет). [гипотеза: меньше церемоний; FormRequest уже задал форму]
- **Размер:** публичный метод сервиса читается сверху вниз как сценарий; детали — в private-методах.
  Длинные методы допустимы только в batch-обработке (`processSessionsBulk`), где разбиение ухудшило бы передачу состояния.

<a id="git"></a>
## Git

- Одно логическое изменение — один коммит. Сообщение описывает, **что** изменилось и **зачем**:
  `Add connector type limit check`, `Move SMS sending to defer`, `Fix refund on pending session stop`.
  Не `changes`. **[подтверждено]**
- Язык и формат свободные, но единообразные внутри проекта (жёсткого Conventional Commits нет). [подтверждено: «логичные»]
- **Никаких следов ИИ:** без трейлеров `Co-Authored-By: Claude …`, без строк «Generated with …»,
  без упоминаний ассистента в сообщениях, PR, ветках и комментариях в коде. **[подтверждено]**
  Это правило автора, и оно важнее любых настроек атрибуции по умолчанию.
- CI на push в `main` сразу деплоит (`git reset --hard`, `migrate --force`, `php artisan swagger`) —
  значит, в `main` не должно попадать ничего непроверенного: pint + phpstan + тесты выполняются **до** push.
