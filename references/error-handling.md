# Errors, logging, API responses

## Domain exceptions

### The class

```php
// app/Exceptions/ConnectorBusyException.php
<?php

namespace App\Exceptions;

use RuntimeException;

class ConnectorBusyException extends RuntimeException {}
```

- Empty body, extends `RuntimeException`, no message and no code.
- One class = one business situation. Generic `BadRequestException`/`TooManyRequestsException` only where detail is hidden
  on purpose (wrong OTP: `UserService::verifyPhone` does not say what exactly is wrong — the code, the expiry or the number
  of attempts). [hypothesis: no oracle for brute force — see security.md]

*Why empty:* the text depends on the client's language and the status on the HTTP contract; neither is knowledge of the
service. [hypothesis]

### Where they are thrown

Only in a **service** or a business **Action**. A repository returns `null`/`false`; a controller checks nothing.

```php
// app/Services/PointService.php
public function delete(int $id): void
{
    if (! $this->pointRepository->softDelete($id)) {
        throw new PointNotFoundException;
    }
}
```

### Mapping to HTTP — one place

```php
// bootstrap/app.php
function mapExceptionToResponseData(Throwable $e): ?array
{
    return match (true) {
        $e instanceof ConnectorBusyException => [Response::HTTP_CONFLICT, __('all.connector_busy')],
        // ...
        default => null,
    };
}

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->reportable(function (Throwable $e) {
        if (mapExceptionToResponseData($e) !== null) {
            return false;            // expected business errors do not pollute the log
        }
    });

    $exceptions->render(function (Throwable $e) {
        $customResponse = mapExceptionToResponseData($e);
        if ($customResponse === null) {
            return null;             // everything else — default Laravel handling
        }
        [$status, $message] = $customResponse;

        return response()->json(['message' => $message], $status);
    });
```

**Adding a new error — three mandatory steps:**
1. A class in `app/Exceptions/`.
2. A line in the `match` in `bootstrap/app.php` (status from the table below).
3. A `snake_case` key in **all three** files: `lang/ru/all.php`, `lang/tg/all.php`, `lang/en/all.php`.

