---
name: foziluff-backend-style
description: Use when writing, refactoring or reviewing backend code in foziluff's style on PHP 8.3 / Laravel 13 (Octane + Swoole, Sanctum, MySQL) — Controller → Service → dumb Repository layers, domain exceptions, FormRequest, Resource, enums instead of raw strings, transactions with lockForUpdate, security in every layer.
---

# foziluff backend style

This skill describes not only *how* the code is written but *why*.
Source: the real project `electrogo/backend` (EV charging: points, chargers, connectors, charging sessions, balance,
OCPP gateway).

Markers used in the text:
- **[confirmed]** — the author confirmed the decision personally;
- **[hypothesis]** — the reason is inferred from the code, not confirmed by the author;
- **[proposal]** — recommended target that does not exist in the project yet;
- **legacy** — this is how the project currently does it; do not write new code this way.

---

## Philosophy

1. **Every layer has exactly one job.** **[confirmed]**
   - **Controller — HTTP layer only:** accept the request, pass the data on, return a Resource.
   - **Service — all business logic:** "is it allowed", "how much to charge", "whom to notify and with what", "in which order".
   - **Repository — as dumb as possible, works with data only:** fetch / write / count / check existence of exactly what was
     asked. No business `if`, no language choice, no text building, no business sorting.
   *Why:* business rules live in one place and can be called from HTTP, cron and the OCPP handler alike
   (`StopChargingSessionAction` is called from all three), and a repository can be rewritten (another query, another index)
   without changing behaviour.

2. **Fetch only what you return.** Explicit `select(Model::DEFAULT_SELECT_COLUMNS)`, eager-loads with selected columns,
   `preventLazyLoading` outside production, and only then a Resource. **[confirmed]**
   *Why:* optimisation is built into the query when it is written, not later; N+1 is caught in dev.
   Columns that are never selected can never leak.

3. **Integrity through locks, not hope.** Every "check → act" scenario (limits, uniqueness, balance, connector occupancy)
   runs in `transaction()` with `lockForUpdate`.
   *Why:* money and shared resources (a connector) are hit by concurrent requests. Cost: locks and slightly longer transactions.

4. **An error is a type.** Every business error is its own empty exception class; HTTP status and translated text are
   defined in one `match` in `bootstrap/app.php`.
   *Why:* the service knows nothing about HTTP, and response codes and texts are managed centrally. [hypothesis]

5. **Minimum infrastructure — maximum from the project itself.** Before adding Redis / queues / Elastic, solve it with the
   stack: MySQL spatial + FULLTEXT ngram, `defer()` instead of queues, bulk `upsert`, `chunkById`, the scheduler. **[confirmed]**
   *Trade-off:* `defer()` loses a task if the worker dies; FULLTEXT search has a quality ceiling.

6. **One source of truth for every value and rule.** **[confirmed]**
   - **DRY always:** identical code in two places is extracted immediately
     (legacy copy of `validateLanguages()` in `PointService` and `CompanyService` — do not do this).
   - **No raw strings in logic:** not `$user->role === 'admin'` but `Role::ADMIN`; not `$status === 'Charging'` but an enum.
     Any value from a fixed set (role, status, type, protocol action, event) is an enum; a repeated technical identifier is a constant.
   - **Enums live in code, not in the DB:** a `string` column, the set of values is a PHP backed enum. Never `$table->enum()`.
   - **Numbers are model constants**, no magic numbers.
   *Why:* adding a new value (role, status, payment method) = adding a `case` to the enum, with no migration/ALTER in production
   and no hunting for strings across the codebase; a typo is caught by Larastan, not by a user.

7. **Secure by default, in every corner.** **[confirmed]** Every layer carries its own part of the defence:
   route groups + throttle, FormRequest with a type and bounds on every field and no privileged fields, ownership checks
   in the service (foreign id → 404), only `validated()` data, bindings in SQL, a Resource with an explicit field list,
   secrets only via `config()` and compared with `hash_equals`, fail closed, no dev shortcuts, nothing secret in git or logs.
   *Why:* a mobile client, hardware payloads and the gateway are untrusted inputs, and the asset is money.
   No layer relies on another layer being the only defence. → [references/security.md](references/security.md)

