# Reference examples with explanations

All fragments come from the project (the path is given above each one), except the target code in example 8 and the test
in example 10, which are written by the rules of this skill. Where real code differs from the target rule it is marked
**legacy → target**.

---

## 1. A whole endpoint through all layers: `POST /api/chargers`

### Route
`routes/api.php`
```php
Route::middleware(['auth:sanctum'])->group(function (): void {
    Route::middleware('role:'.Role::ADMIN->value)->group(function (): void {
        Route::post('chargers', [ChargerController::class, 'store']);
        Route::patch('chargers/{id}', [ChargerController::class, 'update'])->whereNumber('id');
        Route::delete('chargers/{id}', [ChargerController::class, 'destroy'])->whereNumber('id');
    });
});
```
- Access is declared by a **group**, not by a check in the controller: the routes file shows at a glance that only an admin
  can create chargers. The role is an enum value, not the string `'admin'`.
- `whereNumber('id')` — a non-numeric id is cut off by the router (404) before the controller, so the `int $id` signature is safe.

### FormRequest
`app/Http/Requests/Charger/StoreChargerRequest.php`
```php
class StoreChargerRequest extends FormRequest
{
    protected $stopOnFirstFailure = true;

    public function rules(): array
    {
        return [
            'remote_code' => ['bail', 'required', 'string', 'min:1', 'max:255'],
            'point_id' => ['bail', 'required', 'integer', 'min:1', 'max:999999999999999'],
            'label' => ['bail', 'required', 'string', 'min:1', 'max:255'],
        ];
    }
}
```
- Only **shape, types and bounds** are checked. No `exists:points,id` and no `unique:chargers,remote_code` — those would be
  DB queries outside a repository and outside the lock; the service does them.
- **legacy → target:** this is a multi-field admin form, so by the validation policy **remove** `$stopOnFirstFailure` and `bail`
  so the frontend gets all errors at once.

### Controller
`app/Http/Controllers/ChargerController.php`
```php
public function store(StoreChargerRequest $request): JsonResponse
{
    return response()->json($this->chargerService->create($request->validated()));
}
```
- One line: data → service → response.
- **legacy → target:**
  ```php
  public function store(StoreChargerRequest $request): ChargerResource
  {
      return ChargerResource::make($this->chargerService->create($request->validated()));
  }
  ```

### Service
`app/Services/ChargerService.php`
```php
public function create(array $data): Charger
{
    return transaction(function () use ($data) {
        if (! $this->pointRepository->existsById($data['point_id'], true)) {
            throw new PointNotFoundException;
        }

        if ($this->chargerRepository->countByPointId($data['point_id'], true) >= 100) {
            throw new ChargerLimitReachedException;
        }

        if ($this->chargerRepository->existsByRemoteCode($data['remote_code'], true)) {
            throw new ChargerAlreadyExistsException;
        }

        return $this->chargerRepository->create($data);
    });
}
```
- Three "is it allowed" checks → one write, all inside `transaction()` with `lockForUpdate` (the second argument `true`).
  Two concurrent requests will not create a 101st charger or a duplicate `remote_code`.
- Every violation is its own exception type → its own status and text.
- All data access goes through repositories; the service has no Eloquent.
- **legacy → target:** `>= 100` → `>= Charger::MAX_PER_POINT` (a model constant).

### Repository
`app/Repositories/ChargerRepository.php`
```php
public function existsByRemoteCode(string $remoteCode, bool $lockForUpdate = false): bool
{
    $query = Charger::withTrashed()->where('remote_code', $remoteCode);

    if ($lockForUpdate) {
        $query->lockForUpdate();
    }

    return $query->exists();
}

public function create(array $data): Charger
{
    return Charger::create($data);
}
```
- `withTrashed()`: the migration has `unique()` on `remote_code`, and a soft-deleted row still occupies the value.
  The check mirrors what the DB enforces.
- `exists()` on a unique-indexed column — a single index lookup at any table size.
- The repository returns `bool`/a model and decides nothing.

