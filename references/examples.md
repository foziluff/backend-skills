# Эталонные примеры с разбором

Все фрагменты, кроме целевого кода в примере 8 и примера 9, взяты из проекта; путь указан над каждым.
Места, где реальный код расходится с целевым правилом, помечены **legacy → как надо**.

---

## 1. Эндпоинт целиком через все слои: `POST /api/chargers`

### Маршрут
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
- Доступ задаётся **группой**, а не проверкой в контроллере: по файлу маршрутов сразу видно, что создавать зарядники может только админ.
- `whereNumber('id')` — нечисловой id отсекается роутером (404) ещё до контроллера, поэтому сигнатура `int $id` безопасна.

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
- Проверяются только **форма и границы**. Ни `exists:points,id`, ни `unique:chargers,remote_code` — это делает сервис под блокировкой.
- **legacy → как надо:** это многополевая админ-форма, поэтому по политике валидации **убрать** `$stopOnFirstFailure` и `bail`,
  чтобы фронт получил все ошибки сразу.

### Контроллер
`app/Http/Controllers/ChargerController.php`
```php
public function store(StoreChargerRequest $request): JsonResponse
{
    return response()->json($this->chargerService->create($request->validated()));
}
```
- Одна строка: данные → сервис → ответ.
- **legacy → как надо:**
  ```php
  public function store(StoreChargerRequest $request): ChargerResource
  {
      return ChargerResource::make($this->chargerService->create($request->validated()));
  }
  ```

### Сервис
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
- Три проверки «можно ли» → одна запись, всё внутри `transaction()` и с `lockForUpdate` (второй аргумент `true`).
  Два одновременных запроса не создадут 101-й зарядник и не создадут дубль `remote_code`.
- Каждое нарушение — свой тип исключения → свой статус и текст.
- **legacy → как надо:** `>= 100` → `>= Charger::MAX_PER_POINT` (константа модели).

### Репозиторий
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
- `withTrashed()`: в миграции у `remote_code` стоит `unique()`, а мягко удалённая строка по-прежнему занимает значение.
  Проверка учитывает это так же, как это делает БД.
- Репозиторий возвращает `bool`/модель и ничего не решает.

### ❌ Как НЕ надо
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
Почему отклоняется:
- логика и Eloquent в контроллере — правило не переиспользовать из других точек входа;
- `unique`/`exists` и подсчёт лимита — вне транзакции и блокировки: два параллельных запроса пройдут оба;
- у полей нет границ (`max`), формат ошибки свой (`error` вместо `message`), текст не переведён, статус не по таблице;
- наружу отдаётся сырая модель.

---

## 2. Сервис с бизнес-логикой: старт зарядки

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

Разбор по блокам:
- **Публичный метод читается как сценарий**: достать и проверить коннектор → лимит сессий → пользователь → цена → баланс →
  списание → сессия → команда зарядному устройству → пуш. Детали скрыты в `getAndValidate…`/`validate…`.
- **`$authUserId` — аргумент**, а не `auth()->id()`: тот же сервис можно вызвать без HTTP.
- **Порядок блокировок**: сначала коннектор (общий ресурс), потом сессии пользователя, потом сам пользователь.
- **Цена фиксируется в сессии** (`price_per_kw`, `watts` копируются из коннектора): последующее изменение тарифа
  не затрагивает идущую зарядку.
- **Аванс за интервал** (`calculateIntervalCost` → `ChargingSession::getIntervalCost`): формула живёт в модели, сервис её только вызывает.
- **Синхронная внешняя команда внутри транзакции** — осознанное исключение: если зарядник не принял `RemoteStartTransaction`,
  исключение откатывает списание и создание сессии. Пуш в приложение — `defer` (внутри `pushActiveSessions`), он уйдёт после коммита.
- **Log-and-rethrow** снаружи транзакции: инцидент с деньгами виден в канале `charging` с ID, а клиент получает обычный доменный ответ.
- **legacy → как надо:** `$connector->status === ConnectorStatus::OFFLINE` → сравнение по `->value`.

### ❌ Как НЕ надо
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
- `auth()`/`abort()` в сервисе — привязка к HTTP;
- нет транзакции и блокировок: два параллельных старта спишут деньги дважды и займут один коннектор;
- Eloquent и HTTP прямо в сервисе; ошибка шлюза не обработана, списание не откатывается;
- формула стоимости написана инлайн — не та, что в модели;
- `Connector::find` без select и с ленивой загрузкой `->charger`.

---

