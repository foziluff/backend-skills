# Data layer: database, models, queries, transactions, validation, mapping

Query performance at scale, denormalization and memory rules — [performance.md](performance.md).

## Migrations

```php
// database/migrations/2026_04_08_000000_create_charging_sessions_table.php
Schema::create('charging_sessions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('connector_id')->constrained()->cascadeOnDelete();
    $table->string('status');
    $table->unsignedBigInteger('watts_delivered')->default(0);
    $table->decimal('price_per_kw', 36, 2);
    $table->decimal('withdrawn_amount', 36, 2)->default(0);
    $table->unsignedTinyInteger('soc')->nullable();
    $table->timestamp('started_at')->nullable();
    $table->timestamps();

    $table->index(['status', 'last_progress_updated_at']);
    $table->index(['user_id', 'status']);
    $table->index(['connector_id', 'status']);
});
```

- Anonymous class `return new class extends Migration`; `down()` — `Schema::dropIfExists`.
- **One migration = one table**, in its final form. `add_x_to_y` migrations were folded into the create migration in history
  (e.g. `add_withdrawn_amount_to_charging_sessions_table` was removed on 2026-08-16).
  [hypothesis: before the first stable release the schema is edited in place. **After a release this is forbidden** —
  CI runs `migrate --force` in production, and an edited old migration will not be applied.]
- FKs: `foreignId()->constrained()->cascadeOnDelete()`.
- Money `decimal(36, 2)`; energy `unsignedBigInteger` (watt-hours); percentages `unsignedTinyInteger`.
- Statuses/types/roles: a `string` column + a PHP backed enum in `app/Enums`, **never** `$table->enum()`. **[confirmed]**
  *Why:* the set of values lives in code — a new value (role, payment method, status) is one `case`, with no migration and no
  ALTER of a big table in production; one list of values serves validation (`new Enum(...)`), casts and Swagger.
  *Trade-off:* the DB does not protect against an invalid value — the FormRequest (`Enum` rule) and the model cast do.
- Column defaults also come from enums: `$table->string('protocol_type')->default(OCPPProtocolType::OCPP_1_6->value);`
- Business entities use `softDeletes()`; uniqueness checks include deleted rows
  (`ChargerRepository::existsByRemoteCode` uses `withTrashed()`).
