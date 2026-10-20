# Тесты

> **Состояние проекта:** тестов пока нет: нет ни `tests/`, ни `phpunit.xml`, `phpunit/phpunit` не установлен.
> Есть factories и seeders, но они рассчитаны на наполнение dev-БД.
> Автор попросил добавить тесты и выбрать лучший подход. **[подтверждено]**
> Ниже — выбранный подход; первичная настройка выполняется только по явной просьбе автора.

## Выбор: PHPUnit + Feature-first + MySQL

| Решение | Почему | От чего отказываемся |
|---|---|---|
| **PHPUnit** (не Pest) | Нативен для Laravel (`php artisan test`, скрипт `composer test` уже есть). Тесты — такие же классы, как остальной код: типы, `readonly`-окружение; Larastan анализирует их без плагинов | Лаконичность `expect()`-синтаксиса Pest |
| **Feature-тесты на эндпоинты** — основной вид | Вся ценность — в связке FormRequest → Service → Repository → Resource → маппинг исключений. Изолированный юнит-тест сервиса потребовал бы мокать каждый репозиторий (интерфейсов нет) и проверял бы моки, а не поведение | Скорость отдельных юнит-тестов |
| **Unit-тесты** — только на чистую доменную математику моделей | `ChargingSession::calculateCost`, `calculateIntervalWatts`, `getEstimatedWattsDelivered`, `normalizeSoc`: формулы денег без БД | — |
| **MySQL test-БД**, не SQLite | Миграции используют `geometry … srid 4326`, `spatialIndex`, `FULLTEXT … WITH PARSER ngram`, запросы — `ST_Distance_Sphere`, `MBRContains`, `MATCH … AGAINST`. В SQLite это не работает | Скорость in-memory SQLite |
| `Http::fake()` + `Http::preventStrayRequests()` | Все внешние системы (WS-шлюз, FCM, SMS) ходят через `Http`; ни один тест не должен уйти в сеть | — |

## Первичная настройка (выполнять только по просьбе автора)

1. `composer require --dev phpunit/phpunit` (Laravel 13 поддерживает `^11.5.50 || ^12.5.8 || ^13.0.3`).
2. В `composer.json` → `autoload-dev.psr-4`: `"Tests\\": "tests/"`, затем `composer dump-autoload`.
3. Создать БД `electro_test` (тот же MySQL, отдельная схема).
4. `phpunit.xml` из [templates/phpunit.xml.stub](../templates/phpunit.xml.stub).
5. `tests/TestCase.php` из [templates/test-case.stub](../templates/test-case.stub).
6. Запуск: `php artisan test` (или `composer test`).

## Структура и именование

```
tests/
├── TestCase.php
├── Feature/
│   ├── ChargingSession/
│   │   ├── StoreChargingSessionTest.php     # POST /api/charging-sessions
│   │   ├── StopChargingSessionTest.php      # POST /api/charging-sessions/{id}/stop
│   │   └── AutoUpdateProgressTest.php       # cron-сценарий (вызов сервиса, без HTTP)
│   ├── Charger/
│   │   ├── StoreChargerTest.php
│   │   └── ...
│   └── OCPP/
│       └── HandleMessageTest.php
└── Unit/
    └── Models/
        └── ChargingSessionTest.php
```

- Папки Feature повторяют `app/Http/Requests/{Entity}`; файл — `{Verb}{Entity}Test` (как FormRequest, только с суффиксом `Test`).
- Метод — `test_{что}_{результат}_when_{условие}` в snake_case:
  `test_store_returns_409_when_connector_is_busy`, `test_stop_refunds_interval_cost_when_session_is_pending`.
- Один тест — одно поведение. Структура Arrange / Act / Assert отделяется пустыми строками, без комментариев.

## Что тестировать (по приоритету)

1. **Денежные потоки** — создание и остановка сессии, `top-up`, `autoUpdateProgress`: баланс после операции,
   запись в `transactions`, статус сессии. Проверять **точные суммы** (`assertSame(98.5, $user->fresh()->balance)`).
2. **Каждое доменное исключение сервиса** — статус + `message` из `lang` (см. таблицу статусов в
   [error-handling.md](error-handling.md)). Новое исключение без теста = незакрытый пункт чек-листа.
