# Слой данных: БД, модели, запросы, транзакции, валидация, маппинг

## Миграции

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

- Анонимный класс `return new class extends Migration`, `down()` — `Schema::dropIfExists`.
- **Одна миграция — одна таблица**, в финальном виде. Миграции `add_x_to_y` в истории схлопывались в create-миграцию
  (например, `add_withdrawn_amount_to_charging_sessions_table` удалена 2026-08-16).
  [гипотеза: до первого стабильного релиза схема правится in-place. **После релиза так делать нельзя** —
  CI выполняет `migrate --force` на проде, а изменённая старая миграция не применится.]
- FK — `foreignId()->constrained()->cascadeOnDelete()`.
- Деньги — `decimal(36, 2)`; энергия — `unsignedBigInteger` (ватт-часы); проценты — `unsignedTinyInteger`.
- Статусы/типы — `string` (+ PHP enum), не `enum` MySQL. *Почему:* новое значение не требует ALTER. [гипотеза]
- Бизнес-сущности — `softDeletes()`; при этом уникальность учитывает удалённые
  (`ChargerRepository::existsByRemoteCode` использует `withTrashed()`).
- Составные индексы — **под реальные запросы**: порядок колонок = порядок `where` в репозитории
  (`['status', 'last_progress_updated_at']` ↔ `chunkActiveSessions`).
- Спец-возможности MySQL — прямо в миграции:
  `geometry('coordinates', subtype: 'point', srid: 4326)->spatialIndex()`,
  `DB::statement('ALTER TABLE translations ADD FULLTEXT INDEX ... WITH PARSER ngram')`.
- Таблица без `updated_at` — явно: `timestamp('created_at')->nullable()->useCurrent()` + в модели `const null UPDATED_AT = null;`.

<a id="model"></a>
## Модели

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

Обязательный набор для новой модели:
1. `$fillable` — явный список (никакого `$guarded = []`).
2. **`public const array DEFAULT_SELECT_COLUMNS`** — колонки, которые нужны клиенту.
   Для моделей с `HasTranslations` сюда входят и переводимые поля (`'name'`, `'address'`): трейт по этому списку
   решает, какие переводы подгружать.
3. `$hidden` — `created_at`, `updated_at`, `deleted_at` и служебные поля (`remote_code`, `meter_start`, `active_session_exists`).
4. `casts()` с `#[\Override]`: числа → `integer`/`float`, флаги → `boolean`, даты → `datetime`, статус → enum.
5. `@property` в PHPDoc для appends, кастов и связей (Larastan level 5).

Дополнительно:
- Вычисляемые поля — `Attribute::make(get: ...)` + `$appends` (`Connector::status`, `Charger::is_online`).
- Нормализация на запись — `Attribute::make(set: ...)` (`ChargingSession::soc` зажимает 0..100).
- Доменная математика — `public static` методы модели (`calculateCost`, `calculateIntervalWatts`), чтобы одна формула
  использовалась в сервисе, Action и accessor'е. **Не** дублировать формулу в сервисе.
- Бизнес-константы — `public const int` (`MAX_ACTIVE_SESSIONS_PER_USER = 100`, `PROGRESS_INTERVAL_SECONDS = 60`).
- Обогащение без запроса — метод, возвращающий `$this`
  (`User::withUnreadNotifications(bool)` → `setAttribute` + `setVisible`).
- Scope — `scopeXxx(Builder $query): void` + `@method static Builder xxx()` в PHPDoc (`Point::scopeWithPrice`).

## Глобальные настройки (AppServiceProvider)

```php
Model::preventLazyLoading(! app()->isProduction());
JsonResource::withoutWrapping();
app()->setLocale(accept_language());
```

- Ленивая загрузка в dev/test — исключение. *Почему:* N+1 обнаруживается при разработке, а не в проде.
  *Компромисс:* в проде ленивая загрузка молча работает — поэтому тесты обязаны проходить вне `production`.

<a id="repository"></a>
## Запросы в репозиториях

### Явный select

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

- Каждый запрос — `->select(Model::DEFAULT_SELECT_COLUMNS)`; каждая связь — через замыкание с `select`.
  **[подтверждено: «до ресурса лучше по select, чтобы уже было оптимизировано»]**
- Флаги существования связи — `withExists('activeSession')` вместо загрузки связи.
- Обратную связь, которая уже есть в памяти, проставлять через `setRelation` вместо повторного запроса
  (`$connector->setRelation('charger', $charger)` в `getAllByPointId`).

### Флаг блокировки

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