### ❌ Not like this
```php
public function store(Request $request)
{
    $data = $request->validate([
        'remote_code' => 'required|unique:chargers,remote_code',
        'point_id' => 'required|exists:points,id',
        'label' => 'required',
    ]);

    if (Charger::where('point_id', $data['point_id'])->count() >= 100) {
        return response()->json(['error' => 'Limit'], 400);
    }

    return Charger::create($data);
}
```
Why it is rejected:
- logic and Eloquent in the controller — the rule cannot be reused from other entry points; queries outside a repository;
- `unique`/`exists` and the limit count are outside a transaction and a lock: two parallel requests both pass;
- no bounds (`max`), a custom error format (`error` instead of `message`), untranslated text, a status outside the table;
- a raw model is returned.

---

## 2. A service with business logic: starting a charge

`app/Services/ChargingSessionService.php`
```php
public function create(array $data, int $authUserId): ChargingSession
{
    try {
        return transaction(function () use ($data, $authUserId) {
            $connector = $this->getAndValidateConnectorForStart($data['connector_id']);
            $this->validateActiveSessionsLimit($authUserId);
            $user = $this->getAndValidateUserForStart($authUserId);

            $intervalCost = $this->calculateIntervalCost($connector);
            $this->validateUserBalanceForStart($user, $intervalCost);

            $this->deductUserBalance($user->id, $user->balance, $intervalCost);

            $session = $this->chargingSessionRepository->create([
                'user_id' => $authUserId,
                'connector_id' => $connector->id,
                'status' => ChargingSessionStatus::PENDING,
                'price_per_kw' => $connector->price_per_kw,
                'watts' => $connector->watts,
                // ...
            ]);

            if ($connector->charger) {
                $success = $this->ocppCommandRepository->remoteStartTransaction($connector->charger->remote_code, $connector->number, (string) $authUserId);
                if (! $success) {
                    throw new OCPPCommandFailedException;
                }
            }

            $this->mobileCommandRepository->pushActiveSessions($authUserId);

            return $session;
        });
    } catch (Throwable $e) {
        Log::channel('charging')->error('Failed to create charging session: '.$e->getMessage(), [
            'user_id' => $authUserId,
            'connector_id' => $data['connector_id'],
            'exception' => get_class($e),
        ]);
        throw $e;
    }
}

private function getAndValidateConnectorForStart(int $connectorId): Connector
{
    $connector = $this->connectorRepository->findById($connectorId, true);
    if (! $connector) {
        throw new ConnectorNotFoundException;
    }

    if (! $connector->is_connected) {
        throw new ConnectorNotConnectedException;
    }

    if ($connector->status === ConnectorStatus::OFFLINE) {
        throw new ConnectorOfflineException;
    }

    if ($this->chargingSessionRepository->findActiveSessionByConnectorId($connector->id, true)) {
        throw new ConnectorBusyException;
    }

    return $connector;
}
```

Block by block:
- **The public method reads like a scenario**: fetch and check the connector → session limit → user → price → balance →
  charge → session → command to the charger → push. Details hide in `getAndValidate…`/`validate…`.
- **`$authUserId` is an argument**, not `auth()->id()`: the same service can be called without HTTP, and the acting user can
  never come from the request body.
- **Lock order**: the connector (shared resource) first, then the user's sessions, then the user.
- **The price is a snapshot in the session** (`price_per_kw`, `watts` copied from the connector): later tariff changes do not
  affect a running charge. This is a historical fact, not denormalization.
- **Prepaid interval** (`calculateIntervalCost` → `ChargingSession::getIntervalCost`): the formula lives in the model, the service only calls it.
- **Balance (a denormalized total) changes in the same transaction, under the user row lock, via one write path.**
- **A synchronous external command inside the transaction** is a deliberate exception: if the charger rejects
  `RemoteStartTransaction`, the exception rolls back the charge and the session. The push to the app is `defer` (inside
  `pushActiveSessions`), sent after the commit.
- **Log-and-rethrow** outside the transaction: a money incident is visible in the `charging` channel with ids, and the client
  gets the normal domain response.
- **legacy → target:** `$connector->status === ConnectorStatus::OFFLINE` → compare by `->value`.

### ❌ Not like this
```php
public function create(array $data): ChargingSession
{
    $user = auth()->user();
    $connector = Connector::find($data['connector_id']);

    if ($user->balance < $connector->price_per_kw) {
        abort(402);
    }

    $user->decrement('balance', $connector->price_per_kw);
    Http::post(config('custom.ws_server_url').'/command/'.$connector->charger->remote_code, [...]);

    return ChargingSession::create([...]);
}
```
- `auth()`/`abort()` in the service — tied to HTTP;
- no transaction and no locks: two parallel starts charge twice and take the same connector;
- Eloquent and HTTP directly in the service; a gateway failure is not handled, the charge is not rolled back;
- the cost formula is inline — not the one in the model;
- `Connector::find` without a select and with lazy loading of `->charger`.

