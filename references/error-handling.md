# Ошибки, логирование, API-ответы

## Доменные исключения

### Класс

```php
// app/Exceptions/ConnectorBusyException.php
<?php

namespace App\Exceptions;

use RuntimeException;

class ConnectorBusyException extends RuntimeException {}
```

- Пустое тело, наследование от `RuntimeException`, без сообщения и кода.
- Один класс — одна бизнес-ситуация. Общие `BadRequestException`/`TooManyRequestsException` — только там,
  где детализация намеренно скрыта (неверный OTP: `UserService::verifyPhone` не сообщает,
  что именно не так — код, срок или число попыток). [гипотеза: защита от перебора]

*Почему пустые:* текст зависит от языка клиента, а статус — от HTTP-контракта; ни то ни другое
не является знанием сервиса. [гипотеза]

### Где бросаются

Только в **сервисе** или бизнес-**Action**. Репозиторий возвращает `null`/`false`, контроллер не проверяет ничего.

```php
// app/Services/PointService.php
public function delete(int $id): void
{
    if (! $this->pointRepository->softDelete($id)) {
        throw new PointNotFoundException;
    }
}
```

### Маппинг в HTTP — одно место

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
            return false;            // ожидаемые бизнес-ошибки не засоряют лог
        }
    });

    $exceptions->render(function (Throwable $e) {
        $customResponse = mapExceptionToResponseData($e);
        if ($customResponse === null) {
            return null;             // всё остальное — стандартная обработка Laravel
        }
        [$status, $message] = $customResponse;

        return response()->json(['message' => $message], $status);
    });
