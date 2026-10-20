# Architecture

## Project map

```
app/
├── Actions/           # operations shared by several services/entry points (readonly, ::handle)
├── Enums/             # string-backed enums: statuses, roles, types — every fixed set of values
├── Exceptions/        # empty domain exceptions, one per business error
├── Http/
│   ├── Controllers/   # thin, one line per method
│   ├── Middleware/    # RoleMiddleware (role:admin), OCPPMiddleware (gateway shared secret)
│   ├── Requests/{Entity}/{Verb}{Entity}Request.php
│   └── Resources/Client/   # output contract + pagination wrappers
├── Models/            # Eloquent: fillable, DEFAULT_SELECT_COLUMNS, $hidden, casts(), accessors, domain math
├── Providers/AppServiceProvider.php   # preventLazyLoading, withoutWrapping, locale
├── Repositories/      # the ONLY place for DB access + gateways to external systems
├── Services/          # business logic, transactions
├── Support/Helpers/helpers.php        # global functions: transaction(), full_url(), accept_language()
└── Traits/HasTranslations.php         # polymorphic translations
bootstrap/app.php      # middleware aliases + central exception → HTTP mapping
routes/api.php         # all API routes in one file
routes/console.php     # scheduler (closures)
config/custom.php      # all project-specific env settings
lang/{ru,tg,en}/all.php
```

Code is grouped **by layer, not by feature**. `Requests/` has sub-folders per entity because one entity has many FormRequests.

*Why by layer:* [hypothesis] one project, about 13 entities, and most scenarios touch several entities at once
(a session = connector + user + transaction + notification). Grouping by feature would scatter these cross-module dependencies.
*Trade-off:* `Services/` and `Repositories/` grow linearly with the number of entities.

## Dependency direction

```
Controller ─► Service ─► Repository ─► Model
                 │            ▲
                 └─► Action ──┘
Repository ─► Repository   (allowed: MobileCommandRepository → UserRepository, ChargingSessionRepository)
Repository ─► Action       (legacy: only the technical SendWebsocketRequestAction; target — HTTP client inside a repository)
Repository ─► Resource     (only to serialize WS push payloads: MobileCommandRepository)
```

Forbidden:
- Service → Controller/Request/Resource (the service knows nothing about HTTP);
- Service → Eloquent/`DB::`/`Http::` (all data access and external calls go through repositories) **[confirmed]**;
- Repository → Service (a repository makes no decisions);
- Model → Repository/Service;
- Controller → Repository or Eloquent directly (legacy: `UserController::devToken` uses `User::` — do not do this).

