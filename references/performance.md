# Performance, data access, denormalization, memory

The author's principles **[confirmed]**:
1. **Database queries go strictly through repositories** — and so does every outgoing HTTP call to an external system.
   No DB query and no external HTTP call in a service, controller, middleware, FormRequest, Resource, model accessor or anywhere else.
2. **Every query is designed to run in milliseconds even on billions of rows.**
3. **Avoid denormalization as much as possible.** If it is unavoidable, do it like a senior: on the code side, with no
   way for the copies to drift apart.
4. **Memory leaks are strictly unacceptable** (the app runs on long-living Octane/Swoole workers).

---

<a id="data-access"></a>
## 1. Only repositories talk to the database and to external systems

| Place | DB query | External HTTP |
|---|---|---|
| Repository (`app/Repositories`) | ✅ the only place | ✅ the only place (gateway repositories) |
| Service / Action | ❌ — calls repositories | ❌ — calls gateway repositories |
| Controller, middleware | ❌ (`$request->user()` from the auth guard is fine) | ❌ |
| FormRequest | ❌ — hence **no `exists:` / `unique:` rules** (they are DB queries outside the repository and outside the transaction) | ❌ |
| Resource | ❌ — only `whenLoaded()`; never load a relation in a Resource | ❌ |
| Model accessor / `$appends` | ❌ — computed only from loaded attributes/relations (`Connector::status` reads `active_session_exists` / loaded relations) | ❌ |
| Migration / seeder | ✅ (schema, test data) | ❌ |

Why: one place owns every query, so every query can be reviewed for indexes, selected columns and locks; the service stays
testable and free of infrastructure; an external outage is handled in one place.

Legacy that breaks the rule (fix when touching):
- `UserController::devToken` queries `User::` directly (and should not exist at all — see security.md);
- `SendWebsocketRequestAction` performs the outgoing HTTP call from an Action. It is only used by gateway repositories, but by
  the rule the HTTP client belongs to the repository layer, e.g. a private method of a `WebsocketGatewayRepository` that
  `OCPPCommandRepository` and `MobileCommandRepository` depend on. **[proposal]**
- `Point::scopeWithPrice` builds a subquery inside the model. Scopes are acceptable only as building blocks used **exclusively**
  by repositories; new query pieces go into private repository methods (`asLatitudeAndLongitude()` style).
- `Transaction::iconPath` calls `file_exists()` for every row — filesystem I/O inside an accessor, repeated per item of a list.
  Resolve such values from a static map (an enum method returning the icon path) instead.

---

<a id="queries"></a>
## 2. Milliseconds on billions of rows

A query is acceptable only if its cost depends on the size of the **result page**, not on the size of the **table**.
Before writing a query, decide which index serves it — and add that index in the same change.

### Rules

1. **Every `where` + `order by` combination has a matching index**, columns in the same order as in the query
   (equality columns first, then the range/sort column).
   Real examples: `charging_sessions` → `['status', 'last_progress_updated_at']` for `chunkActiveSessions`,
   `['user_id', 'status']`, `['connector_id', 'status']`.
   InnoDB secondary indexes carry the primary key, so `['user_id', 'id']` (or a plain `user_id` index) serves
   `where('user_id', …)->latest('id')`. Near miss in the project: `transactions` has only `['user_id', 'created_at']` while
   `TransactionRepository::cursorPaginateByUserId` orders by `id` → a filesort over all of the user's rows
   (see [examples.md](examples.md) example 9).
2. **Keyset pagination only:** `cursorPaginate(20)` ordered by an indexed column. **No `OFFSET`/`paginate()`** —
   `OFFSET 10_000_000` reads ten million rows. `simplePaginate` only for relevance-ordered search over an already
   bounded candidate set (`PointRepository::search` takes at most 200 candidates from the FULLTEXT index first).
3. **No `COUNT(*)` over big tables in a request path.** Use `exists()` (`hasActiveSessionsByUserId`, `existsById`),
   `withExists()` instead of loading or counting a relation, or a count that is bounded by an indexed equality
   (`countActiveSessionsByUserId` — few rows per user). `paginate()` is forbidden partly because it runs a `COUNT(*)`.
4. **Every read is bounded:** `first()`, `cursorPaginate(n)`, `limit(n)`, `chunkById(n)`, or a domain cap
   (`MAX_ACTIVE_SESSIONS_PER_USER`, `Point::MAX_RADIUS` + `limit(1000)`). A bare `->get()` is allowed only on a
   result bounded by an indexed equality or a small dictionary table (`ConnectorType::MAX_COUNT`, `languages`).
5. **Indexes must stay usable:**
   - no functions on indexed columns in `where` (`DATE(created_at) = ?` → range `created_at >= ? and created_at < ?`);
   - no leading-wildcard `LIKE '%…'` — use FULLTEXT (`MATCH … AGAINST`, ngram parser) as the project does;
   - be careful with `OR` across different columns and `OR … IS NULL`: prefer two indexed queries or `UNION`
     (risk in `NotificationRepository::getNotificationsByDesc`: `(user_id = ? OR user_id IS NULL) ORDER BY id DESC` —
     all broadcast notifications are scanned and sorted for every user; at scale, split personal and broadcast feeds
     or add an index that serves the sort);
   - compare columns with values of the same type (string column vs string value) to avoid implicit casts.
6. **Cheap prefilter + exact check** for geometry: `MBRContains` on the spatial index, then `ST_Distance_Sphere`
   (`PointRepository::getInRadius`).