---

## 3. Error handling: from `throw` to JSON

`app/Services/ChargingSessionService.php`
```php
public function getById(int $id, int $authUserId): ChargingSession
{
    $session = $this->chargingSessionRepository->findById($id);

    if (! $session || $session->user_id !== $authUserId) {
        throw new ChargingSessionNotFoundException;
    }

    return $session;
}
```
`app/Exceptions/ChargingSessionNotFoundException.php`
```php
class ChargingSessionNotFoundException extends RuntimeException {}
```
`bootstrap/app.php`
```php
$e instanceof ChargingSessionNotFoundException => [Response::HTTP_NOT_FOUND, __('all.charging_session_not_found')],
```
`lang/en/all.php`
```php
'charging_session_not_found' => 'Charging session not found',
```
Response: `404 {"message": "Charging session not found"}`. The intent is the language from `Accept-Language`; but the locale is
set once in `AppServiceProvider::boot()`, so under Octane the message language most likely does not change per request
(see testing.md → "Locale").

Explanation:
- "Does not exist" and "belongs to someone else" are **one branch and one answer**: a client cannot enumerate ids to learn
  which sessions exist (IDOR protection). [hypothesis]
- The service throws a type and knows nothing about status or text. The code or the text changes in one place.
- `reportable` returns `false` for mapped exceptions → expected 404s do not reach the log.
- The repository's `findById` returns `?ChargingSession`; the service decides that this is an error.

### ❌ Not like this
```php
$session = ChargingSession::findOrFail($id);          // query outside a repository; default 404 without translation
if ($session->user_id !== auth()->id()) {
    abort(403, 'Forbidden');                           // reveals that the session exists; abort in a service
}
throw new \Exception('Session not found');             // 500, untranslated, pollutes the log
```

---

## 4. Transaction, locks and refunds: stopping a session

`app/Actions/StopChargingSessionAction.php`
```php
public function execute(int $id, User $authUser, bool $isHardwareInitiated = false, ?int $meterStop = null): ChargingSession
{
    try {
        return transaction(function () use ($id, $authUser, $isHardwareInitiated, $meterStop) {
            $session = $this->getAndValidateSessionForStop($id, $authUser->id);   // findById(..., true) + status check

            $now = now();
            $isPending = $session->status === ChargingSessionStatus::PENDING;
            $intervalCost = $session->getIntervalCost();

            if ($isPending) {
                $refundAmount = $intervalCost;
                $finalWatts = 0;
                $finalAmount = 0.00;
                $status = ChargingSessionStatus::CANCELLED;
            } else {
                $timeBasedFinalWatts = $session->getEstimatedWattsDelivered($now);

                if ($meterStop !== null && $session->meter_start !== null) {
                    $delivered = (int) $meterStop - (int) $session->meter_start;
                    $finalWatts = $delivered > 0 ? $delivered : $timeBasedFinalWatts;
                } else {
                    $finalWatts = $timeBasedFinalWatts;
                }

                $targetTotalCost = ChargingSession::calculateCost($finalWatts, $session->price_per_kw);
                $exactCost = round($targetTotalCost - (float) $session->withdrawn_amount, 2);

                $refundAmount = round($intervalCost - $exactCost, 2);
                $finalAmount = $targetTotalCost;
                $status = ChargingSessionStatus::COMPLETED;
            }

            $updatedUser = $this->refundUserBalance($session->user_id, $refundAmount);   // findById(..., true) + update
            // ... ledger record, session update, notification, remote stop, pushes (all external calls via defer)

            return $updatedSession;
        });
    } catch (Throwable $e) {
        Log::channel('charging')->error('Failed to stop charging session: '.$e->getMessage(), [
            'session_id' => $id,
            'user_id' => $authUser->id,
            'exception' => get_class($e),
        ]);
        throw $e;
    }
}
```

Explanation:
- **An Action, not a service method:** stopping is called by the user (HTTP), the charger (OCPP `StopTransaction`/`StatusNotification`)
  and cron (low balance, stale pending). `$isHardwareInitiated` tells whether to send `RemoteStopTransaction` back.