---

## Architecture in one paragraph

An HTTP request enters the flat `routes/api.php` (groups `auth:sanctum` → `role:admin`, plus a separate `ocpp` prefix with
`OCPPMiddleware`). The controller takes data from the FormRequest (`validated()`), gets `auth()->user()`/`auth()->id()` itself
and passes them **as explicit arguments** to a `readonly` service. The service opens `transaction()`, checks existence and
limits through repositories with locks, throws a domain exception on violation, and otherwise writes through a repository.
Side effects (FCM, SMS, WebSocket pushes, charger commands) go through "gateway repositories" into `defer()` and run after
the response. The controller wraps the result in a Resource. The global handler in `bootstrap/app.php` turns exceptions into JSON.

```
routes/api.php ─► Middleware (auth:sanctum, throttle, role:admin, OCPPMiddleware)
      │
      ▼
Controller ── FormRequest::validated()  (shape, types and bounds; no DB access)
      │        auth()->user() / auth()->id()  → passed as arguments
      ▼
Service (readonly) ── transaction(), business checks, ownership, throw XxxException
      │   └──► Action (readonly, ::handle) — an operation shared by several services
      ▼
Repository (readonly, dumb) ── Eloquent / raw SQL with bindings / gateways (HTTP→WS, FCM, SMS) via defer()
      ▼
Model ── fillable, DEFAULT_SELECT_COLUMNS, $hidden, casts() (enums), accessors, domain math (static)

Output: Model ─► Resource (Http/Resources/...) ─► JSON without a wrapper
Errors: XxxException ─► bootstrap/app.php match ─► {"message": __('all.key')} + status
```

Details: [references/architecture.md](references/architecture.md).

---

## Main rules

