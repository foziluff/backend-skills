# Conventions: naming, structure, style

## Where new files go

| What | Where | Name |
|---|---|---|
| Controller | `app/Http/Controllers/` | `{Entity}Controller` (`ChargingSessionController`) |
| FormRequest | `app/Http/Requests/{Entity}/` | `{Verb}{Entity}Request`: `Store`, `Update`, `Index`, `Search`, `Get…` (`StoreChargerRequest`, `IndexPointRequest`) |
| Resource | `app/Http/Resources/Client/` | `{Entity}Resource`; composite ones by meaning (`ActiveSessionsResource`, `TokenResource`, `ContactsResource`) |
| Service | `app/Services/` | `{Entity}Service` |
| Repository | `app/Repositories/` | `{Entity}Repository`; gateway — `{Target}CommandRepository` (`OCPPCommandRepository`, `MobileCommandRepository`) |
| Action | `app/Actions/` | `{Verb}{Object}Action` (`StopChargingSessionAction`, `StoreImageAction`) |
| Model | `app/Models/` | singular (`Connector`) |
| Enum | `app/Enums/` | `{Entity}{Aspect}` (`ChargingSessionStatus`, `TransactionSubType`) or `{Protocol}{Aspect}` (`OCPPProtocolType`) |
| Exception | `app/Exceptions/` | `{Entity}{Problem}Exception`: `NotFound`, `AlreadyExists`, `LimitReached`, `Busy`, `Offline`, `Rejected` |
| Trait | `app/Traits/` | `Has{Feature}` |
| Global helper | `app/Support/Helpers/helpers.php` | snake_case, wrapped in `if (! function_exists(...))` |
| Project config | `config/custom.php` | snake_case keys |
| Translations | `lang/{ru,tg,en}/all.php` | snake_case keys; error key = exception name without `Exception` (`ConnectorBusyException` → `connector_busy`) |

Abbreviations in class names are upper case: `OCPPService`, `OCPPController`, `OCPPMiddleware`, `OCPPCommandRepository`,
`OCPPProtocolType`. (A 2026-08-17 commit renamed `OcppCommandFailedException` → `OCPPCommandFailedException`.)

## Method naming

### Repository (returns data, never throws)

| Prefix | Returns | Example |
|---|---|---|
| `findBy{X}` | `?Model` | `findById(int $id, bool $lockForUpdate = false)`, `findByRemoteCode`, `findByPhone`, `findByChargerAndNumber` |
| `getAll…` / `get{X}By{Y}` | `Collection` | `getAll()`, `getAllByPointId`, `getActiveSessionsByUserId`, `getInRadius` |
| `getLocked{X}By{Y}` | `Collection` with `lockForUpdate` | `getLockedActiveSessionsByIds`, `getLockedUsersByIdsKeyed` |
| `existsBy{X}` | `bool` | `existsById($id, $lockForUpdate)`, `existsByRemoteCode`, `existsByCode` |
| `countBy{X}` | `int` | `countByPointId`, `countActiveSessionsByUserId`, `countByCodes` (bounded by an indexed equality only) |
| `has{X}` | `bool` | `hasActiveSessionsByUserId`, `hasUnread` |
| `cursorPaginate[By{X}]` | `CursorPaginator` | `cursorPaginate()`, `cursorPaginateByChargerId`, `cursorPaginateByUserId` |
| `create` / `update` / `softDelete` | `Model` / `?Model` / `bool` | `update(int $id, array $data): ?Charger` |
| `upsert{X}` / `insert` | `void` | `upsertProgress`, `upsertBalances` |
| `chunk{X}` | `void` + callback | `chunkActiveSessions(500, $cb)`, `chunkStalePendingSessions` |
| gateway verbs | `void`/`bool` | `remoteStartTransaction`, `pushActiveSessions`, `pushUserUpdated`, `sendSms`, `sendFirebasePush` |

### Service (a scenario, throws domain exceptions)