## 3. Обработка ошибки: от `throw` до JSON

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
`lang/ru/all.php`
```php
'charging_session_not_found' => 'Зарядная сессия не найдена',
```
Ответ: `404 {"message": "Зарядная сессия не найдена"}`. Замысел — язык из `Accept-Language`; но локаль выставляется
один раз в `AppServiceProvider::boot()`, поэтому под Octane, скорее всего, язык сообщений не меняется от запроса к запросу
(см. testing.md → «Локаль»).

Разбор:
- «Нет» и «чужая» — **одна ветка и один ответ**: клиент не может перебором id узнать, какие сессии существуют. [гипотеза]
- Сервис бросает тип, ничего не зная о статусе и тексте. Поменять код или текст можно в одном месте.
- `reportable` возвращает `false` для замапленных исключений → ожидаемые 404 не попадают в лог.
- Репозиторный `findById` возвращает `?ChargingSession`; решение «это ошибка» принимает сервис.

### ❌ Как НЕ надо
```php
$session = ChargingSession::findOrFail($id);          // ModelNotFoundException → стандартный 404 без перевода
if ($session->user_id !== auth()->id()) {
    abort(403, 'Forbidden');                           // раскрывает, что сессия существует; abort в сервисе
}
throw new \Exception('Session not found');             // 500, текст не переведён, лог засоряется
```

---

## 4. Транзакция, блокировки и возврат денег: остановка сессии