- **The session and the user are locked** (`findById(..., true)`): a parallel stop from the app and from the charger cannot refund
  twice; the second call gets `InvalidSessionStateException` (idempotency through a state check).
- **Two sources of truth for energy:** the charger meter (`meterStop - meter_start`) if present and plausible, otherwise the
  time-based estimate. The fallback is decided in one place. Hardware data is cast to `int` (untrusted input).
- **Every intermediate money value is rounded** with `round(..., 2)`; the balance and the ledger change in one transaction.
- **Pending → CANCELLED with a full refund** — a separate branch, no withdrawal record.
- **legacy → target:** the method must be named `handle`; compare statuses by `->value`.

### ❌ Not like this
```php
$session->update(['status' => 'completed', 'finished_at' => now()]);   // raw status string
$user->update(['balance' => $user->balance + $refund]);                 // no transaction/lock: a crash between updates loses money
FirebasePush::toToken($user->fcm_token)->send();                        // synchronous, inside the scenario: FCM down → stop fails
```

---

## 5. Gateway repository and `defer()`

`app/Repositories/MobileCommandRepository.php`
```php
public function pushUserUpdated(User $user): void
{
    $userData = UserResource::make($user)->resolve();
    $userId = $user->id;

    defer(function () use ($userData, $userId) {
        $payload = [
            'event' => 'UserUpdated',
            'data' => $userData,
            'user_id' => $userId,
        ];

        $this->sendPayload($payload);
    });
}
```
`app/Actions/SendWebsocketRequestAction.php`
```php
try {
    $response = Http::withHeaders(['Ws-Secret' => $secret])
        ->timeout(5)
        ->post("{$gatewayUrl}{$path}", $payload);

    if (! $response->successful()) {
        Log::error("{$logPrefix} Failed to send WS request", ['path' => $path, 'status' => $response->status(), 'body' => $response->body()]);

        return null;
    }

    return $response->json() ?? [];
} catch (\Throwable $e) {
    Log::error("{$logPrefix} Exception sending WS request", ['path' => $path, 'error' => $e->getMessage()]);

    return null;
}
```

Explanation:
- The service calls `pushUserUpdated($user)` and does not know there is HTTP, a Node gateway and a WebSocket behind it.
- **Data snapshot before `defer`** (`resolve()` outside): the closure captures a ready array and an int — no lazy loading after
  the request, no risk the model changes before sending, and no model graph kept alive in memory.
- `defer` runs after the response: the business operation does not wait for the network, no queue worker needed.
- Gateway failure → log + `null`; the operation does not fail. `timeout(5)` bounds how long a Swoole worker is occupied.
- The secret goes in a header, the URL comes from config (no SSRF).
- **legacy → target:** `'UserUpdated'` → `MobileEvent::USER_UPDATED->value`; the HTTP client belongs in a repository, not an Action.

### ❌ Not like this
```php
// in a service
Http::post(config('custom.ws_server_url').'/broadcast', ['data' => $user]);   // external HTTP in a service, synchronous, raw model
dispatch(new SendMobileCommandJob($user));                                      // a queue for one HTTP call — extra infrastructure
defer(fn () => $this->send(UserResource::make($user)->resolve()));              // serializing inside defer — lazy loads, stale state, model kept in memory
```

---

## 6. An optimised query with MySQL alone

`app/Repositories/PointRepository.php`
```php
public function getInRadius(float $latitude, float $longitude, float $radius): Collection
{
    $latOffset = $radius / 111320;
    $lngOffset = $radius / (111320 * max(abs(cos(deg2rad($latitude))), 0.0001));

    $bbox = sprintf('POLYGON((%F %F, %F %F, %F %F, %F %F, %F %F))', /* square corners */);

    return Point::select(Point::DEFAULT_SELECT_COLUMNS)
        ->withPrice()
        ->selectRaw($this->asLatitudeAndLongitude())
        ->whereRaw('MBRContains(ST_GeomFromText(?, 4326), coordinates)', [$bbox])
        ->whereRaw(
            "ST_Distance_Sphere(coordinates, ST_GeomFromText(CONCAT('POINT(', ?, ' ', ?, ')'), 4326)) <= ?",
            [$latitude, $longitude, $radius]
        )
        ->limit(1000)
        ->get();
}
```
plus in the service: `$radius = min(Point::MAX_RADIUS, $radius);`

