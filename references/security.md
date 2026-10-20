# Security

**The author's principle: make the code as secure as possible, in every corner.** **[confirmed]**
Every layer has its own security duty, and no layer relies on a layer above or below to be the only defence.
When security conflicts with convenience, security wins. When it conflicts with performance, take the secure option
and optimise inside it (an index, a narrower select), not around it.

Markers: **[confirmed]** — stated by the author; **[hypothesis]** — inferred from code; **legacy** — exists in the
project today and must not be copied; **[proposal]** — recommended target, not yet in the project.

---

## Threat model of the project

| Actor | Trust | Talks to | Main risks |
|---|---|---|---|
| Mobile client | **untrusted** | `routes/api.php` (public + `auth:sanctum`) | account takeover via OTP, IDOR (someone else's session), double spending, scraping |
| Admin | trusted after `role:admin` | admin group | privilege escalation, mass assignment of `role`/`balance` |
| Node WS/OCPP gateway | trusted via shared secret | `/api/ocpp/*` (`OCPPMiddleware`), receives `/broadcast`, `/command/*` | forged OCPP messages → free charging / fake stops, leaked secret |
| Chargers | untrusted hardware data | gateway → `OCPPService` | garbage/hostile payloads, replays, unknown statuses |
| External APIs (FCM, SMS) | outgoing only | repositories-gateways | SSRF, secret leakage, hanging calls |

Money is the main asset: balance, prepaid intervals, refunds. Any bug in concurrency is a security bug.

---

## 1. Authentication

What the project does (OTP over SMS + Sanctum bearer tokens), `app/Services/UserService.php`:

| Control | Where | Why |
|---|---|---|
| 6-digit code from `random_int(100000, 999999)` | `sendCode` | CSPRNG, not `rand()` |
| Resend limit: one code per `MINUTES` + `throttle:1,1` on `send-code` | `isAlreadySent`, `routes/api.php` | SMS bombing, cost abuse |
| `MAX_ATTEMPTS = 3`, attempt counted **before** the comparison | `verifyPhone` → `incrementAttempts` | brute force: a failed comparison still burns an attempt |
| TTL: code valid only for `MINUTES` | `created_at->between(...)` | short brute-force window |
| `throttle:4,1` on `verify-phone` | routes | second layer against brute force |
| One generic `BadRequestException` for "no code / too many attempts / wrong / expired" | `verifyPhone` | no oracle telling the attacker which condition failed [hypothesis] |
| Codes deleted after successful login | `deleteVerifyCodes` | a code is single-use |
| All previous tokens revoked on login | `deleteTokens` | one device = one session; a stolen old token dies on next login [hypothesis] |
| Logout deletes only the current token | `deleteCurrentToken` | |

Rules for new code:
- Compare secrets, codes and tokens with **`hash_equals($expected, $given)`**, never `===`/`!==`
  (legacy: `$phoneCode->code !== $code`). **[proposal]**
- Store OTP codes **hashed** (`hash('sha256', $code)`) so a DB leak does not expose live codes. **[proposal]**
- Give Sanctum tokens a lifetime: `config/sanctum.php` → `'expiration' => null` today (tokens never expire). **[proposal]**
- **No dev shortcuts in code at all**: no `dev-token` route, no "magic phone gets code 123456"
  (legacy: `UserController::devToken` issues an **admin** token without auth; `UserService::sendCode` bypass for `999999999`).
  CI deploys `main` as is, so any shortcut ships to production. For local work, use seeders and `php artisan tinker`.
- Every authentication-related public endpoint has a `throttle`.

## 2. Authorization

- Access level is declared by **route groups**, not inside controllers: public → `auth:sanctum` → `role:` + `Role::ADMIN->value`.
- Role checks compare **enum values**, never raw strings. Cast `'role' => Role::class` on `User`. **[proposal]**
- **The acting user comes only from `auth()`** in the controller and is passed to the service as an argument
  (`getById($id, auth()->id())`). Never take `user_id` from the request body or query for ownership.
- **Ownership is checked in the service** and answers `NotFound`, not `Forbidden`:
  ```php
  if (! $session || $session->user_id !== $authUserId) {
      throw new ChargingSessionNotFoundException;
  }
  ```
  An attacker enumerating ids cannot tell "not yours" from "does not exist". [hypothesis]
- Every new endpoint that returns or changes a user-owned record needs an ownership check **and** a test for it (404 for another user's id).

## 3. Mass assignment and privileged fields

- Services receive only `$request->validated()` — never `$request->all()`, `$request->input()` without validation, or `$request->except(...)`.
- Models declare an explicit `$fillable`; `$guarded = []` and `forceFill($input)` are forbidden.
- `$fillable` on `User` contains `role` and `balance`, so the **FormRequest whitelist is the real protection**:
  user-facing requests must never contain privileged fields. Real example — `UpdateSelfRequest` allows only
  `name`, `surname`, `fcm_token`, `device`, `app_version`, `app_lang`.
- Privileged fields (`role`, `balance`, `last_read_notification_id`, `price_per_kw` of a running session) are changed only by
  dedicated service methods (`topUp`, the charging flow), never by a generic `update($data)`.

## 4. Input validation

- Every field has a type **and** bounds: strings `min:1|max:255`, ids `min:1|max:999999999999999`, money
  `min:0.01|max:99999999.99`, arrays `min:1|max:1000`, coordinates `between`.
  Unbounded input is a DoS and overflow vector.
- Regexes are anchored: `regex:/^\d{9}$/`.
- Fixed sets — `new Enum(X::class)`; never trust a free string to match a case.
- **File uploads**: `image` + `mimes:png,jpg,jpeg,webp` + `max:2048`, stored with `$file->store($dir, 'public')`
  (random file name — the client name is never used), old file deleted on replace (`ConnectorTypeService::update`).
  Never accept SVG/HTML uploads on a public disk (stored XSS).
- Data from hardware (OCPP payloads) is untrusted: cast (`(int) ($payload['connectorId'] ?? 0)`), default, and
  map unknown values to an explicit branch (`tryFrom` → `null` → early return). Never `match` on a raw payload string
  without a default.

## 5. Injection

- Eloquent / query builder with bindings everywhere.
- Raw SQL only in repositories and only with `?` bindings:
  ```php
  ->whereRaw('MBRContains(ST_GeomFromText(?, 4326), coordinates)', [$bbox])
  ```
- No user input is ever interpolated into raw SQL. Values built in PHP from typed floats via `sprintf('%F', ...)` are
  acceptable (`PointRepository::getInRadius`), because the type guarantees the format.
  `DB::getPdo()->quote()` (legacy in `PointRepository::search`) works, but bindings are preferred.
- Column names, sort directions and relation names never come from input. If sorting/filtering by a client field is
  needed — a whitelist enum.
- Output to HTML views (`resources/views`) only via `{{ }}`, never `{!! !!}` with user data.

## 6. Data exposure

- **Select only what you return**: `DEFAULT_SELECT_COLUMNS` on every query. Columns never selected cannot leak. **[confirmed]**
- `$hidden` on models for timestamps, `deleted_at`, and sensitive identifiers (`Charger::$hidden` includes `remote_code` —
  the charger's identity towards the gateway — and `last_heartbeat_at`).
- **Always a Resource** with an explicit field list; a raw model is never returned. **[confirmed]**
- Errors never leak internals: mapped domain exceptions return a translated `message` only;
  `APP_DEBUG=false` in production (stack traces and env values otherwise go to the client).
- Logs carry ids only — never phone numbers, OTP codes, tokens, FCM tokens or payment data.

## 7. Secrets and configuration

- Secrets live only in `.env`, are read only in `config/*.php` (project ones in `config/custom.php`), and are used via `config(...)`.
- **Nothing secret is committed.** `.env.example` and `firebase.example.json` contain empty placeholders only.
  (legacy: `firebase.json` with a `private_key` is tracked in git — the key must be rotated and the file removed from history.)
- Secret comparison: `hash_equals`.
- **Fail closed**: if a required secret is empty, reject — never treat "both empty" as a match.
  Target shape for a shared-secret middleware — [templates/secret-middleware.stub](../templates/secret-middleware.stub).
  (legacy: `OCPPMiddleware` compares `$request->header('Ws-Secret') !== $secret`; with `WS_SECRET` unset and no header,
  `null !== null` is `false` and the request passes.)
- Outgoing requests send secrets only in headers (`Ws-Secret`), never in URLs or query strings
  (legacy: the SMS gateway receives `login` and a hash in the query string of a GET request — keep, the provider dictates it,
  but prefer HTTPS for its URL).

## 8. Service-to-service and outgoing calls

- Incoming gateway calls: shared secret in a header, checked by middleware on the whole `ocpp` prefix.
- Unknown charger → `ChargerRejectedException` (401); every OCPP message updates `last_heartbeat_at` only after the charger is found.
- Outgoing calls: URL from config only (no URL built from input → no SSRF), `timeout(5)`, response status checked
  (`OCPPCommandRepository::sendCommand` treats anything but `Accepted` as failure), failures logged without secrets.
- Network calls run in `defer()` so a slow or malicious peer cannot hold a DB transaction or a request open,
  except when the result decides the commit (`remoteStartTransaction`).

## 9. Abuse and DoS limits

- Hard caps on every potentially large result: `Point::MAX_RADIUS`, `limit(1000)`, `cursorPaginate(20)`, `simplePaginate(15)`,
  `MAX_ACTIVE_SESSIONS_PER_USER`, `ConnectorType::MAX_COUNT`, charger-per-point limit.
- `throttle` on authentication endpoints today; add it to expensive public endpoints
  (`points`, `points/search`, `connectors/{id}`). **[proposal]**
- Background work is chunked (`chunkById(500)`) and protected with `withoutOverlapping()`.

## 10. Money integrity (security of funds)

- Every check-then-act on money or a shared resource runs inside `transaction()` with `lockForUpdate`
  (connector → user sessions → user, same order everywhere).
- State checks make operations idempotent: stopping an already stopped session throws `InvalidSessionStateException`
  instead of refunding twice.
- Every movement of money is recorded in `transactions` (`WITHDRAWAL` / `REPLENISHMENT`).
- Amounts are rounded with `round(..., 2)` after each step; stored as `decimal(36, 2)`.
- External confirmation before charging starts: if the charger rejects `RemoteStartTransaction`, the whole transaction
  (including the prepaid interval) rolls back.

## 11. Octane / long-running workers

The app runs on Octane + Swoole: one worker serves many users in sequence.
- **No per-request state in static properties, singletons or service fields.** `readonly` services, repositories and
  actions are stateless by construction, so one user's data cannot leak into the next request. [hypothesis: one of the
  reasons everything is `readonly`]
- Anything request-scoped (current user, locale, request data) is passed as an argument, never cached on an object.
- Data for `defer()` is captured as a snapshot before the callback (`UserResource::make($user)->resolve()`).

## 12. Known gaps in the current project (fix when touching the code)

| Severity | Gap | Where |
|---|---|---|
| critical | Public route issues an admin token | `routes/api.php` `dev-token`, `UserController::devToken` |
| critical | Service-account private key in git | `firebase.json` |
| high | Shared secret check passes when the secret is unset; not constant-time | `OCPPMiddleware` |
| high | OTP bypass for a fixed phone (guarded only by `isLocal()`) | `UserService::sendCode` |
| medium | Sanctum tokens never expire | `config/sanctum.php` `expiration => null` |
| medium | OTP stored in plain text, compared with `!==` | `VerificationCode`, `UserService::verifyPhone` |
| medium | `role` not cast to enum, compared as a string | `User`, `RoleMiddleware` |
| low | No throttle on public read endpoints | `routes/api.php` |
| low | SMS errors swallowed without a log | `NotificationRepository::sendSms` |

## Security checklist (every change)

- [ ] Route is in the right access group; authentication-related or expensive public routes have `throttle`.
- [ ] Acting user comes from `auth()` in the controller; ownership checked in the service → 404 for foreign ids (+ test).
- [ ] Service receives only `validated()`; no privileged fields in user-facing FormRequests; `$fillable` explicit.
- [ ] Every input field has a type and bounds; fixed sets via `Enum`; uploads restricted by `image|mimes|max`.
- [ ] No user input in raw SQL, column names or URLs; raw SQL only with bindings.
- [ ] Query selects only needed columns; response is a Resource; sensitive fields in `$hidden`.
- [ ] Secrets only via `config()`; compared with `hash_equals`; fail closed when empty; nothing secret in git or logs.
- [ ] Outgoing calls: URL from config, `timeout`, response checked, failures logged without secrets.
- [ ] Money/shared resources: `transaction()` + `lockForUpdate`, state check for idempotency, ledger record.
- [ ] No mutable per-request state on objects (Octane).
- [ ] No dev shortcuts, debug routes or bypasses in the diff.