7. **Select only needed columns** (`DEFAULT_SELECT_COLUMNS`) so rows stay small and covering indexes become possible.
8. **No N+1:** eager-load with selected columns; `preventLazyLoading` in non-production turns any N+1 into an exception.
9. **Batch writes:** `upsert` / `insert` for many rows (`upsertProgress`, `upsertBalances`), not a loop of `save()`.
10. **Locks are narrow:** `lockForUpdate` only on rows found by the primary key or an index (otherwise InnoDB locks
    ranges/gaps of an index scan), and only inside a short transaction without network calls.
11. **Verify with `EXPLAIN`** every new or changed query against realistic volume: `type` must be `const`/`eq_ref`/`ref`/`range`,
    never `ALL` on a large table; no `Using filesort` on large row sets; `rows` proportional to the page size.

### Background processing

- Large sets: `chunkById(500, …)` (never `chunk()` with offset, never `->get()` of everything).
- One batch = one transaction with bulk `upsert`; on failure, fall back to one-by-one processing (`processSessionsBulk` →
  `processSessionsIndividually`).
- Scheduled tasks: `withoutOverlapping()`.

---

<a id="denormalization"></a>
## 3. Denormalization: avoid; if unavoidable, make drift impossible

### Prefer computing over storing

The project computes derived state instead of storing a copy:
- `Charger::is_online` — computed from `last_heartbeat_at` (no `is_online` column that could go stale);
- `Connector::status` — computed from the existence of an active session (`withExists('activeSession')`) and charger liveness,
  not a `status` column that every flow would have to keep in sync;
- `Point::price` — a subquery, not a copied column.

### Snapshots are not denormalization

`charging_sessions.price_per_kw` and `watts` are copied from the connector **at start**. That is a historical fact
(the price agreed for this session), not a cache — later tariff changes must not affect a running or finished session.
Name and treat such columns as snapshots; never "sync" them back.

### When a stored derived value is unavoidable

`users.balance` is the main real denormalization: a running total that could be computed from the money ledger.
It is kept consistent the senior way, and every new denormalized value must follow the same rules:

1. **One write path.** The value changes only in dedicated service/action methods (`topUp`, session start, progress,
   `StopChargingSessionAction`), never through a generic `update($data)` from input.
2. **Same transaction as the source-of-truth change.** Balance update and the `transactions` ledger insert happen in one
   `transaction()`.
3. **Under a row lock.** The row holding the value is read with `lockForUpdate` before computing the new value
   (`findById($userId, true)`, `getLockedUsersByIdsKeyed`); never read-modify-write without a lock.
4. **Computed from the locked value, then rounded** (`round($currentBalance - $intervalCost, 2)`).
5. **Reconstructible.** It must be possible to recompute the value from the source of truth. Provide a reconciliation
   check (a scheduled read-only comparison that logs mismatches) for every denormalized value. **[proposal]**
   Note: today a session's prepaid interval and `withdrawn_amount` reduce the balance before a final `WITHDRAWAL`
   ledger row exists, so a reconciliation must account for active sessions.
6. **Documented on the model** as derived (`@property float $balance` + one-line comment naming the source of truth). **[proposal]**

What is never acceptable: updating a copy "later" outside the transaction, in `defer()`, in an event listener, or in two
different code paths.

---

<a id="memory"></a>
## 4. No memory leaks (Octane / Swoole)

One worker process serves thousands of requests; anything that survives a request accumulates.
(`config/octane.php` already has `'garbage' => 50` — the GC threshold in MB.)

1. **No mutable state on long-living objects.** Services, repositories and actions are `readonly` and stateless;
   never add caches, arrays or "current user" fields to them, never push into a `static` array.
2. **Register things once, never per request.** Macros, global HTTP middleware, event listeners, `Model::` hooks — only in
   service providers or behind a static "already registered" guard (the removed `ObfuscateIds` middleware registered
   `Http::globalRequestMiddleware` behind `self::$httpMacroRegistered` for exactly this reason). Registering per request
   grows the list forever.
3. **Never load unbounded sets into memory.** `chunkById` / `lazyById` for big sets; process and drop each chunk.
   Do not `->get()` and then filter in PHP.
4. **`defer()` closures capture the minimum:** scalars and already-resolved arrays (`$userData = UserResource::make($user)->resolve();`
   then `use ($userData, $userId)`), not whole models with loaded relations, not `$request`.
5. **Keep batch accumulators local** to the method/chunk callback (`$sessionsToUpdate`, `$usersToPush` live inside the chunk
   closure in `autoUpdateProgress` and die with it).
6. **No query log in production** (`DB::enableQueryLog()` stores every query of the worker's lifetime); no `dump()`/`dd()`.
7. **Bounded log context:** ids and small arrays, never full payloads in loops (legacy: `Log::info("OCPP …", $payload)`
   on every message — fine for size today, but keep payloads small).
8. **Safety net, not a fix:** recycle workers with `php artisan octane:start --max-requests=…` **[proposal]**, but treat any
   growth of worker memory as a bug to be found, not something to hide with restarts.

---

## Performance checklist (every change)

- [ ] Every DB query and external HTTP call is inside a repository.
- [ ] For each new/changed query: the serving index exists (migration in the same change), `EXPLAIN` shows no full scan / large filesort.
- [ ] Reads are bounded (`first`, `limit`, `cursorPaginate`, `chunkById`); no `OFFSET` pagination; no `COUNT(*)` on big tables.
- [ ] No functions on indexed columns, no leading-wildcard `LIKE`, no index-breaking `OR`.
- [ ] Only `DEFAULT_SELECT_COLUMNS`; eager-loads with selected columns; no lazy loading.
- [ ] No new denormalized value — or: one write path, same transaction as the source of truth, row lock, reconstructible.
- [ ] No mutable state on objects/statics; nothing registered per request; `defer()` captures only snapshots.