Explanation:
- **Two-step filter:** `MBRContains` on the bounding box uses the spatial index and cheaply discards almost everything;
  the exact `ST_Distance_Sphere` runs only for the remaining rows. Cost depends on the area, not the table size.
- `max(..., 0.0001)` prevents division by zero near the poles.
- **Hard caps:** `MAX_RADIUS` in the service and `limit(1000)` in the query — one request cannot pull the whole table (DoS).
- All user values go through `?` bindings; `%F` formats typed floats only.
- `withPrice()` — a subquery in the select instead of loading connectors for one price.
- No PostGIS, Elastic or geo services (minimum infrastructure).

### ❌ Not like this
```php
return Point::all()->filter(fn ($p) => haversine($p, $lat, $lng) <= $radius);   // whole table in memory, all columns, N+1 on translations
```

---

## 7. FormRequest: a form vs a service request

Service request — **stop** on the first error:
`app/Http/Requests/User/VerifyPhoneRequest.php`
```php
protected $stopOnFirstFailure = true;

public function rules(): array
{
    return [
        'phone' => ['bail', 'required', 'string', 'regex:/^\d{9}$/'],
        'code' => ['bail', 'required', 'string', 'size:6'],
    ];
}
```

Multi-field form — **without** `bail` and `$stopOnFirstFailure` (the target; the project currently has `$stopOnFirstFailure = true`, which is legacy):
```php
class UpdatePointRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'latitude' => ['sometimes', 'numeric', 'between:-90,90'],
            'longitude' => ['sometimes', 'numeric', 'between:-180,180'],
            'company_id' => ['sometimes', 'integer', 'min:1', 'max:999999999999999999'],
            'translations' => ['sometimes', 'array', 'min:1', 'max:1000'],
            'translations.*.language_code' => ['required', 'distinct', 'string', 'size:2'],
            'translations.*.name' => ['sometimes', 'string', 'min:1', 'max:255'],
            'opening_time' => ['sometimes', 'date_format:H:i'],
            'closing_time' => ['sometimes', 'date_format:H:i'],
        ];
    }
}
```
- The admin sees every invalid field of the form in one request.
- Both variants have bounds on every field, anchored regexes, and no privileged fields.

### ❌ Not like this
```php
'company_id' => 'required|exists:companies,id',   // a DB query in a FormRequest, outside the transaction; string syntax; no bounds
'name' => 'required',                               // no type and no max → anything goes to the DB
'role' => ['sometimes', 'string'],                  // a privileged field in a user-facing request → privilege escalation
```

---

## 8. Fixed values: an enum instead of raw strings

`app/Services/OCPPService.php::handleStatusNotification` (legacy)
```php
$status = $payload['status'] ?? '';
// ...
$isPluggedIn = in_array($status, ['Preparing', 'Charging', 'SuspendedEV', 'SuspendedEVSE', 'Finishing']);
$this->connectorRepository->update($connector->id, ['is_connected' => $isPluggedIn]);
// ...
if ($status === 'Charging') {
    // ...
} elseif (in_array($status, ['Finishing', 'Available', 'Faulted'])) {
    // ...
}
```

Target — the set of values and the rules about it live in the enum:
```php
// app/Enums/OCPPConnectorStatus.php
enum OCPPConnectorStatus: string
{
    case AVAILABLE = 'Available';
    case PREPARING = 'Preparing';
    case CHARGING = 'Charging';
    case SUSPENDED_EV = 'SuspendedEV';
    case SUSPENDED_EVSE = 'SuspendedEVSE';
    case FINISHING = 'Finishing';
    case FAULTED = 'Faulted';

    public function isPluggedIn(): bool
    {
        return in_array($this, [self::PREPARING, self::CHARGING, self::SUSPENDED_EV, self::SUSPENDED_EVSE, self::FINISHING], true);
    }

    public function stopsSession(): bool
    {
        return in_array($this, [self::FINISHING, self::AVAILABLE, self::FAULTED], true);
    }
}

// app/Services/OCPPService.php
$status = OCPPConnectorStatus::tryFrom($payload['status'] ?? '');
if (! $status) {
    return [];
}

$this->connectorRepository->update($connector->id, ['is_connected' => $status->isPluggedIn()]);

if ($status->value === OCPPConnectorStatus::CHARGING->value) {
    // ...
} elseif ($status->stopsSession()) {
    // ...
}
```