Strict data-access rule (where queries and external HTTP may live) — [performance.md](performance.md#data-access).

---

<a id="controller"></a>
## Controller

**HTTP layer only** **[confirmed]**. **What it does:** takes validated data, gets the current user, calls **one** service
method and wraps the result in a Resource.

```php
// app/Http/Controllers/ChargingSessionController.php
public function store(StoreChargingSessionRequest $request): ChargingSessionResource
{
    return ChargingSessionResource::make($this->chargingSessionService->create($request->validated(), auth()->id()));
}
```

Rules:
- Extends the empty `abstract class Controller`.
- Depends on **one** service via `private readonly` in constructor promotion.
  (The controller class itself is not `readonly`, so `readonly` sits on the property — this does **not** break the
  "no double readonly" rule.)
- The return type is a concrete Resource (`PointResource`, `CursorPaginationResource`, `AnonymousResourceCollection`, `EmptyResource`).
- Scalar parameters via `$request->integer('point_id')` / `$request->float('radius')` / `$request->validated('phone')`;
  the whole payload via `$request->validated()`. Never `$request->all()`.
- `auth()->user()` / `auth()->id()` are called **only here** and passed to the service as arguments (`getById($id, auth()->id())`).
- Path parameters are typed (`int $id`, `string $code`) and the route is constrained with `->whereNumber('id')`.
- Method names: REST (`index`, `show`, `store`, `update`, `destroy`) or an action verb
  (`stop`, `topUp`, `sendCode`, `verifyPhone`, `getSelf`, `updateSelf`).

*Why:* the same rule is called from HTTP, from the scheduler (`routes/console.php` → `autoUpdateProgress`) and from the OCPP
handler. Logic in a controller cannot be called without an HTTP request.
*Trade-off:* for trivial CRUD the service often just proxies the call (`SettingService`) — this is intended.

---

<a id="service"></a>
## Service

**What it does:** all business logic — checks, limits, calculations, ownership, transactions, order of side effects.
It does not query the database and does not call external systems itself — only through repositories. **[confirmed]**

```php
readonly class ChargerService
{
    public function __construct(
        private ChargerRepository $chargerRepository,
        private PointRepository $pointRepository,
    ) {}
```

Rules:
- `readonly class`, dependencies `private` without a repeated `readonly`. **[confirmed]**
  (legacy with double readonly: `UserService`, `NotificationService`, `PaymentService`, `TransactionService`.)
- Depends on repositories of its own and **neighbouring** entities and on Actions. It does not inject other services
  (there is no Service → Service dependency in the project). [hypothesis: no cycles; shared code goes into an Action]
- Public method = a scenario: `getById`, `create`, `update`, `delete`, `cursorPaginate`, `getInRadius`, `stop`, `topUp`.
- `getById()` **throws** `XxxNotFoundException` when the repository returns `null`; the same `getById()` is reused inside
  `update()` (`ChargerService::update`, `ConnectorService::update`).
- Checks live in private methods with meaningful names:
  `getAndValidateConnectorForStart()`, `validateActiveSessionsLimit()`, `validateUserBalanceForStart()`.
- Modifying scenarios are wrapped in `transaction(function () use (...) { ... })`; inside it every check uses `lockForUpdate`
  (`existsById($id, true)`, `countByPointId($id, true)`).
- Errors are only domain exceptions (`throw new ConnectorBusyException;`).
- Ownership check answers `NotFound`, not 403:
  `if (! $session || $session->user_id !== $authUserId) throw new ChargingSessionNotFoundException;`
  [hypothesis: do not reveal that someone else's resource exists]

---

<a id="action"></a>
## Action

Used when an operation is called by **several** services or entry points.

| Action | Called by | Kind |
|---|---|---|
| `StopChargingSessionAction` | `ChargingSessionService::stop`, `ChargingSessionService::autoUpdateProgress`, `OCPPService::handleStatusNotification/handleStopTransaction` | instance with DI |
| `SendWebsocketRequestAction` | `MobileCommandRepository`, `OCPPCommandRepository` | static, technical (legacy placement: external HTTP belongs in a repository) |
| `StoreImageAction` / `DeleteImageAction` | `ConnectorTypeService` | static, technical (filesystem) |

Rules:
- `readonly class`, the only public method is **`handle`**. **[confirmed]**
  (legacy: `execute` in `StopChargingSessionAction` and `SendWebsocketRequestAction`.)
- An Action that needs repositories is an instance with DI (`public function handle(...)`).
  A pure technical utility without dependencies is `public static function handle(...)`.
- An Action follows the same data-access rule as a service: it talks to the DB and external systems only through repositories.
- A business Action (`StopChargingSessionAction`) owns its transaction, logging and side effects, because it is called from
  different contexts.

*Why not a service:* a service is tied to one entity/controller; if `OCPPService` injected `ChargingSessionService`,
services would become coupled. An Action is a neutral shared point. [hypothesis]

---

<a id="repository"></a>
## Repository (database)

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

- **As dumb as possible** **[confirmed]**: data operations only, not a single business decision
  (full "allowed / forbidden" table — [data-layer.md](data-layer.md#repository)).
- **The only place** where Eloquent, `DB::` and raw SQL appear (besides migrations/seeders). **[confirmed]**
- Every query must be served by an index and bounded — [performance.md](performance.md#queries).
- `readonly class` without an interface, injected by its concrete class.
  *Why no interfaces:* [hypothesis] one implementation, one DB; an interface would only add a file.
  *Trade-off:* tests substitute it with `$this->mock(ConcreteRepository::class)`, not a binding.
- Returns `?Model`, `bool`, `int`, `Collection`, `CursorPaginator`/`Paginator`. **Never throws** domain exceptions.

<a id="repository-gateway"></a>
## Repository as a gateway to an external system

In this project a "repository" is **any data source or sink**, not only the DB; outgoing HTTP to external systems lives
**only** here. **[confirmed]**

| Repository | What it hides |
|---|---|
| `OCPPCommandRepository` | HTTP commands to the Node gateway → chargers (`RemoteStartTransaction`, `RemoteStopTransaction`, `ChangeConfiguration`) |
| `MobileCommandRepository` | WebSocket pushes to the mobile app via the Node gateway (`ActiveSessions`, `UserUpdated`) |
| `NotificationRepository` | `notifications` table + FCM (`FirebasePush`) + SMS gateway |
| `PaymentRepository` | payment methods (a TODO stub for now) |

Gateway rules:
- A gateway is **transport only**: it receives ready data (title, text, recipient, payload) and delivers it.
  What to send, to whom and in which language is decided by the service. (legacy: `NotificationRepository` builds texts with
  `__()` and picks the translation/topic itself — move this into `NotificationService` when touching it.)
- Event names, protocol actions and response statuses are enums, not strings
  (legacy: `'UserUpdated'`, `'ActiveSessions'`, `'RemoteStartTransaction'`, `'Accepted'` as strings in `MobileCommandRepository` and `OCPPCommandRepository`).
- URL only from config, `timeout(5)`, response checked, failures logged without secrets — [security.md](security.md#8-service-to-service-and-outgoing-calls).
- Network calls inside `defer(function () use (...) { ... })` so the response is not delayed and no transaction is held open.
  Exception: when the answer is needed now (`remoteStartTransaction` returns `bool`, otherwise the session cannot be created).
- Data for a deferred call is **serialized before `defer`**
  (`$userData = UserResource::make($user)->resolve();` in `MobileCommandRepository::pushUserUpdated`):
  no lazy loading inside `defer`, a snapshot of the state is sent, and the closure holds a small array, not a model graph.
- An external failure does not break the business operation: the gateway logs it and returns `null`/`false`.

*Why `defer()` instead of queues:* the project runs on Octane/Swoole; `defer` runs after the response in the same worker,
so no queue worker is needed. Job classes (`SendFirebasePushJob`, `SendSmsJob`, `SendMobileCommandJob`, etc.) existed and
were removed on 2026-08-17 in favour of `defer`. **[confirmed by the "minimum infrastructure" principle]**
*Trade-off:* no retries or persistence — a task is lost if the worker dies.
Only if guaranteed delivery becomes necessary (payments) do we return to queues.

---

## Model

A model is the data schema and domain math, with no calls to other layers and **no queries or I/O in accessors**:
- `ChargingSession::calculateCost()`, `calculateIntervalWatts()`, `getEstimatedWattsDelivered()`, `getIntervalCost()` —
  pure calculations used by the service, the Action and the `amount` accessor;
- computed state via `Attribute` + `$appends` (`Connector::status`, `Charger::is_online`) — computed from loaded data
  instead of a stored, denormalized column;
- business constants (`MAX_ACTIVE_SESSIONS_PER_USER`, `PROGRESS_INTERVAL_SECONDS`, `MAX_RADIUS`, `MAX_COUNT`).

Details — [data-layer.md](data-layer.md#model).

---

## External perimeter

```
Mobile app ──HTTP──► Laravel API (Octane/Swoole)
     ▲                    │  POST {WS_SERVER_URL}/broadcast, /command/{remote_code}
     │ WebSocket          ▼  header Ws-Secret
     └────────── Node WS/OCPP gateway (separate repository) ◄──OCPP──► Chargers
                          │
                          └──► POST /api/ocpp/message, GET /api/ocpp/chargers/{code}  (OCPPMiddleware)
```

- Laravel holds no WebSocket connections: the Node gateway does (until 2026-08-30 it lived in this repo as `ws-server/`, `ykc-server/`).
- Incoming OCPP messages are dispatched by `OCPPService::handleMessage` via `match ($action)` to `handleXxx` handlers
  (target: match on an `OCPPAction` enum, not raw strings — [conventions.md](conventions.md#enum)).

## Background processes

```php
// routes/console.php
Schedule::call(function (ChargingSessionService $chargingSessionService) {
    $chargingSessionService->autoUpdateProgress();
})->everyMinute()
    ->name('charging:update-progress')
    ->withoutOverlapping();
```

- The scheduler is a closure that calls **a service method** (the `UpdateChargingProgressCommand` class was removed on 2026-08-15).
- `withoutOverlapping()` is mandatory for tasks that may not finish within the interval.
- Batches of `chunkById(500)`; a batch is processed in one transaction (bulk upsert), with a one-by-one fallback
  (`processSessionsBulk` → `processSessionsIndividually`). Batch accumulators are local to the chunk callback (no leaks).

<a id="swagger"></a>
## Generated documentation: why the shape of the code matters

After deploy, CI runs `php artisan swagger` (the author's package `foziluff/swagger`) → `public/api-docs.json`, `public/docs.html`.
The generator **does not use annotations**: it reads routes, reflection and **source code via regular expressions**.
So some style rules are not taste but a condition for correct documentation:

| What the generator does | Resulting rule |
|---|---|
| Finds the FormRequest by the **parameter type** of the controller method (reflection) | Input only via a FormRequest in the signature, never `$request->validate()` |
| Creates the FormRequest with `newInstanceWithoutConstructor()` and calls `rules()` | `rules()` **must not** depend on the request, route or user (`$this->route()`, `$this->user()`, `$this->input()`) — otherwise generation fails or is wrong |
| Parses rules as an **array** (or a `\|` string) and guesses the type from string tokens `integer` / `numeric` / `boolean` / `array` / `image` / `file` / `mimes` | Every rule has an explicit type token; rules are written as arrays |
| `min:`/`max:` → `minimum`/`maximum` (`minLength`/`maxLength`), `required` → required, `new Enum(X::class)` and `in:` → enum | Explicit bounds and enum rules show up in the docs |
| If the FormRequest has an `example()` method, its result becomes the request example | You may add `public function example(): array` for complex bodies (not used in the project yet) |
| Response codes: looks for `throw new X` in the controller method body **and in the body of the public service method** called as `$this->{prop}->{method}(` — **one level deep**; the status comes from `bootstrap/app.php` | The controller calls the service exactly as `$this->xxxService->method(...)` through a typed property. Exceptions thrown in private helpers, Actions or a nested `getById()` do **not** reach the docs |
| The mapping is read with the regex `$e instanceof Class => [Response::HTTP_CONST` | The `match` line keeps exactly this form, on one line, status as a `Response::HTTP_*` constant |
| `auth*` middleware → `security: bearerAuth`; `throttle*` → `429`; FormRequest → `422` | Access and limits are declared by route middleware, not by controller code |
| Tag from the controller name, summary from the method name | Meaningful controller method names (`getByCompany`, `topUp`) |

Consequence for new services [hypothesis — to confirm]: if a response code must appear in the docs, the `throw new` should
be in the body of the public method called from the controller (as in `ChargerService::create`). Checks extracted into
private methods (`ChargingSessionService::create` → `getAndValidateConnectorForStart`) remain good style, but the generator
will not see their codes.

## Decision summary

| Decision | Problem it solves | Trade-off | Status |
|---|---|---|---|
| Controller → Service → Repository | Logic reused from HTTP, cron, OCPP | Thin proxy services for CRUD | confirmed |
| DB queries and external HTTP only in repositories | One place to review indexes, columns, locks; services free of infrastructure | More repository methods | confirmed |
| Repository as dumb as possible | Behaviour changes only in services; a query can be rewritten safely | More code in services | confirmed |
| No repository interfaces | Less code with a single implementation | Tests mock the concrete class | [hypothesis] |
| Empty domain exceptions + `match` | Service knows no HTTP; texts and codes in one place | Long `match` in `bootstrap/app.php` | pattern |
| `existsById(..., lock)` in the service | Races, domain error texts, no DB access in FormRequests | An extra query | confirmed (no DB outside repos) |
| `DEFAULT_SELECT_COLUMNS` everywhere | Performance + `HasTranslations` + nothing unselected can leak | The constant must follow schema changes | confirmed |
| Indexed, bounded, keyset queries | Milliseconds at any table size | Index per access path | confirmed |
| Computed state instead of denormalized columns | No drift between copies | Computation at read time | confirmed |
| `defer()` instead of queues | No worker or Redis; response does not wait for the network | No retries | confirmed |
| MySQL spatial/FULLTEXT instead of PostGIS/Elastic | One database | Search quality ceiling | confirmed |
| Bulk upsert + one-by-one fallback | Per-minute cron over thousands of sessions | Complex code | pattern |
| Enum in code, `string` column | New value = one `case`, no migration; no raw strings | The DB does not validate values | confirmed |
| Stateless `readonly` classes | No cross-request state or leaks on Octane | No per-object caching | confirmed (no leaks) |
| Ownership → 404 | Do not reveal existence of others' data | Less informative | [hypothesis] |
| One token per user (`deleteTokens` on login) | One device — one session | Cannot log in on two devices | [hypothesis] |