Каждый метод чтения, который может участвовать в сценарии «проверил → записал», получает параметр
`bool $lockForUpdate = false`. Решение о блокировке принимает **сервис**.

### Create / Update / Delete

- `create(array $data): Model` — `Model::create($data)`. Если нужна связанная запись (переводы) —
  своя `transaction()`, внутри `save()` + `syncTranslations($data)`, затем перечитать через `findById($id, true)`.
- `update(int $id, array $data): ?Model` — найти (с `lockForUpdate`, если в той же функции есть запись),
  `null`, если нет; `update($data)`; **вернуть перечитанную модель** через `findById` (с правильным select и связями).
- `softDelete(int $id): bool` — `Model::where('id', $id)->delete()` (возвращает число строк → bool).
- Транзакция в репозитории — только если **сам метод** делает несколько записей (`save` + `syncTranslations`,
  `lock` + `update` + перечитывание). Внутри сервисной транзакции это превращается в savepoint.
  [гипотеза: репозиторный метод атомарен сам по себе, даже если его вызовут вне сервиса]

### Raw SQL

Только в репозитории, всегда через биндинги или `DB::getPdo()->quote()`:

```php
// app/Repositories/PointRepository.php::getInRadius
->whereRaw('MBRContains(ST_GeomFromText(?, 4326), coordinates)', [$bbox])          // дешёвый префильтр по spatial-индексу
->whereRaw("ST_Distance_Sphere(coordinates, ST_GeomFromText(CONCAT('POINT(', ?, ' ', ?, ')'), 4326)) <= ?",
    [$latitude, $longitude, $radius])                                                // точная проверка
->limit(1000)
```

Фрагменты, которые повторяются в нескольких методах, — private-метод репозитория
(`asLatitudeAndLongitude()`, `asPoint()`).

### Bulk

- Массовое обновление — `Model::upsert($rows, ['id'], ['col1', 'col2'])` (`upsertProgress`, `upsertBalances`).
- Массовая вставка — `Model::insert($rows)` с ручными `created_at`/`updated_at` и enum'ами через `->value`
  (bulk не применяет касты и не заполняет timestamps).
- Обход больших выборок — `chunkById($count, $callback)`.

## Транзакции и блокировки

```php
// app/Services/ChargerService.php::create
return transaction(function () use ($data) {
    if (! $this->pointRepository->existsById($data['point_id'], true)) {
        throw new PointNotFoundException;
    }

    if ($this->chargerRepository->countByPointId($data['point_id'], true) >= 100) {   // legacy: должна быть константа
        throw new ChargerLimitReachedException;
    }

    if ($this->chargerRepository->existsByRemoteCode($data['remote_code'], true)) {
        throw new ChargerAlreadyExistsException;
    }

    return $this->chargerRepository->create($data);
});
```

- Хелпер **`transaction(Closure)`** из `app/Support/Helpers/helpers.php` (обёртка над `DB::transaction`),
  а не `DB::transaction` в сервисах. *Почему:* сервис не импортирует фасад `DB`. [гипотеза]
- Транзакцию открывает **сервис** (или бизнес-Action) — на весь сценарий.
- Внутри транзакции **все** проверки, от которых зависит запись, — с `lockForUpdate`:
  родитель существует, лимит не превышен, уникальность, баланс, коннектор свободен.
- Порядок блокировок — сначала «общие» ресурсы (коннектор), потом пользователь
  (`ChargingSessionService::create`: connector → active sessions → user). Держать одинаковый порядок
  во всех сценариях. [гипотеза: профилактика дедлоков]
- Сетевые вызовы внутри транзакции — только через `defer()` (выполнятся после ответа),
  кроме случаев, когда от ответа зависит коммит (`remoteStartTransaction`: при `false` → `OCPPCommandFailedException`
  → откат списания).

## Деньги

- Списание — **авансом за один интервал** (`getIntervalCost`), при остановке — пересчёт и возврат разницы
  (`StopChargingSessionAction`: `refundAmount = intervalCost - exactCost`).
- Каждое промежуточное значение — `round(..., 2)`.
- Каждое списание фиксируется записью в `transactions` (`TransactionType::WITHDRAWAL`, `TransactionSubType::CHARGING`),
  пополнение — `REPLENISHMENT`.
- Баланс может уйти в минус на один интервал при остановке — это допустимо; новый старт проверяется
  `validateUserBalanceForStart`.
- *Компромисс:* `float` в PHP вместо BCMath/целых копеек; компенсируется `decimal(36, 2)` в БД и округлением. [гипотеза]

