# Tests

> **Project state:** there are no tests yet: no `tests/`, no `phpunit.xml`, `phpunit/phpunit` is not installed.
> Factories and seeders exist, but they are designed to fill a dev database.
> The author asked to add tests and choose the best approach. **[confirmed]**
> Below is the chosen approach; the initial setup is done only when the author explicitly asks for it.

## The choice: PHPUnit + Feature-first + MySQL

| Decision | Why | What we give up |
|---|---|---|
| **PHPUnit** (not Pest) | Native to Laravel (`php artisan test`, the `composer test` script already exists). Tests are classes like the rest of the code: types, `readonly` surroundings; Larastan analyses them without plugins | The terse `expect()` syntax of Pest |
| **Feature tests for endpoints** — the main kind | All the value is in the chain FormRequest → Service → Repository → Resource → exception mapping. An isolated service unit test would need every repository mocked (there are no interfaces) and would test mocks, not behaviour | Speed of isolated unit tests |
| **Unit tests** only for pure domain math of models and enum rules | `ChargingSession::calculateCost`, `calculateIntervalWatts`, `getEstimatedWattsDelivered`, `normalizeSoc`, enum methods like `isPluggedIn()`: money formulas without a DB | — |
| **MySQL test DB**, not SQLite | Migrations use `geometry … srid 4326`, `spatialIndex`, `FULLTEXT … WITH PARSER ngram`; queries use `ST_Distance_Sphere`, `MBRContains`, `MATCH … AGAINST`. SQLite cannot run them | In-memory SQLite speed |
| `Http::fake()` + `Http::preventStrayRequests()` | All external systems (WS gateway, FCM, SMS) go through `Http`; no test may reach the network | — |

## Initial setup (only when the author asks)

1. `composer require --dev phpunit/phpunit` (Laravel 13 supports `^11.5.50 || ^12.5.8 || ^13.0.3`).
2. In `composer.json` → `autoload-dev.psr-4`: `"Tests\\": "tests/"`, then `composer dump-autoload`.
3. Create the `electro_test` database (same MySQL, separate schema).
4. `phpunit.xml` from [templates/phpunit.xml.stub](../templates/phpunit.xml.stub).
5. `tests/TestCase.php` from [templates/test-case.stub](../templates/test-case.stub).
6. Run: `php artisan test` (or `composer test`).

## Structure and naming

```
tests/
├── TestCase.php
├── Feature/
│   ├── ChargingSession/
│   │   ├── StoreChargingSessionTest.php     # POST /api/charging-sessions
│   │   ├── StopChargingSessionTest.php      # POST /api/charging-sessions/{id}/stop
│   │   └── AutoUpdateProgressTest.php       # cron scenario (service call, no HTTP)
│   ├── Charger/
│   │   ├── StoreChargerTest.php
│   │   └── ...
│   └── OCPP/
│       └── HandleMessageTest.php
└── Unit/
    ├── Models/
    │   └── ChargingSessionTest.php
    └── Enums/
        └── OCPPConnectorStatusTest.php
```

- Feature folders mirror `app/Http/Requests/{Entity}`; the file is `{Verb}{Entity}Test` (like the FormRequest, with a `Test` suffix).
- Method name: `test_{what}_{result}_when_{condition}` in snake_case:
  `test_store_returns_409_when_connector_is_busy`, `test_stop_refunds_interval_cost_when_session_is_pending`.
- One test = one behaviour. Arrange / Act / Assert are separated by blank lines, no comments.

## What to test (by priority)

1. **Money flows**: starting and stopping a session, `top-up`, `autoUpdateProgress` — balance after the operation, the
   `transactions` record, the session status. Assert **exact amounts** (`assertSame(98.5, $user->fresh()->balance)`).
   Also assert that the denormalized balance agrees with the ledger after the flow.
2. **Every domain exception of the service**: status + `message` from `lang` (status table in
   [error-handling.md](error-handling.md)). A new exception without a test is an open checklist item.