```

**Добавление новой ошибки — три обязательных шага:**
1. Класс в `app/Exceptions/`.
2. Строка в `match` в `bootstrap/app.php` (статус по таблице ниже).
3. Ключ `snake_case` в **трёх** файлах: `lang/ru/all.php`, `lang/tg/all.php`, `lang/en/all.php`.

Формат строки в `match` фиксирован — `$e instanceof XxxException => [Response::HTTP_XXX, __('all.xxx')],` на одной строке:
генератор Swagger читает статусы из `bootstrap/app.php` регуляркой (см. [architecture.md](architecture.md#swagger)).

Если пропустить шаг 2, клиент получит 500 и запись в лог; если пропустить шаг 3 — сырой ключ `all.xxx` в ответе.

> Язык сообщения определяется локалью приложения. Она выставляется в `AppServiceProvider::boot()` через
> `app()->setLocale(accept_language())` — один раз при загрузке приложения. Под Octane провайдеры не перезагружаются на каждый
> запрос, а `FlushLocaleState` сбрасывает локаль к `config('app.locale')`, поэтому язык ошибок, скорее всего,
> не следует `Accept-Language` (переводы **данных** работают: `accept_language()` читает заголовок на каждом запросе).
> Новый код на это не полагается; правка — в списке улучшений проекта.

### Выбор HTTP-статуса (по факту проекта)

| Ситуация | Статус | Примеры |
|---|---|---|
| Сущности нет / она чужая | `404 HTTP_NOT_FOUND` | `PointNotFoundException`, `ChargingSessionNotFoundException` (в т. ч. чужая сессия) |
| Нарушено бизнес-правило / лимит / недопустимое состояние | `422 HTTP_UNPROCESSABLE_ENTITY` | `ChargerLimitReachedException`, `ChargerAlreadyExistsException`, `InvalidSessionStateException`, `ConnectorOfflineException`, `OCPPCommandFailedException` |
| Конфликт с текущим состоянием ресурса | `409 HTTP_CONFLICT` | `ConnectorBusyException`, `ActiveSessionExistsException` |
| Не хватает денег | `402 HTTP_PAYMENT_REQUIRED` | `InsufficientFundsException` |
| Не выполнено физическое предусловие | `412 HTTP_PRECONDITION_FAILED` | `ConnectorNotConnectedException` (кабель не вставлен) |
| Неизвестный зарядник на OCPP-эндпоинте | `401 HTTP_UNAUTHORIZED` | `ChargerRejectedException` |
| Слишком часто | `429` | `TooManyRequestsException`, `ThrottleRequestsException` |
| Неверный запрос без деталей | `400` | `BadRequestException` |
| Нет роли | `403` | `RoleMiddleware` → `abort(Response::HTTP_FORBIDDEN)` |

*Почему разные коды:* мобильный клиент ветвится по статусу (402 → экран пополнения, 412 → «вставьте кабель»,
409 → «коннектор занят»), не разбирая текст. [гипотеза]

`abort()` допустим **только в middleware** (`RoleMiddleware`, `OCPPMiddleware`) — это уровень HTTP.

<a id="api-contract"></a>
## Формат ответов

| Случай | Тело | Как получить |
|---|---|---|
| Бизнес-ошибка | `{"message": "Коннектор уже занят"}` | доменное исключение |
| Ошибка валидации | стандартный Laravel 422: `{"message": "...", "errors": {"field": ["..."]}}` | FormRequest |
| Throttle | `{"retry_after": 60}` + 429 | отдельный `render` для `ThrottleRequestsException` |
| Объект | `{...}` **без** обёртки `data` | `XxxResource::make($model)` (в `AppServiceProvider` — `JsonResource::withoutWrapping()`) |
| Список без пагинации | `[...]` | `XxxResource::collection($items)` |
| Cursor-пагинация | `{"data": [...], "next_cursor": "..."\|null}` | `CursorPaginationResource::make($paginator, XxxResource::class)` |
| Простая пагинация (поиск) | `{"data": [...], "next_page_url": "..."\|null}` | `LengthAwarePaginationResource::make($paginator, XxxResource::class)` |
| Пусто (действие выполнено) | `[]` | `EmptyResource::make()` |
| Токен | `{"token": "..."}` | `TokenResource::make($plainTextToken)` |

Правила:
- **Всегда Resource**, в том числе в админке и для пустых ответов. **[подтверждено]**
  `response()->json($model)` и `response()->json()` в `PointController`, `CompanyController`, `ChargerController` и т. д. — legacy.
- **Сначала select, потом Resource.** Resource только формирует контракт; лишние колонки отсекаются уже в запросе
  (`DEFAULT_SELECT_COLUMNS`). **[подтверждено]**
- В Resource — явные приведения: `(int) $this->watts`, `(float) $this->balance`, `(bool) $this->is_online`,
  nullable — `$this->price !== null ? (float) $this->price : null`. *Почему:* decimal из MySQL приходит строкой,
  а клиенту нужен стабильный тип.
- Связи в Resource — через `whenLoaded('relation')`, никогда не загружать в Resource.
- Даты отдаются как есть (Carbon → ISO-8601).
- **Статус успеха** выставляет Laravel: Resource поверх модели с `wasRecentlyCreated = true` → `201`, иначе `200`.
  Поэтому `create` в репозитории, который возвращает модель из `Model::create()`, даёт `201`
  (`POST /charging-sessions`), а `create`, который перечитывает модель через `findById` (`PointRepository::create`), — `200`.
  При переводе legacy-эндпоинтов с `response()->json()` на Resource проверить, что клиент не завязан на конкретный код 200/201.
- Пагинация — cursor по 20 (`cursorPaginate(20)`). *Почему:* бесконечные ленты в мобильном приложении,
  нет `COUNT(*)`, стабильность при вставках. `simplePaginate` — только когда нужна сортировка по вычисляемой
  релевантности (`PointRepository::search`), где cursor невозможен.
- Для админки, если форма ответа отличается от клиентской (например, нужен полный массив `translations`), —
  отдельный Resource. Предлагаемое место: `app/Http/Resources/Admin/{Entity}Resource.php`.
  **[предложение — в проекте пока нет, подтвердить]**

## Логирование

### Каналы

- `charging` — `daily`, 14 дней (`config/logging.php`). Всё, что касается денег и сессий зарядки.
- Канал по умолчанию — входящие OCPP-сообщения (`Log::info("OCPP [{$remoteCode}] {$action}", $payload)`),
  ошибки внешних шлюзов (`SendWebsocketRequestAction`).

### Что и как

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

- **Log-and-rethrow** на границе денежной операции: снаружи `transaction()`, с контекстом идентификаторов и классом
  исключения, после чего исключение пробрасывается (обработчик превратит его в ответ). [гипотеза: разбор
  денежных инцидентов по `user_id`/`session_id` без включения отчётов для ожидаемых бизнес-ошибок]
- Сообщения — на английском, в формате `"Failed to <action>: ".$e->getMessage()`; ID — в контексте или в тексте.
- Уровни:
  - `error` — операция не выполнена;
  - `warning` — сработал fallback (`Bulk progress update failed, falling back to individual processing`) или
    внешняя сторона отклонила команду (`[OCPP Command] {$action} rejected`);
  - `info` — значимое бизнес-событие (`Session {id} stopped due to charger offline timeout`).
- Префикс подсистемы в квадратных скобках: `[OCPP Command]`, `[Mobile Command]`.
- **Не логировать** персональные данные сверх ID (телефон, OTP-код, токены).
- Отладочные `Log::info('DEFER EXECUTED…')` в коммит не попадают (legacy в `MobileCommandRepository::pushBulkActiveSessions`).

### Где глотать исключения

Глотать можно **только явно перечисленные ожидаемые** исключения и только в фоновых потоках, где
состояние могло измениться конкурентно:

```php
// ChargingSessionService::autoUpdateProgress — зависшие pending-сессии
try {
    $this->stopChargingSessionAction->execute($session->id, $session->user, false);
} catch (InvalidSessionStateException|ChargingSessionNotFoundException $e) {
}
```

В batch-цикле — `catch (Throwable $e)` с логированием, чтобы одна сломанная сессия не останавливала пачку.

Ошибка внешнего шлюза (WS, SMS, FCM) не должна ронять бизнес-операцию: результат — `null`/`false` + лог.
(legacy: `catch (Exception) {}` без лога в `NotificationRepository::sendSms` — нужно логировать.)