3. **Доступ:** без токена → 401, не-админ на admin-маршруте → 403, чужая сущность → 404 (ownership).
4. **Контракт ответа:** `assertExactJson`/`assertJsonStructure` + типы ключевых полей (Resource — это контракт мобильного приложения).
5. **Валидация:** границы (`min`/`max`), обязательность; для форм — что возвращаются **все** ошибки сразу,
   для остальных — ровно одна (`stopOnFirstFailure`).
6. **OCPP-хендлеры:** `StartTransaction`/`StopTransaction`/`StatusNotification` переводят сессию в правильный статус.
7. **Отправленные побочные эффекты:** `Http::assertSent(fn (Request $r) => str_ends_with($r->url(), '/broadcast') && $r['event'] === 'ActiveSessions')`.

**Не тестировать:** поведение фреймворка (что `required` работает), тривиальные прокси-методы, Resource отдельно
от эндпоинта, реальные внешние API. Гонки `lockForUpdate` PHPUnit не воспроизводит — их корректность обеспечивает
ревью по правилу «проверил → записал внутри транзакции с блокировкой».

## Окружение и фейки

```php
abstract class TestCase extends BaseTestCase
{
    use RefreshDatabase;

    #[\Override]
    protected function setUp(): void
    {
        parent::setUp();

        Http::preventStrayRequests();
        Http::fake([
            '*/broadcast' => Http::response([]),
            '*/command/*' => Http::response(['status' => 'Accepted']),
        ]);
    }
}
```

- **`defer()`**: в HTTP-тестах отложенные колбэки выполняются на `terminate` (тестовый клиент Laravel его вызывает),
  поэтому `Http::assertSent` работает после `postJson`. При прямом вызове сервиса без HTTP
  (`autoUpdateProgress`) — вызвать `$this->withoutDefer()`, чтобы колбэки выполнялись сразу.
- **FCM:** в factories у пользователя нет `fcm_token` → push не отправляется. Если нужен тест push —
  `$this->mock(FirebasePushService::class)` (фасад `FirebasePush` резолвит этот класс).
- **SMS:** `Http::fake([config('custom.sms.server').'*' => Http::response()])` + `Http::assertSent(...)`.
- **Время:** `$this->travelTo(now()->startOfMinute())`, `$this->travel(2)->minutes()` — для интервалов зарядки
  и проверки «прошла минута» (`PROGRESS_INTERVAL_SECONDS`), TTL OTP.
- **Аутентификация:** `Sanctum::actingAs($user)`; админ — `User::factory()->create(['role' => Role::ADMIN->value])`.
- **Локаль:** сообщение ошибки сравнивать через `__('all.connector_busy')`, а не строкой. Заголовок `Accept-Language` в тесте
  **не меняет** язык сообщений ошибок: `app()->setLocale(accept_language())` вызывается один раз в `AppServiceProvider::boot()`,
  до тестового запроса (а переводы данных через `accept_language()` читают заголовок на каждом запросе и работают).
  Тест на язык ошибок станет возможен, когда локаль будет выставляться per-request (см. итоговый отчёт: улучшения проекта).
- **FULLTEXT-поиск** (`points/search`): InnoDB обновляет FULLTEXT-индекс только при коммите, а `RefreshDatabase`
  держит тест в транзакции → для тестов поиска использовать `DatabaseTruncation` вместо `RefreshDatabase`.
- **Factories:** текущие factories берут случайные существующие строки (`User::inRandomOrder()->first()`) — это удобно
  для сидов, но недетерминированно в тестах. В тестах всегда передавать связи явно:
  `ChargingSession::factory()->for($user)->for($connector)->create([...])` или добавить factory-states.
  Учитывать fallback'и в текущих factories: `ConnectorFactory` подставляет `charger_id`/`connector_type_id = 1`, если строк нет,
  и генерирует `number` строкой `Connector-##` — в тестах задавать эти поля явно.

## Эталон

Полный пример с разбором — [examples.md → «Feature-тест»](examples.md#example-test);
шаблоны — [templates/feature-test.stub](../templates/feature-test.stub), [templates/unit-test.stub](../templates/unit-test.stub).

## Перед коммитом

```bash
php vendor/bin/pint
php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon
php artisan test
```

(Если тесты добавлены, имеет смысл включить `tests` в список путей phpstan.)