3. **Security:** no token → 401; non-admin on an admin route → 403; another user's entity → 404 (ownership);
   privileged fields in a user request do not change anything (`PATCH /user` with `role`/`balance`);
   shared-secret endpoints reject a missing/wrong secret and an empty configured secret.
4. **Response contract:** `assertExactJson`/`assertJsonStructure` + types of key fields (the Resource is the mobile app's contract);
   sensitive fields (`remote_code`, timestamps) are absent.
5. **Validation:** bounds (`min`/`max`), required fields; for forms — **all** errors are returned at once,
   for the rest — exactly one (`stopOnFirstFailure`).
6. **OCPP handlers:** `StartTransaction`/`StopTransaction`/`StatusNotification` move the session to the right status;
   unknown statuses/actions are handled explicitly.
7. **Side effects sent:** `Http::assertSent(fn (Request $r) => str_ends_with($r->url(), '/broadcast') && $r['event'] === 'ActiveSessions')`.
8. **No N+1 on list endpoints:** `preventLazyLoading` already throws in tests; for extra safety count queries with `DB::listen`.

**Do not test:** framework behaviour (that `required` works), trivial proxy methods, Resources separately from the endpoint,
real external APIs. PHPUnit cannot reproduce `lockForUpdate` races — their correctness is ensured by review against the rule
"check → write inside a transaction with a lock". Query plans are verified with `EXPLAIN` on realistic data, not in tests
([performance.md](performance.md#queries)).

## Environment and fakes

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

- **`defer()`**: in HTTP tests deferred callbacks run on `terminate` (Laravel's test client calls it), so `Http::assertSent`
  works after `postJson`. When calling a service directly without HTTP (`autoUpdateProgress`), call `$this->withoutDefer()`
  so callbacks run immediately.
- **FCM:** users from factories have no `fcm_token` → no push. To test a push — `$this->mock(FirebasePushService::class)`
  (the `FirebasePush` facade resolves this class).
- **SMS:** `Http::fake([config('custom.sms.server').'*' => Http::response()])` + `Http::assertSent(...)`.
- **Time:** `$this->travelTo(now()->startOfMinute())`, `$this->travel(2)->minutes()` — for charging intervals,
  "a minute has passed" (`PROGRESS_INTERVAL_SECONDS`) and the OTP TTL.
- **Authentication:** `Sanctum::actingAs($user)`; admin — `User::factory()->create(['role' => Role::ADMIN->value])`.
- **Locale:** compare error messages with `__('all.connector_busy')`, never a literal string. The `Accept-Language` header in a
  test **does not change** the language of error messages: `app()->setLocale(accept_language())` is called once in
  `AppServiceProvider::boot()`, before the test request (translations of data through `accept_language()` read the header per
  request and do work). Testing the error language becomes possible once the locale is set per request (see project improvements).
- **FULLTEXT search** (`points/search`): InnoDB updates the FULLTEXT index only on commit, while `RefreshDatabase` keeps the test
  inside a transaction → use `DatabaseTruncation` instead of `RefreshDatabase` for search tests.
- **Factories:** current factories pick random existing rows (`User::inRandomOrder()->first()`) — handy for seeding, but
  non-deterministic in tests. In tests always pass relations explicitly:
  `ChargingSession::factory()->for($user)->for($connector)->create([...])`, or add factory states.
  Mind the fallbacks in current factories: `ConnectorFactory` uses `charger_id`/`connector_type_id = 1` when no rows exist and
  generates `number` as a string `Connector-##` — set these fields explicitly in tests.

## Reference

Full example with explanations — [examples.md → "Feature test"](examples.md#example-test);
templates — [templates/feature-test.stub](../templates/feature-test.stub), [templates/unit-test.stub](../templates/unit-test.stub).

## Before committing

```bash
php vendor/bin/pint
php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon
php artisan test
```

(Once tests exist, it makes sense to add `tests` to the phpstan paths.)