| Method | Meaning |
|---|---|
| `getById(int $id, …): Model` | find or `throw XxxNotFoundException` (unlike the repository's `findById`, which returns `null`) |
| `create(array $data[, int $authUserId]): Model` | create with checks in a transaction |
| `update(int $id, array $data): Model` | update; NotFound if the entity does not exist |
| `delete(int $id): void` | `softDelete`; NotFound if `false` |
| `cursorPaginate…`, `getAll…`, `getInRadius`, `search` | reads |
| private `getAndValidate{X}For{Scenario}()` | fetch an entity and check it is fit for the scenario (returns it) |
| private `validate{Rule}()` | check only, `void`, throws |
| private `handle{Case}()` | a processing branch (`handleValidActiveSession`, `handleBootNotification`) |

## Routes

```php
Route::get('charging-sessions/{id}', [ChargingSessionController::class, 'show'])->whereNumber('id');
Route::post('charging-sessions/{id}/stop', [ChargingSessionController::class, 'stop'])->whereNumber('id');
Route::patch('languages/{code}', [LanguageController::class, 'update'])->where('code', '[a-zA-Z]{2}');
```

- URL: kebab-case, plural resource: `charging-sessions`, `connector-types`, `payment-methods`.
- CRUD: `GET` (list/show), `POST` (create), `PATCH` (partial update, **not** `PUT`), `DELETE`.
- Actions on a resource: `POST {resource}/{id}/{verb}` (`/stop`) or `POST {resource}/{verb}` (`user/top-up`).
- "Own" resource without an id: `GET|PATCH|DELETE user`.
- Every path parameter is constrained: `->whereNumber('id')` or `->where('code', regex)`.
- Groups by access level, top to bottom: public → `auth:sanctum` → nested `role:` + `Role::ADMIN->value` → `ocpp` (`OCPPMiddleware`, `prefix('ocpp')`).
- Rate limits on sensitive public routes: `->middleware('throttle:1,1')` (send-code), `throttle:4,1` (verify-phone); add them to
  expensive public reads too ([security.md](security.md#9-abuse-and-dos-limits)).
- No versioning (`/v1`). [hypothesis: one client, backward compatibility through Resources; minimum client versions are served by
  `android-version`/`ios-version`]

## Database and env

- Tables: snake_case plural (`charging_sessions`, `connector_types`); columns: snake_case.
- Boolean flags `is_*` (`is_connected`, `is_online`); points in time `*_at` (`started_at`, `last_heartbeat_at`).
- Foreign keys `{entity}_id`; status/type/role — a `string` column + a backed enum (never `$table->enum()`).
- Env variables UPPER_SNAKE (`WS_SERVER_URL`, `APP_DEFAULT_LANGUAGE`). **Read only in `config/*.php`**, in code via
  `config('custom.ws_server_url')`. *Why:* with `config:cache`, `env()` outside config files returns `null`.
- New project settings go into `config/custom.php` and into the `#CUSTOM#…` block of `.env.example` (empty value for secrets).

<a id="enum"></a>
## Enums and the ban on raw strings **[confirmed]**

The author's principle: **avoid raw strings in code as much as possible.** Every value from a fixed set is an enum; every
repeated technical identifier is a constant. The set of values lives in code, not in the DB (a `string` column, no
`$table->enum()`), so a new value is added with one `case` and no migration.

| ❌ Raw string (legacy in the project) | ✅ Target |
|---|---|
| `$user->role !== $role` (`RoleMiddleware`, `role` not cast) | cast `'role' => Role::class` on `User`, compare `$user->role?->value !== $role` |
| `match ($action) { 'BootNotification' => ... }` (`OCPPService`) | `match (OCPPAction::tryFrom($action)) { OCPPAction::BOOT_NOTIFICATION => ... }` |
| `in_array($status, ['Preparing', 'Charging', ...])` (`OCPPService::handleStatusNotification`) | enum `OCPPConnectorStatus` with an `isPluggedIn(): bool` method |
| `$measurand === 'Energy.Active.Import.Register'` (`OCPPService::handleMeterValues`) | enum `OCPPMeasurand::ENERGY_ACTIVE_IMPORT_REGISTER` |
| `['status' => 'Accepted']`, `'Invalid'` (`OCPPService`, `OCPPCommandRepository`) | enum `OCPPResponseStatus::ACCEPTED->value` |
| `'event' => 'UserUpdated'`, `'ActiveSessions'` (`MobileCommandRepository`) | enum `MobileEvent::USER_UPDATED->value` |
| `Log::channel('charging')` in 8 places | a channel constant/enum (e.g. `LogChannel::CHARGING->value`) |
| `$phone == '999999999'` (`UserService::sendCode`) | must not exist at all (a dev bypass — see security.md) |

(Enum names in the right column are proposals; these classes do not exist in the project yet.)

Not considered "raw strings": column and relation names in queries, translation keys (`'all.connector_busy'`),
array keys of responses/config, log message texts.

### Enum shape

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

- String-backed only. Cases UPPER_CASE; values lowercase snake (or exactly as an external protocol defines them, e.g. OCPP `'Charging'`).
- Groups of values and rules about values are enum methods (`activeStatuses()`, `isPluggedIn()`), not arrays copied around the code.
- In the model — a cast (`'status' => ChargingSessionStatus::class`).
- In validation — `new Enum(ChargingSessionStatus::class)`.
- **Comparison by `->value`** **[confirmed]**:
  ```php
  if ($session->status->value === ChargingSessionStatus::CHARGING->value) { ... }
  ```
  [hypothesis about the reason: the same values are written into arrays for `upsert`/`insert`
  (`'status' => ChargingSessionStatus::COMPLETED->value`), so string comparison looks the same everywhere.]
- In arrays for `upsert`/`insert`/`DB::table` — always `->value` (bulk operations do not apply casts).
- `in_array` with enum objects of the same enum is fine: `in_array($status, ChargingSessionStatus::activeStatuses())`.
- Unknown external values: `X::tryFrom($raw)` → `null` → explicit branch / early return. Never `X::from()` on untrusted input.

<a id="actions"></a>
## Actions

```php
readonly class {{Verb}}{{Object}}Action
{
    public function __construct(private SomeRepository $someRepository) {}

    public function handle(...): Result { ... }
}
```

- The only public method is **`handle`**. **[confirmed]**
- Static `handle` only for utilities without dependencies (`StoreImageAction::handle($file, 'connector_types')`).

## Code style

Formatting: **Laravel Pint with the default (`laravel`) preset**; there is no `pint.json`.
Static analysis: **Larastan, level 5**:

```bash
php vendor/bin/pint            # whole project
php vendor/bin/pint app/Services/ChargerService.php   # one file
php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon
```

What follows from this (and is consistently visible in the code):
- `! $x` with a space, `fn () =>`, concatenation `'a'.$b` without spaces, trailing commas in multi-line lists.
- `throw new XxxException;` without parentheses.
- Sorted `use` imports; functions via `use function Illuminate\Support\defer;`.

Conventions beyond Pint:
- **Classes:** `readonly class` for services, repositories and Actions; constructor promotion; empty body `{}`.
  Stateless by construction — nothing per-request survives in a long-living Octane worker.
- **Types:** on every parameter and return. Nullable `?Type`. Typed constants: `public const int`, `public const array`,
  `const null UPDATED_AT = null;`.
- **`#[\Override]`** on overridden framework methods (`casts()`, `register()`).
- **Early return / guard clauses** instead of nested `if`s:
  ```php
  if (! isset($data['translations'])) {
      return;
  }
  ```
- **Multi-line conditions**: each sub-expression on its own line, the operator at the end of the line:
  ```php
  if (
      isset($data['remote_code']) &&
      $data['remote_code'] !== $charger->remote_code &&
      $this->chargerRepository->existsByRemoteCode($data['remote_code'], true)
  ) {
  ```
- **Named arguments** only where positional ones are unclear (migrations: `geometry('coordinates', subtype: 'point', srid: 4326)`).
- **Comments:** almost none. Only `// TODO` with a reason. The code explains itself through method names
  (`getAndValidateConnectorForStart`).
- **PHPDoc:** only for the static analyser:
  - `@property` on models for appends, casts and relations (`@property ConnectorStatus $status`, `@property Charger|null $charger`);
  - `@mixin Model` + `@property` on Resources;
  - generics: `@param Collection<int, ChargingSession> $sessions`, `@return array<int, self>`;
  - `/** @var Translation $item */` inside closures.
- **null:** a repository returns `null` → the service throws. Inside code use `?->` and `??`
  (`$user?->balance`, `$this->watts ?? $connectorWatts`).
- **Arrays:** input data is `array $data` from `validated()` (no DTOs). [hypothesis: less ceremony; the FormRequest already defines the shape]
- **Size:** a public service method reads top-down as a scenario; details go into private methods.
  Long methods are acceptable only in batch processing (`processSessionsBulk`), where splitting would make state passing worse.

<a id="git"></a>
## Git

- One logical change = one commit. The message says **what** changed and **why**:
  `Add connector type limit check`, `Move SMS sending to defer`, `Fix refund on pending session stop`.
  Not `changes`. **[confirmed]**
- Language and format are free but consistent within the project (no strict Conventional Commits). **[confirmed: "logical"]**
- **No traces of AI:** no `Co-Authored-By: Claude …` trailers, no "Generated with …" lines, no mention of an assistant in
  messages, PRs, branches or code comments. **[confirmed]** This is the author's rule and it overrides any default attribution settings.
- CI deploys every push to `main` immediately (`git reset --hard`, `migrate --force`, `php artisan swagger`), so nothing
  unverified may reach `main`: pint + phpstan + tests run **before** the push.