- **Indexes are part of the query, not an afterthought:** every repository query gets its serving index in the same change;
  composite index column order = equality columns first, then range/sort (`['status', 'last_progress_updated_at']` ↔
  `chunkActiveSessions`). Details — [performance.md](performance.md#queries).
- MySQL-specific features right in the migration:
  `geometry('coordinates', subtype: 'point', srid: 4326)->spatialIndex()`,
  `DB::statement('ALTER TABLE translations ADD FULLTEXT INDEX ... WITH PARSER ngram')`.
- A table without `updated_at` says so explicitly: `timestamp('created_at')->nullable()->useCurrent()` + `const null UPDATED_AT = null;` in the model.
- Prefer normalized tables; derived values are computed, not stored — see [performance.md](performance.md#denormalization).

<a id="model"></a>
## Models

```php
/**
 * @property int $id
 * @property bool $is_connected
 * @property Charger|null $charger
 * @property ConnectorStatus $status
 */
class Connector extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = ['charger_id', 'connector_type_id', 'number', 'watts', 'price_per_kw', 'is_connected'];

    public const array DEFAULT_SELECT_COLUMNS = ['id', 'charger_id', 'connector_type_id', 'number', 'watts', 'price_per_kw', 'is_connected'];

    protected $hidden = ['created_at', 'updated_at', 'deleted_at', 'active_session_exists'];

    protected $appends = ['status'];

    #[\Override]
    protected function casts(): array
    {
        return [
            'charger_id' => 'integer',
            'watts' => 'integer',
            'price_per_kw' => 'float',
            'is_connected' => 'boolean',
        ];
    }
```

Mandatory set for a new model:
1. `$fillable` — an explicit list (never `$guarded = []`).
2. **`public const array DEFAULT_SELECT_COLUMNS`** — the columns the client needs. For models with `HasTranslations` it also
   lists the translatable fields (`'name'`, `'address'`): the trait uses this list to decide which translations to load.
3. `$hidden` — `created_at`, `updated_at`, `deleted_at` and internal/sensitive fields (`remote_code`, `meter_start`, `active_session_exists`).
4. `casts()` with `#[\Override]`: numbers → `integer`/`float`, flags → `boolean`, dates → `datetime`, every fixed-set column → its enum.
5. `@property` in PHPDoc for appends, casts and relations (Larastan level 5).

Additionally:
- Computed fields: `Attribute::make(get: ...)` + `$appends` (`Connector::status`, `Charger::is_online`). They read only
  loaded attributes/relations — **no queries and no I/O in accessors**.
- Normalization on write: `Attribute::make(set: ...)` (`ChargingSession::soc` clamps to 0..100).
- Domain math: `public static` model methods (`calculateCost`, `calculateIntervalWatts`), so one formula is shared by the service,
  the Action and the accessor. **Never** duplicate the formula in a service.
- Business constants: `public const int` (`MAX_ACTIVE_SESSIONS_PER_USER = 100`, `PROGRESS_INTERVAL_SECONDS = 60`).
- Enrichment without a query: a method returning `$this` (`User::withUnreadNotifications(bool)` → `setAttribute` + `setVisible`).
- Scopes (`scopeXxx(Builder $query): void` + `@method static Builder xxx()` in PHPDoc, `Point::scopeWithPrice`) are allowed only as
  building blocks used exclusively by repositories; new query fragments go into private repository methods.

## Global settings (AppServiceProvider)

```php
Model::preventLazyLoading(! app()->isProduction());
JsonResource::withoutWrapping();
app()->setLocale(accept_language());
```

- Lazy loading throws in dev/test. *Why:* N+1 is found during development, not in production.
  *Trade-off:* in production lazy loading silently works — so tests must run outside `production`.

<a id="repository"></a>
## Queries in repositories

**All database queries live in repositories and nowhere else.** **[confirmed]** — [performance.md](performance.md#data-access)

### The repository is as dumb as possible **[confirmed]**

A repository works **only with data** and does exactly what the service asked. It does not know *why* the data is needed.

| Allowed in a repository | Forbidden in a repository (→ service) |
|---|---|
| `select`, `where`, `join`, eager-load, `withExists`, subqueries | business conditions ("if the user is an admin…", "if the balance…") |
| `lockForUpdate` driven by a flag from the service | deciding whether a lock is needed |
| `create` / `update` / `softDelete` / `upsert` / `insert` / `chunkById` | side rules ("null this `fcm_token` for other users") |
| Ordering by a column/index (`latest('id')`, `orderByDesc('relevance')`) | business sorting ("chargers with a free connector first") |
| Technical storage conversions (lat/lng → `POINT`) | building texts (`__()`), choosing a language, a recipient or a topic |
| Returning `null` / `false` when nothing is found | throwing domain exceptions, hard-coded business data |

Legacy that contradicts the rule (move the logic into a service when touching this code):
- `NotificationRepository::sendChargingStoppedNotification` builds translated texts and push `data`;
  `NotificationRepository::send` picks the translation by the user's language and the topic `all_{lang}`;
- `UserRepository::update` / `updateByModel` — the rule "one `fcm_token` belongs to one user";
- `ChargerRepository::getAllByPointId` — sorts chargers by having a free connector;
- `PaymentRepository::getPaymentMethods` — a hard-coded list (TODO).

### Explicit select

```php
// app/Repositories/ChargerRepository.php::getAllByPointId
$chargers = Charger::with([
    'connectors' => fn (Relation $query) => $query
        ->select(Connector::DEFAULT_SELECT_COLUMNS)
        ->withExists('activeSession'),
    'connectors.connectorType' => fn (Relation $query) => $query->select(ConnectorType::DEFAULT_SELECT_COLUMNS),
])
    ->where('point_id', $pointId)
    ->select(Charger::DEFAULT_SELECT_COLUMNS)
    ->get();
```

- Every query: `->select(Model::DEFAULT_SELECT_COLUMNS)`; every relation through a closure with `select`.
  **[confirmed: "select before the resource so it is already optimised"]**
- Relation existence flags: `withExists('activeSession')` instead of loading the relation.
- An inverse relation already in memory is set via `setRelation` instead of a second query
  (`$connector->setRelation('charger', $charger)` in `getAllByPointId`).
- Every read is bounded and index-served — [performance.md](performance.md#queries).

### The lock flag

```php
public function existsById(int $id, bool $lockForUpdate = false): bool
{
    $query = Point::where('id', $id);

    if ($lockForUpdate) {
        $query->lockForUpdate();
    }

    return $query->exists();
}
```

Every read method that may take part in a "check → write" scenario gets `bool $lockForUpdate = false`.
The **service** decides about the lock. Lock rows found by primary key or index only (narrow locks).

### Create / Update / Delete

- `create(array $data): Model` — `Model::create($data)`. If a related write is needed (translations) — its own `transaction()`,
  `save()` + `syncTranslations($data)` inside, then re-read via `findById($id, true)`.
- `update(int $id, array $data): ?Model` — find (with `lockForUpdate` when the same method writes), `null` if missing;
  `update($data)`; **return the re-read model** via `findById` (with the right select and relations).
- `softDelete(int $id): bool` — `Model::where('id', $id)->delete()` (the affected row count → bool).
- A transaction inside a repository only when **the method itself** performs several writes (`save` + `syncTranslations`,
  `lock` + `update` + re-read). Inside a service transaction it becomes a savepoint.
  [hypothesis: a repository method is atomic on its own, even if called outside a service]

### Raw SQL

Only in a repository, always with bindings — never user input interpolated into SQL:

```php
// app/Repositories/PointRepository.php::getInRadius
->whereRaw('MBRContains(ST_GeomFromText(?, 4326), coordinates)', [$bbox])          // cheap prefilter on the spatial index
->whereRaw("ST_Distance_Sphere(coordinates, ST_GeomFromText(CONCAT('POINT(', ?, ' ', ?, ')'), 4326)) <= ?",
    [$latitude, $longitude, $radius])                                                // exact check
->limit(1000)
```

Fragments repeated in several methods become private repository methods (`asLatitudeAndLongitude()`, `asPoint()`).
(legacy: `DB::getPdo()->quote()` in `PointRepository::search` — works, but prefer bindings.)

### Bulk

- Mass update: `Model::upsert($rows, ['id'], ['col1', 'col2'])` (`upsertProgress`, `upsertBalances`).
- Mass insert: `Model::insert($rows)` with manual `created_at`/`updated_at` and enums via `->value`
  (bulk operations apply no casts and fill no timestamps).
- Walking big sets: `chunkById($count, $callback)`.

## Transactions and locks

```php
// app/Services/ChargerService.php::create
return transaction(function () use ($data) {
    if (! $this->pointRepository->existsById($data['point_id'], true)) {
        throw new PointNotFoundException;
    }

    if ($this->chargerRepository->countByPointId($data['point_id'], true) >= 100) {   // legacy: must be a constant
        throw new ChargerLimitReachedException;
    }

    if ($this->chargerRepository->existsByRemoteCode($data['remote_code'], true)) {
        throw new ChargerAlreadyExistsException;
    }

    return $this->chargerRepository->create($data);
});
```

- The **`transaction(Closure)`** helper from `app/Support/Helpers/helpers.php` (a wrapper over `DB::transaction`), not
  `DB::transaction` in services. *Why:* the service does not import the `DB` facade — consistent with "no DB access outside
  repositories". [hypothesis]
- The **service** (or a business Action) opens the transaction for the whole scenario.
- Inside the transaction **every** check the write depends on uses `lockForUpdate`: the parent exists, the limit is not
  exceeded, uniqueness, balance, the connector is free.
- Lock order: shared resources first (the connector), then the user (`ChargingSessionService::create`: connector →
  active sessions → user). Keep the same order in all scenarios. [hypothesis: deadlock prevention]
- Transactions are short: network calls inside a transaction only through `defer()` (they run after the response), except
  when the answer decides the commit (`remoteStartTransaction`: `false` → `OCPPCommandFailedException` → rollback of the charge).

## Money

- Charging is **prepaid one interval ahead** (`getIntervalCost`); on stop — recalculation and a refund of the difference
  (`StopChargingSessionAction`: `refundAmount = intervalCost - exactCost`).
- Every intermediate value is `round(..., 2)`.
- Every final charge is recorded in `transactions` (`TransactionType::WITHDRAWAL`, `TransactionSubType::CHARGING`);
  a top-up is `REPLENISHMENT`.
- `users.balance` is a denormalized running total — kept consistent by the rules in
  [performance.md](performance.md#denormalization) (one write path, same transaction, row lock, reconstructible).
- The balance may go negative by one interval on stop — accepted; a new start is checked by `validateUserBalanceForStart`.
- *Trade-off:* PHP `float` instead of BCMath/integer cents; compensated by `decimal(36, 2)` in the DB and rounding. [hypothesis]

<a id="validation"></a>
## Validation

```php
class StoreChargingSessionRequest extends FormRequest
{
    protected $stopOnFirstFailure = true;

    public function rules(): array
    {
        return [
            'connector_id' => ['bail', 'required', 'integer', 'min:1', 'max:999999999999999'],
        ];
    }
}
```

- Only `rules()`. No `authorize()` — access is checked by middleware (`auth:sanctum`, `role:admin`) and by the service (ownership).
- `rules()` is a **pure function that never reads the request**: the Swagger generator creates the FormRequest without a
  constructor and calls `rules()` outside HTTP (see [architecture.md](architecture.md#swagger)). No `$this->route('id')` / `$this->user()` in rules.
- **No database access in a FormRequest** — so no `exists:` / `unique:` rules; existence and uniqueness are checked by the
  service through repositories inside the transaction. **[confirmed: queries only in repositories]**
- Every rule has an explicit type token (`integer`, `numeric`, `string`, `boolean`, `array`, `image`): the docs type is built from it.
- Rules are arrays, in order: `bail` (if used) → `required`/`sometimes`/`present` → `nullable` → type → bounds.
- **Every field has bounds:** strings `min:1|max:255`, ids `min:1|max:999999999999999`, money `min:0.01|max:99999999.99`,
  coordinates `between:-90,90`, arrays `min:1|max:1000`. *Why:* protection against BIGINT overflow, garbage and oversized input (DoS).
- Formats are specific and anchored: phone `regex:/^\d{9}$/`, time `date_format:H:i`, OTP `size:6`.
- Enum: `new Enum(AppLanguage::class)`.
- Update: the same rules as Store, with `sometimes` instead of `required`.
- Nested translation arrays: `translations.*.language_code` → `required`, `distinct`, `size:2`.
- **No privileged fields** (`role`, `balance`, …) in user-facing requests — the FormRequest whitelist is the mass-assignment
  protection ([security.md](security.md#3-mass-assignment-and-privileged-fields)).

### `bail` / `$stopOnFirstFailure` policy **[confirmed]**

| Request kind | Setting | Why |
|---|---|---|
| Multi-field form (admin store/update, `UpdateSelfRequest`) | **without** `$stopOnFirstFailure` and **without** `bail` | The frontend shows all errors at once; the user does not fix them one by one |
| Filters, query parameters, service endpoints, 1–2-field actions (`send-code`, `verify-phone`, `charging-sessions`, `ocpp/message`) | `protected $stopOnFirstFailure = true;` + `'bail'` first in every rule | There is nothing to show but the first error; the remaining rules are not evaluated for nothing |

(Current code has `$stopOnFirstFailure = true` in every request — for forms this is legacy.)

## Mapping input ↔ entity ↔ output

```
FormRequest::validated() : array  ──►  Service(array $data)  ──►  Repository::create(array $data)  ──►  Model::create($data)
                                                                                                         │
Resource::toArray() ◄── Model (select DEFAULT_SELECT_COLUMNS, casts, appends) ◄──────────────────────────┘
```

- No DTOs: `array $data` straight from `validated()` travels between layers; `$fillable` + the FormRequest whitelist protect
  against extra fields. [hypothesis]
- Input conversions (file → path, lat/lng → `POINT`) happen in the layer that knows how data is stored:
  file → `StoreImageAction` in the service (`ConnectorTypeService::create`); coordinates → `asPoint()` in the repository.
- Output only through a Resource with explicit type casts (see [error-handling.md](error-handling.md#api-contract)).

<a id="translations"></a>
## Translations (HasTranslations)

- Translatable fields are **not table columns**: they are listed in `protected array $translatable = ['name', 'address'];`
  and stored in the polymorphic table `translations (translatable_type, translatable_id, language_code, key, value)`
  (unique index on all four key columns + FULLTEXT on `value`).
- Input: `translations: [{language_code: 'ru', name: '...', address: '...'}, ...]` → `$model->syncTranslations($data)` (upsert).
- Output: a global scope uses the `select` list to eager-load `translationsByLocale` only for the requested fields;
  locale — `accept_language()` (`Accept-Language` header, fallback `config('custom.app_default_language')`).
- Full list of translations (admin) — `->with('translations')`.
- Before saving translations the service checks that every `language_code` exists
  (today `validateLanguages()` is copied into two services — **extract it into one place**: DRY).
  The check is needed by several services, so by the layer rules it is an Action, e.g.
  `App\Actions\ValidateTranslationLanguagesAction` (DI `LanguageRepository`, `handle(array $data, bool $lockForUpdate = false): void`,
  throws `LanguageNotFoundException`). **[proposal — no such name or place in the project yet]**
- Searching translations: FULLTEXT ngram (`MATCH(value) AGAINST(? IN NATURAL LANGUAGE MODE)`), candidates limited first
  (`limit(200)`), no external search engine.