The `match` line has a fixed form — `$e instanceof XxxException => [Response::HTTP_XXX, __('all.xxx')],` on one line:
the Swagger generator reads statuses from `bootstrap/app.php` with a regex (see [architecture.md](architecture.md#swagger)).

Skipping step 2 gives the client a 500 and a log entry; skipping step 3 returns the raw key `all.xxx`.

> The message language is the application locale. It is set in `AppServiceProvider::boot()` via
> `app()->setLocale(accept_language())` — once, when the application boots. Under Octane providers are not re-booted per
> request and `FlushLocaleState` resets the locale to `config('app.locale')`, so error messages most likely do not follow
> `Accept-Language` (translations of **data** work: `accept_language()` reads the header on every request).
> New code must not rely on it; the fix is in the list of project improvements (a middleware setting the locale per request).

### Choosing the HTTP status (as used in the project)

| Situation | Status | Examples |
|---|---|---|
| The entity does not exist / belongs to someone else | `404 HTTP_NOT_FOUND` | `PointNotFoundException`, `ChargingSessionNotFoundException` (including another user's session) |
| Business rule / limit / invalid state violated | `422 HTTP_UNPROCESSABLE_ENTITY` | `ChargerLimitReachedException`, `ChargerAlreadyExistsException`, `InvalidSessionStateException`, `ConnectorOfflineException`, `OCPPCommandFailedException` |
| Conflict with the current state of the resource | `409 HTTP_CONFLICT` | `ConnectorBusyException`, `ActiveSessionExistsException` |
| Not enough money | `402 HTTP_PAYMENT_REQUIRED` | `InsufficientFundsException` |
| Physical precondition not met | `412 HTTP_PRECONDITION_FAILED` | `ConnectorNotConnectedException` (cable not plugged in) |
| Unknown charger on an OCPP endpoint | `401 HTTP_UNAUTHORIZED` | `ChargerRejectedException` |
| Too often | `429` | `TooManyRequestsException`, `ThrottleRequestsException` |
| Bad request without details | `400` | `BadRequestException` |
| Missing role | `403` | `RoleMiddleware` → `abort(Response::HTTP_FORBIDDEN)` |

*Why different codes:* the mobile client branches on the status (402 → top-up screen, 412 → "plug in the cable",
409 → "connector is busy") without parsing the text. [hypothesis]

`abort()` is allowed **only in middleware** (`RoleMiddleware`, `OCPPMiddleware`) — that is the HTTP level.

<a id="api-contract"></a>
## Response format

| Case | Body | How |
|---|---|---|
| Business error | `{"message": "Connector is already busy"}` | a domain exception |
| Validation error | standard Laravel 422: `{"message": "...", "errors": {"field": ["..."]}}` | FormRequest |
| Throttle | `{"retry_after": 60}` + 429 | a separate `render` for `ThrottleRequestsException` |
| Object | `{...}` **without** a `data` wrapper | `XxxResource::make($model)` (`JsonResource::withoutWrapping()` in `AppServiceProvider`) |
| List without pagination | `[...]` | `XxxResource::collection($items)` (only for bounded lists) |
| Cursor pagination | `{"data": [...], "next_cursor": "..."\|null}` | `CursorPaginationResource::make($paginator, XxxResource::class)` |
| Simple pagination (search) | `{"data": [...], "next_page_url": "..."\|null}` | `LengthAwarePaginationResource::make($paginator, XxxResource::class)` |
| Empty (action done) | `[]` | `EmptyResource::make()` |
| Token | `{"token": "..."}` | `TokenResource::make($plainTextToken)` |

Rules:
- **Always a Resource**, including admin endpoints and empty responses. **[confirmed]**
  `response()->json($model)` and `response()->json()` in `PointController`, `CompanyController`, `ChargerController`, etc. are legacy.
- **Select first, then the Resource.** The Resource only shapes the contract; extra columns are cut off in the query
  (`DEFAULT_SELECT_COLUMNS`). **[confirmed]**
- Explicit casts in Resources: `(int) $this->watts`, `(float) $this->balance`, `(bool) $this->is_online`,
  nullable — `$this->price !== null ? (float) $this->price : null`. *Why:* MySQL decimals arrive as strings, and the client
  needs a stable type.
- Relations in a Resource only via `whenLoaded('relation')`; a Resource never loads anything (no queries outside repositories).
- Dates are returned as is (Carbon → ISO-8601).
- **The success status is set by Laravel:** a Resource over a model with `wasRecentlyCreated = true` → `201`, otherwise `200`.
  So a repository `create` returning the model from `Model::create()` gives `201` (`POST /charging-sessions`), while a
  `create` that re-reads the model via `findById` (`PointRepository::create`) gives `200`.
  When moving legacy endpoints from `response()->json()` to Resources, check that the client does not depend on 200 vs 201.
- Pagination: cursor, 20 per page (`cursorPaginate(20)`). *Why:* infinite feeds in the mobile app, no `COUNT(*)`, stable under
  inserts, constant cost at any table size. `simplePaginate` only when sorting by a computed relevance over a bounded candidate
  set (`PointRepository::search`), where a cursor is impossible.
- For the admin panel, if the response shape differs from the client one (e.g. the full `translations` array is needed) —
  a separate Resource. Proposed location: `app/Http/Resources/Admin/{Entity}Resource.php`. **[proposal — to confirm]**
- Error responses never contain internals (stack traces, SQL, class names): `APP_DEBUG=false` in production.

## Logging

### Channels

- `charging` — `daily`, 14 days (`config/logging.php`). Everything about money and charging sessions.
- Default channel — incoming OCPP messages (`Log::info("OCPP [{$remoteCode}] {$action}", $payload)`), external gateway
  failures (`SendWebsocketRequestAction`).

### What and how

```php
// app/Services/ChargingSessionService.php::create
} catch (Throwable $e) {
    Log::channel('charging')->error('Failed to create charging session: '.$e->getMessage(), [
        'user_id' => $authUserId,
        'connector_id' => $data['connector_id'],
        'exception' => get_class($e),
    ]);
    throw $e;
}
```

- **Log-and-rethrow** at the boundary of a money operation: outside `transaction()`, with id context and the exception class,
  then the exception is rethrown (the handler turns it into a response). [hypothesis: money incidents can be traced by
  `user_id`/`session_id` without enabling reports for expected business errors]
- Messages in English, `"Failed to <action>: ".$e->getMessage()`; ids in the context or in the text.
- Levels:
  - `error` — the operation was not performed;
  - `warning` — a fallback kicked in (`Bulk progress update failed, falling back to individual processing`) or the other side
    rejected a command (`[OCPP Command] {$action} rejected`);
  - `info` — a significant business event (`Session {id} stopped due to charger offline timeout`).
- Subsystem prefix in square brackets: `[OCPP Command]`, `[Mobile Command]`.
- **Never log** personal data or credentials: phone numbers, OTP codes, tokens, FCM tokens, secrets, full request bodies.
- Debug lines like `Log::info('DEFER EXECUTED…')` never get committed (legacy in `MobileCommandRepository::pushBulkActiveSessions`).
- Log context stays small (ids, short arrays) — large payloads in loops waste memory and disk.

### Where swallowing exceptions is allowed

Only **explicitly listed expected** exceptions, and only in background flows where the state may have changed concurrently:

```php
// ChargingSessionService::autoUpdateProgress — stale pending sessions
try {
    $this->stopChargingSessionAction->execute($session->id, $session->user, false);
} catch (InvalidSessionStateException|ChargingSessionNotFoundException $e) {
}
```

In a batch loop — `catch (Throwable $e)` with logging, so one broken session does not stop the batch.

An external gateway failure (WS, SMS, FCM) must not break the business operation: the result is `null`/`false` + a log line.
(legacy: `catch (Exception) {}` without a log in `NotificationRepository::sendSms` — must log.)