<a id="validation"></a>
## Валидация

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

- Только `rules()`. `authorize()` нет — доступ проверяют middleware (`auth:sanctum`, `role:admin`)
  и сервис (ownership).
- `rules()` — **чистая функция без обращения к запросу**: генератор Swagger создаёт FormRequest без конструктора
  и вызывает `rules()` вне HTTP (см. [architecture.md](architecture.md#swagger)). Никаких `$this->route('id')` / `$this->user()` в правилах.
- В каждом правиле — явный токен типа (`integer`, `numeric`, `string`, `boolean`, `array`, `image`): по нему строится тип в документации.
- Правила — массивом, порядок: `bail` (если нужен) → `required`/`sometimes`/`present` → `nullable` → тип → границы.
- **У каждого поля границы:** строки `min:1|max:255`, id `min:1|max:999999999999999`, деньги `min:0.01|max:99999999.99`,
  координаты `between:-90,90`, массивы `min:1|max:1000`. [гипотеза: защита от переполнения BIGINT и мусорного ввода]
- Форматы — точечно: телефон `regex:/^\d{9}$/`, время `date_format:H:i`, код OTP `size:6`.
- Enum — `new Enum(AppLanguage::class)`.
- Update — те же правила, что и Store, но `sometimes` вместо `required`.
- Вложенные массивы переводов: `translations.*.language_code` → `required`, `distinct`, `size:2`.
- **Существование FK не проверяется правилом `exists:`** — это делает сервис внутри транзакции.

### Политика `bail` / `$stopOnFirstFailure` **[подтверждено]**

| Тип запроса | Настройка | Почему |
|---|---|---|
| Многополевая форма (admin store/update, `UpdateSelfRequest`) | **без** `$stopOnFirstFailure` и **без** `bail` | Фронт показывает все ошибки сразу; пользователю не приходится исправлять их по одной |
| Фильтры, query-параметры, служебные эндпоинты, действия с 1–2 полями (`send-code`, `verify-phone`, `charging-sessions`, `ocpp/message`) | `protected $stopOnFirstFailure = true;` + `'bail'` первым в каждом правиле | Показывать нечего, кроме первой ошибки; не проверяем остальные правила зря |

(В текущем коде `$stopOnFirstFailure = true` стоит во всех запросах — для форм это legacy.)

## Маппинг вход ↔ сущность ↔ выход

```
FormRequest::validated() : array  ──►  Service(array $data)  ──►  Repository::create(array $data)  ──►  Model::create($data)
                                                                                                         │
Resource::toArray() ◄── Model (select DEFAULT_SELECT_COLUMNS, casts, appends) ◄──────────────────────────┘
```

- DTO нет: между слоями передаётся `array $data` прямо из `validated()`; от лишних полей защищает `$fillable`. [гипотеза]
- Преобразования входа (файл → путь, lat/lng → `POINT`) делает слой, который знает, как хранить данные:
  файл → `StoreImageAction` в сервисе (`ConnectorTypeService::create`); координаты → `asPoint()` в репозитории.
- Выход — только Resource с явными приведениями типов (см. [error-handling.md](error-handling.md#api-contract)).

<a id="translations"></a>
## Переводы (HasTranslations)

- Переводимые поля **не являются колонками таблицы**: они перечислены в `protected array $translatable = ['name', 'address'];`
  и хранятся в полиморфной таблице `translations (translatable_type, translatable_id, language_code, key, value)`.
- Вход: `translations: [{language_code: 'ru', name: '...', address: '...'}, ...]` → `$model->syncTranslations($data)` (upsert).
- Выход: global scope по списку `select` подгружает `translationsByLocale` только для запрошенных полей;
  локаль — `accept_language()` (заголовок `Accept-Language`, fallback — `config('custom.app_default_language')`).
- Полный список переводов (для админки) — `->with('translations')`.
- Перед сохранением переводов сервис проверяет, что все `language_code` существуют
  (сейчас это `validateLanguages()`, скопированный в два сервиса, — **вынести в одно место**: DRY).
  Проверка нужна нескольким сервисам, поэтому по правилу слоёв это Action, например
  `App\Actions\ValidateTranslationLanguagesAction` (DI `LanguageRepository`, `handle(array $data, bool $lockForUpdate = false): void`,
  бросает `LanguageNotFoundException`). **[предложение — имени и места в проекте пока нет]**
- Поиск по переводам — FULLTEXT ngram (`MATCH(value) AGAINST(? IN NATURAL LANGUAGE MODE)`), без внешнего поискового движка.