Explanation:
- **One source of truth:** "cable plugged in" and "time to stop the session" are declared once, in the enum, next to the values.
- **A new value = a new `case`.** No migration, no ALTER, no searching for strings (the enum lives in code, the column is `string`).
- **Typos are caught statically:** `OCPPConnectorStatus::CHARGNG` is a Larastan error, while `'Chargng'` silently returns `false`.
- **An unknown value from hardware is handled explicitly** (`tryFrom` → `null` → early return) instead of falling through every branch.
- Comparison by `->value` (the author's rule).

### ❌ Not like this
```php
if ($user->role == 'admin') { ... }                        // raw string, loose comparison, no cast
$table->enum('status', ['pending', 'charging']);           // the set of values in the DB: a new value = ALTER in production
'event' => 'UserUpdated'                                    // an event name as a string in three places
```

---

## 9. A query that stays in milliseconds on billions of rows

`database/migrations/2026_04_09_000000_create_transactions_table.php` + `app/Repositories/TransactionRepository.php`
```php
// migration
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->index(['user_id', 'created_at']);

// repository
public function cursorPaginateByUserId(int $userId): CursorPaginator
{
    return Transaction::where('user_id', $userId)
        ->select(Transaction::DEFAULT_SELECT_COLUMNS)
        ->latest('id')
        ->cursorPaginate(20);
}
```

Explanation — this one is a near miss, and a good lesson:
- **The filter is served, the sort is not.** The only index starting with `user_id` is `['user_id', 'created_at']`
  (MySQL reuses it for the foreign key, so no separate `user_id` index is created). The query orders by `id`, so for one user
  MySQL must read **all** of that user's rows and filesort them before returning 21. Fine for a typical user, linear for a user
  (or an admin/system account) with millions of ledger rows.
- **legacy → target:** make the index match `where` + `order by` exactly — either order by the indexed columns
  (`->orderByDesc('created_at')->orderByDesc('id')` with the index `['user_id', 'created_at', 'id']`), or add `['user_id', 'id']`
  and keep `latest('id')`. Then MySQL walks the index in order and stops after 21 rows.
- **Keyset pagination:** the next page is `where (sort columns) < :cursor` on the same index — page 1 and page 100 000 cost the
  same. No `OFFSET`, no `COUNT(*)`.
- **Only the needed columns** (`DEFAULT_SELECT_COLUMNS`) — small rows, nothing extra to leak.
- Check it with `EXPLAIN`: the target plan has `type: ref` on the composite index and **no** `Using filesort`.

`app/Repositories/ChargingSessionRepository.php::chunkActiveSessions` + index `['status', 'last_progress_updated_at']`
```php
ChargingSession::with([...])
    ->where('status', ChargingSessionStatus::CHARGING)
    ->where(function ($query) {
        $query->where('last_progress_updated_at', '<=', now()->subSeconds(ChargingSession::PROGRESS_INTERVAL_SECONDS))
            ->orWhereNull('last_progress_updated_at');
    })
    ->chunkById($count, $callback);
```
- The composite index matches the filter (equality on `status`, range on `last_progress_updated_at`); finished sessions — the
  overwhelming majority over time — are never touched.
- `chunkById` walks by primary key in fixed-size batches: constant memory per batch, no `OFFSET`.

### ❌ Not like this
```php
Transaction::where('user_id', $userId)->orderByDesc('created_at')->paginate(20);   // COUNT(*) + OFFSET: page N reads N×20 rows
Transaction::whereRaw('DATE(created_at) = ?', [$day])->get();                       // function on an indexed column → full scan; unbounded get()
Point::where('name', 'like', '%'.$q.'%')->get();                                    // leading wildcard → full scan; use FULLTEXT
User::all()->each(fn ($u) => $u->update(['balance' => $u->transactions->sum('amount')]));   // everything in memory, N+1, denormalized write outside a lock
```

---

<a id="example-test"></a>
## 10. A Feature test (reference — the project has no tests yet)

`tests/Feature/ChargingSession/StoreChargingSessionTest.php`
```php
<?php

namespace Tests\Feature\ChargingSession;

use App\Enums\ChargingSessionStatus;
use App\Models\Charger;
use App\Models\ChargingSession;
use App\Models\Connector;
use App\Models\ConnectorType;
use App\Models\User;
use Illuminate\Http\Client\Request;
use Illuminate\Support\Facades\Http;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

class StoreChargingSessionTest extends TestCase
{
    public function test_store_creates_pending_session_and_deducts_interval_cost(): void
    {
        $user = User::factory()->create(['balance' => 100]);
        $connector = $this->makeReadyConnector(watts: 60000, pricePerKw: 1.5);
        Sanctum::actingAs($user);

        $response = $this->postJson('/api/charging-sessions', ['connector_id' => $connector->id]);

        $response->assertCreated()
            ->assertJsonPath('status', ChargingSessionStatus::PENDING->value)
            ->assertJsonPath('connector_id', $connector->id);
        $this->assertSame(98.5, $user->fresh()->balance);
        Http::assertSent(fn (Request $request) => str_contains($request->url(), '/command/') && $request['action'] === 'RemoteStartTransaction');
    }

    public function test_store_returns_409_when_connector_is_busy(): void
    {
        $user = User::factory()->create(['balance' => 100]);
        $connector = $this->makeReadyConnector();
        ChargingSession::factory()->for($connector)->for(User::factory())->create(['status' => ChargingSessionStatus::CHARGING]);
        Sanctum::actingAs($user);

        $response = $this->postJson('/api/charging-sessions', ['connector_id' => $connector->id]);

        $response->assertConflict()
            ->assertExactJson(['message' => __('all.connector_busy')]);
        $this->assertSame(100.0, $user->fresh()->balance);
    }

    public function test_store_returns_402_when_balance_is_insufficient(): void
    {
        $user = User::factory()->create(['balance' => 0]);
        $connector = $this->makeReadyConnector();
        Sanctum::actingAs($user);

        $this->postJson('/api/charging-sessions', ['connector_id' => $connector->id])
            ->assertStatus(402);

        $this->assertDatabaseCount('charging_sessions', 0);
    }

    public function test_store_rolls_back_when_charger_rejects_command(): void
    {
        Http::fake(['*/command/*' => Http::response(['status' => 'Rejected'])]);
        $user = User::factory()->create(['balance' => 100]);
        $connector = $this->makeReadyConnector();
        Sanctum::actingAs($user);

        $this->postJson('/api/charging-sessions', ['connector_id' => $connector->id])
            ->assertUnprocessable();

        $this->assertSame(100.0, $user->fresh()->balance);
        $this->assertDatabaseCount('charging_sessions', 0);
    }

    public function test_store_requires_authentication(): void
    {
        $this->postJson('/api/charging-sessions', ['connector_id' => 1])
            ->assertUnauthorized();
    }

    private function makeReadyConnector(int $watts = 60000, float $pricePerKw = 1.5): Connector
    {
        $charger = Charger::factory()->create(['last_heartbeat_at' => now()]);

        return Connector::factory()
            ->for($charger)
            ->for(ConnectorType::factory())
            ->create([
                'number' => 1,
                'watts' => $watts,
                'price_per_kw' => $pricePerKw,
                'is_connected' => true,
            ]);
    }
}
```

Explanation:
- **Money is asserted as an exact amount** (`98.5` = 1 interval = 60 kW × 60 s / 3600 = 1 kWh × 1.5), not "the balance went down".
- **Every service exception is its own test**, with the status from the table and the text from `lang` via `__()`, not a literal.
- **The rollback is asserted explicitly**: when the charger rejects, neither the balance nor the sessions table changes.
- **External systems only via `Http::fake`**; `preventStrayRequests` in the base `TestCase` keeps the test off the network.
  `Http::assertSent` works because `defer` runs on `terminate` of an HTTP test.
- Relations are created explicitly (`for($connector)`), not from random DB rows.
- `number` and `ConnectorType` are set explicitly: `ConnectorFactory` defaults `number` to `Connector-##` (while
  `remoteStartTransaction` expects an `int`) and falls back to `connector_type_id = 1` when no types exist.
- (`Charger::factory()` needs `HasFactory` + a factory for `Charger` — the project does not have one yet.)

### ❌ Not like this
```php
public function test_it_works(): void
{
    $this->seed();                                                      // depends on seeders and random data
    $user = User::first();
    $this->actingAs($user)->post('/api/charging-sessions', ['connector_id' => 1])->assertStatus(200);
    $this->assertTrue(User::first()->balance < $user->balance);          // does not check the amount
}
```
Plus mocking every repository in a service "unit test" — that tests the mocks, not the behaviour.