`app/Actions/StopChargingSessionAction.php`
```php
public function execute(int $id, User $authUser, bool $isHardwareInitiated = false, ?int $meterStop = null): ChargingSession
{
    try {
        return transaction(function () use ($id, $authUser, $isHardwareInitiated, $meterStop) {
            $session = $this->getAndValidateSessionForStop($id, $authUser->id);   // findById(..., true) + проверка статуса

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
            // ... transaction record, session update, notification, remote stop, pushes (все внешние — через defer)

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

Разбор:
- **Action, а не метод сервиса:** остановку вызывают пользователь (HTTP), зарядник (OCPP `StopTransaction`/`StatusNotification`)
  и cron (низкий баланс, зависшие pending). Флаг `$isHardwareInitiated` говорит, нужно ли слать `RemoteStopTransaction` обратно.
- **Сессия и пользователь блокируются** (`findById(..., true)`): параллельный стоп из приложения и от зарядника
  не вернёт деньги дважды; второй вызов получит `InvalidSessionStateException`.
- **Двойной источник истины для энергии:** счётчик зарядника (`meterStop - meter_start`), если он есть и правдоподобен,
  иначе — оценка по времени. Решение о fallback принимается в одном месте.
- **Каждое промежуточное значение денег округлено** `round(..., 2)`.
- **Pending → CANCELLED с полным возвратом** — отдельная ветка, без транзакции списания.
- **legacy → как надо:** метод должен называться `handle`; сравнение статуса — через `->value`.

### ❌ Как НЕ надо
```php
$session->update(['status' => 'completed', 'finished_at' => now()]);
$user->update(['balance' => $user->balance + $refund]);    // без транзакции: упадёт между двумя update → деньги потеряны
FirebasePush::toToken($user->fcm_token)->send();           // синхронно, внутри сценария: FCM лёг → стоп не прошёл
```

---

## 5. Репозиторий-шлюз и `defer()`

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

Разбор:
- Сервис вызывает `pushUserUpdated($user)` и не знает, что за этим стоят HTTP, Node-шлюз и WebSocket.
- **Снимок данных до `defer`** (`resolve()` снаружи): в замыкание уходит готовый массив — без ленивых загрузок
  после окончания запроса и без риска, что модель изменится до отправки.
- `defer` — после ответа клиенту: бизнес-операция не ждёт сеть, отдельный воркер очередей не нужен (минимум инфраструктуры).
- Ошибка шлюза → лог + `null`; операция не падает. `timeout(5)` ограничивает время, на которое занят воркер Swoole.

### ❌ Как НЕ надо
```php
// в сервисе
Http::post(config('custom.ws_server_url').'/broadcast', ['data' => $user]);   // синхронно, в сервисе, сырая модель
dispatch(new SendMobileCommandJob($user));                                      // очередь ради одного HTTP-вызова — лишняя инфраструктура
defer(fn () => $this->send(UserResource::make($user)->resolve()));              // сериализация внутри defer — ленивые загрузки/устаревшее состояние
```

---

## 6. Оптимизированный запрос средствами MySQL

`app/Repositories/PointRepository.php`
```php
public function getInRadius(float $latitude, float $longitude, float $radius): Collection
{
    $latOffset = $radius / 111320;
    $lngOffset = $radius / (111320 * max(abs(cos(deg2rad($latitude))), 0.0001));

    $bbox = sprintf('POLYGON((%F %F, %F %F, %F %F, %F %F, %F %F))', /* углы квадрата */);

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
плюс в сервисе: `$radius = min(Point::MAX_RADIUS, $radius);`

Разбор:
- **Двухступенчатый фильтр:** `MBRContains` по bounding box использует spatial-индекс и дёшево отсекает почти всё,
  точный `ST_Distance_Sphere` считается только для оставшихся строк.
- `max(..., 0.0001)` защищает от деления на ноль у полюсов.
- **Жёсткие потолки:** `MAX_RADIUS` в сервисе и `limit(1000)` в запросе — один запрос не может выгрузить всю таблицу.
- Все пользовательские значения — через биндинги `?`.
- `withPrice()` — подзапрос в select вместо загрузки коннекторов ради одной цены.
- PostGIS, Elastic и гео-сервисы не нужны (минимум инфраструктуры).

### ❌ Как НЕ надо
```php
return Point::all()->filter(fn ($p) => haversine($p, $lat, $lng) <= $radius);   // вся таблица в память, все колонки, N+1 на переводах
```

---

## 7. FormRequest: форма и служебный запрос

Служебный запрос — **с** остановкой на первой ошибке:
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

Многополевая форма — **без** `bail` и `$stopOnFirstFailure` (целевой вид; сейчас в проекте стоит `$stopOnFirstFailure = true`, это legacy):
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
- Админ видит все неверные поля формы за один запрос.
- Границы есть у каждого поля в обоих вариантах.

### ❌ Как НЕ надо
```php
'company_id' => 'required|exists:companies,id',   // проверка существования вне транзакции; строковый синтаксис; нет границ
'name' => 'required',                               // нет типа и max → в БД уйдёт что угодно
```

---

## 8. Фиксированные значения: enum вместо сырых строк

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

Как надо — набор значений и правило о нём живут в enum:
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

Разбор:
- **Одна точка истины:** список «кабель вставлен» и «сессию пора остановить» объявлен один раз, в enum, рядом со значениями.
- **Новое значение = новый `case`.** Миграция, ALTER и поиск строки по коду не нужны (enum в коде, колонка `string`).
- **Опечатка ловится статически:** `OCPPConnectorStatus::CHARGNG` — ошибка Larastan, а `'Chargng'` молча вернёт `false`.
- **Неизвестное значение от железа обрабатывается явно** (`tryFrom` → `null` → ранний return), а не уходит в «ни одна ветка не сработала».
- Сравнение — по `->value` (правило автора).

### ❌ Как НЕ надо
```php
if ($user->role == 'admin') { ... }                        // сырая строка, нестрогое сравнение, нет каста
$table->enum('status', ['pending', 'charging']);           // набор значений в БД: новое значение = ALTER на проде
'event' => 'UserUpdated'                                    // имя события строкой в трёх местах
```

---

<a id="example-test"></a>
## 9. Feature-тест (эталон — в проекте тестов пока нет)

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

Разбор:
- **Деньги проверяются точной суммой** (`98.5` = 1 интервал = 60 кВт × 60 с / 3600 = 1 кВт·ч × 1.5), а не «баланс уменьшился».
- **Каждое исключение сервиса — отдельный тест** со статусом из таблицы и текстом из `lang` через `__()`, а не строкой.
- **Откат транзакции проверяется явно**: при отказе зарядника баланс и таблица сессий не меняются.
- **Внешние системы — только через `Http::fake`**; `preventStrayRequests` в базовом `TestCase` не даст тесту уйти в сеть.
  `Http::assertSent` работает, потому что `defer` выполняется на `terminate` HTTP-теста.
- Связи создаются явно (`for($connector)`), без опоры на случайные строки из БД.
- `number` и `ConnectorType` задаются явно: `ConnectorFactory` по умолчанию генерирует `number` вида `Connector-##`
  (а `remoteStartTransaction` ждёт `int`) и подставляет `connector_type_id = 1`, если в БД нет типов.
- (`Charger::factory()` потребует `HasFactory` + factory для `Charger` — в проекте её пока нет.)

### ❌ Как НЕ надо
```php
public function test_it_works(): void
{
    $this->seed();                                                      // зависимость от сидов и случайных данных
    $user = User::first();
    $this->actingAs($user)->post('/api/charging-sessions', ['connector_id' => 1])->assertStatus(200);
    $this->assertTrue(User::first()->balance < $user->balance);          // не проверяет сумму
}
```
Плюс мок каждого репозитория в «юнит-тесте» сервиса — проверяет моки, а не поведение.