### Layers
- Controller: one line per method — call the service and return a Resource. No `if`, no queries, no Eloquent.
  → [architecture.md](references/architecture.md#controller)
- The service knows nothing about HTTP: no `request()`, `auth()`, `abort()`, `response()`. The user comes in as an argument.
  → [architecture.md](references/architecture.md#service)
- Eloquent and `DB::` are used **only** in repositories (and migrations/seeders). Raw SQL only in repositories.
  → [data-layer.md](references/data-layer.md#repository)
- External systems (WS gateway, Firebase, SMS) are hidden behind a gateway repository; network calls go through `defer()`.
  → [architecture.md](references/architecture.md#repository-gateway)
- An Action is for an operation called by ≥2 services/entry points. The method is always `handle`. **[confirmed]**
  → [conventions.md](references/conventions.md#actions)
- DI only via constructor promotion in a `readonly class`; properties are `private` **without** a repeated `readonly`. **[confirmed]**

### Security
- Access is declared by route groups; authentication and expensive public routes get `throttle`.
- The acting user comes only from `auth()` in the controller; the service checks ownership and answers 404 for foreign ids.
- Services receive only `validated()`; user-facing FormRequests never contain privileged fields (`role`, `balance`).
- Secrets: only via `config()`, compared with `hash_equals`, fail closed when empty, never in git, URLs or logs.
- No dev routes, OTP bypasses or debug shortcuts in code — CI deploys `main` as is.
  → [security.md](references/security.md)

### Documentation (Swagger is generated from source)
- CI runs `php artisan swagger`: the generator reads **code**, not annotations. Therefore: FormRequest as a parameter type,
  `rules()` without access to the request/route/user, an explicit type token in every rule, the service called as
  `$this->xxxService->method(...)`, the mapping line `$e instanceof X => [Response::HTTP_*, ...]` on one line.
  → [architecture.md](references/architecture.md#swagger)

### Data
- **The repository is as dumb as possible** **[confirmed]**: one method = one data operation. Allowed — select, where,
  join, eager-load, lock, create/update/delete, upsert, chunk, pagination. Forbidden — business conditions, choosing a
  recipient/language, building notification texts (`__()`), business sorting, side rules ("null someone else's fcm_token"),
  hard-coded data. If a domain `if` appears in a repository method, it moves to the service.
  (legacy: `NotificationRepository::send`/`sendChargingStoppedNotification`, `UserRepository::update`, `ChargerRepository::getAllByPointId`, `PaymentRepository`.)
  → [data-layer.md](references/data-layer.md#repository)
- A repository returns `?Model` / `bool` / a collection / a paginator and **never throws** domain exceptions.
  The service turns `null`/`false` into `XxxNotFoundException`.
- The service checks existence of related entities via `existsById($id, true)` inside `transaction()`,
  not with an `exists:` rule in the FormRequest. [hypothesis: race protection + a domain error text]
- Every read has an explicit `select(...DEFAULT_SELECT_COLUMNS)`; every eager-load uses a closure that selects columns too.
- Money: `decimal(36, 2)` in the DB, `float` cast in the model, `round(..., 2)` after **every** arithmetic step.
- Limits and business constants are typed model constants (`public const int MAX_COUNT = 200;`), no magic numbers.
  (legacy: `>= 100` in `ChargerService::create`)
- Mass updates: `chunkById` + `upsert`; if a batch fails, fall back to one-by-one processing.
  → [data-layer.md](references/data-layer.md)

### Validation
- One FormRequest per action: `Requests/{Entity}/{Verb}{Entity}Request`. Rules as arrays, every number/string has
  explicit `min`/`max`, enums via `new Enum(X::class)`.
- **Stop policy** **[confirmed]**:
  - multi-field **forms** (admin store/update, profile editing) — **without** `$stopOnFirstFailure` and **without**
    `bail`: the frontend gets all errors at once, and the user does not fix them one by one;
  - everything else (filters, query parameters, one/two-field actions such as `send-code`, `verify-phone`,
    starting a session) — `protected $stopOnFirstFailure = true;` + `'bail'` first in every rule.
  → [data-layer.md](references/data-layer.md#validation)

### API responses
- **Always a Resource**, including admin endpoints and empty responses (`EmptyResource::make()`). **[confirmed]**
  `response()->json($model)` in existing controllers is legacy.
- `JsonResource::withoutWrapping()` — an object is returned without `data`; lists — `{data, next_cursor}` (`CursorPaginationResource`)
  or `{data, next_page_url}` (`LengthAwarePaginationResource` on top of `simplePaginate`).
- Default pagination — cursor, 20 per page.
  → [error-handling.md](references/error-handling.md#api-contract)

### Errors and logs
- A new business error = class `App\Exceptions\XxxException extends RuntimeException {}` + a line in the `match`
  in `bootstrap/app.php` + a key in `lang/{ru,tg,en}/all.php`. All three steps are mandatory.
- Money flows and charging sessions are logged to the `charging` channel with context (`user_id`, `session_id`, `exception`),
  then the exception is rethrown (log-and-rethrow). [hypothesis] Logs never contain phones, OTPs, tokens.
  → [error-handling.md](references/error-handling.md)

### Style
- `readonly class`, constructor promotion, types on all parameters and returns, typed `const int|array`,
  `casts()` as a method with `#[\Override]`, early return, `throw new XxxException;` without parentheses.
- **No raw string values in code** **[confirmed]**: role, status, type, OCPP action/status/measurand, WS event,
  sub_type — only via enums (`Role::ADMIN->value`, `ChargingSessionStatus::CHARGING`). Columns with a fixed set of values
  are cast to an enum in the model (`'role' => Role::class`). Repeated technical strings (log channel `charging`,
  disk `public`) — a constant or an enum. → [conventions.md](references/conventions.md#enum)
- Enums are stored in the DB as `string`, **not** `$table->enum()`: a new value needs no migration. **[confirmed]**
- Enum comparison by `->value`: `$session->status->value === ChargingSessionStatus::CHARGING->value`. **[confirmed]**
- Almost no comments; PHPDoc only for `@property`, `@mixin` and generics (`Collection<int, User>`) for Larastan.
  → [conventions.md](references/conventions.md)

### Tests
- PHPUnit; Feature tests for every endpoint (happy path + every service exception + access/ownership + validation bounds),
  Unit tests for pure domain math of models. MySQL test DB (spatial/FULLTEXT do not work in SQLite).
  `Http::fake()` + `Http::preventStrayRequests()`.
  → [testing.md](references/testing.md)

### Git
- A commit = one logical change with a meaningful message (not `changes`). **[confirmed]**
- **No traces of AI in git:** no `Co-Authored-By: Claude…`, no "Generated with…", no mention of an assistant
  in messages, branches, PRs or code. **[confirmed]**
  → [conventions.md](references/conventions.md#git)

---

## Checklist before handing in code

**Layers**
- [ ] Controller: only `validated()`/`integer()`/`float()` → service → `Resource::make()/collection()`.
- [ ] The service does not call `auth()`, `request()`, `abort()`, `response()`; the user/id came in as an argument.
- [ ] No Eloquent/`DB::` outside `app/Repositories` (and migrations/seeders).
- [ ] No business `if`, text building, language/recipient choice or business sorting in the repository — data only.
- [ ] Repeated logic is extracted into a private method/Action, not copied (DRY).

**Data**
- [ ] Every query has `select(...DEFAULT_SELECT_COLUMNS)`; relations via `with([... fn (Relation $q) => $q->select(...)])`.
- [ ] New model: `$fillable`, `DEFAULT_SELECT_COLUMNS`, `$hidden` (timestamps, `deleted_at`, internal/sensitive fields), `casts()`.
- [ ] "Check → write" runs inside `transaction()` with `lockForUpdate`.
- [ ] Money rounded with `round(..., 2)`; money columns `decimal(36, 2)`.
- [ ] Limits are model constants.
- [ ] Not a single raw string value (`'admin'`, `'Charging'`, `'Accepted'`, `'UserUpdated'`) — only enums/constants;
      a new column with a fixed set of values = `string` + enum + model cast.
- [ ] Migration: `foreignId()->constrained()->cascadeOnDelete()`, `softDeletes()` for business entities, indexes matching real `where`s.

**Security**
- [ ] Right route group; `throttle` on authentication and expensive public routes.
- [ ] Ownership checked in the service (foreign id → 404) and covered by a test.
- [ ] Only `validated()` reaches the service; no privileged fields in user-facing FormRequests.
- [ ] No user input in raw SQL, column names or outgoing URLs.
- [ ] Secrets via `config()`, compared with `hash_equals`, fail closed; nothing secret in git, logs or URLs.
- [ ] No dev routes, bypasses or debug output in the diff. Full list: [security.md](references/security.md#security-checklist-every-change).

**Contract**
- [ ] FormRequest follows the `bail`/`stopOnFirstFailure` policy (form or not); every field has bounds.
- [ ] Response is a Resource with explicit `(int)/(float)/(bool)`; empty response — `EmptyResource::make()`.
- [ ] Route: kebab-case plural, `->whereNumber('id')`, right group (public / `auth:sanctum` / `role:admin`).
- [ ] New exceptions: class + `match` line in `bootstrap/app.php` + key in `lang/ru`, `lang/tg`, `lang/en`.
- [ ] Swagger: `rules()` does not read the request, every rule has a type, the `match` line has the form `$e instanceof X => [Response::HTTP_*, …]`.

**Quality**
- [ ] `php vendor/bin/pint` (or `php vendor/bin/pint <path>`) — no changes.
- [ ] `php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon` — 0 errors.
- [ ] `php artisan test` — green; the new endpoint has a Feature test.
- [ ] The commit is logical and has no traces of AI.

---

## Never do

| ❌ Forbidden | Why |
|---|---|
| Logic, `if`, queries, `Model::` in a controller | The rule becomes unavailable from cron/OCPP and gets duplicated |
| `auth()`/`request()`/`abort()` in a service | The service is called without HTTP (scheduler, OCPP messages) |
| `response()->json($model)` | The whole model leaks out; the contract changes with the schema |
| `Model::all()`, `->get()` without `select` | Extra columns and translations; contradicts "fetch only what you return" |
| Lazy loading of relations | Fails in dev (`preventLazyLoading`), N+1 in prod |
| `exists:` in a FormRequest instead of a check in the service | Check outside the transaction/lock, no domain text [hypothesis] |
| `throw new HttpException`/`abort()` for business errors | Only domain classes + the central `match` |
| An exception without a `match` line and a translation | The user gets a 500 instead of a clear answer |
| Balance/limit check without `lockForUpdate` | Race of two requests → double charge / limit exceeded |
| Money arithmetic without `round(..., 2)` | Float error accumulates every interval |
| Copying a method into two services | DRY always [confirmed] |
| New queues/Redis/Elastic "just in case" | Solve with the project's own means first [confirmed] |
| `readonly class` + `private readonly` together | Redundant [confirmed] |
| An Action with an `execute` method | The standard is `handle` [confirmed] |
| Magic numbers | A model constant with a meaningful name |
| Raw string values: `=== 'admin'`, `in_array($status, ['Charging', ...])`, `'event' => 'UserUpdated'` | Only enum/constant: a typo is not caught, a new value must be hunted across the code [confirmed] |
| `$table->enum('status', [...])` in a migration | A new value needs ALTER in production; the set of values lives in a PHP enum [confirmed] |
| Business logic in a repository (conditions, texts, `__()`, language choice, business sorting) | The repository is as dumb as possible — data only [confirmed] |
| `$request->all()`, `$guarded = []`, `forceFill($input)` | Mass assignment of `role`/`balance` [confirmed: security] |
| `user_id` for ownership taken from the request | IDOR: acting on someone else's data |
| Secrets compared with `===`, or "empty secret == empty header" | Timing attacks; open door when the env var is missing |
| Dev routes, OTP bypasses, secrets (`firebase.json`, keys) in the repository | Access leaks to production (CI deploys `main` as is) |
| Phones, OTP codes, tokens in logs | Personal data / credential leak |
| `Co-Authored-By`/"Generated with" in commits | No traces of AI in git [confirmed] |

---

## Navigating the references

| Task | Read |
|---|---|
| Where to put a new class, who depends on whom, Service vs Action vs Repository | [references/architecture.md](references/architecture.md) |
| Naming files/methods/routes/columns/translation keys, enums, formatting, git | [references/conventions.md](references/conventions.md) |
| A new business error, HTTP status, response format, pagination, logging | [references/error-handling.md](references/error-handling.md) |
| Migration, model, query, transaction, locks, validation, translations, money | [references/data-layer.md](references/data-layer.md) |
| Authentication, authorization, mass assignment, secrets, injection, abuse limits | [references/security.md](references/security.md) |
| Writing a test, setting up the test environment | [references/testing.md](references/testing.md) |
| Reference code with explanations and anti-examples | [references/examples.md](references/examples.md) |
| Creating a new CRUD/endpoint from scratch | [templates/README.md](templates/README.md) |

## Order of work on a new feature

1. Migration → enum(s) for fixed sets → model → factory → seeder.
2. Exceptions (`XxxNotFoundException` and business errors) → `match` in `bootstrap/app.php` → keys in all three `lang` files.
3. Repository (data only, `lockForUpdate` flags).
4. Service (transactions, checks, ownership, exceptions).
5. FormRequests (`bail` policy, bounds, no privileged fields) → Resource → controller → routes (right group, throttle).
6. Feature tests → `pint` → `phpstan` → `php artisan test` → security checklist → a logical commit without traces of AI.

Templates for all files: [templates/](templates/README.md).
