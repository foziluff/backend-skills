# Шаблоны

Шаблоны готовы к копированию: заменить плейсхолдеры и пройти комментарии `// РЕШЕНИЕ:`, после чего удалить эти комментарии
(в итоговом коде проекта комментариев почти нет).

## Плейсхолдеры

| Плейсхолдер | Пример (`Tariff`, принадлежит `Point`) |
|---|---|
| `{{EntityName}}` | `Tariff` |
| `{{entityName}}` | `tariff` (camelCase: переменные, свойства) |
| `{{entity_name}}` | `tariff` (snake_case: ключи lang, поля логов) |
| `{{entity_names}}` | `tariffs` (таблица) |
| `{{entity-names}}` | `tariffs` (URL, kebab-case во множественном числе: `connector-types`) |
| `{{ParentName}}` / `{{parentName}}` / `{{parent_name}}` / `{{PARENT_NAME}}` | `Point` / `point` / `point` / `POINT` |
| `{{parent_id}}` | `point_id` |
| `{{Verb}}` / `{{verb}}` | `Stop` / `stop` |
| `{{Problem}}` / `{{problem}}` | `LimitReached` / `limit_reached` |
| `{{YYYY_MM_DD_HHMMSS}}` | `2026_10_20_120000` |

## Порядок создания нового CRUD

| # | Файл | Шаблон |
|---|---|---|
| 1 | `database/migrations/…_create_{{entity_names}}_table.php` | [migration.stub](migration.stub) |
| 2 | `app/Models/{{EntityName}}.php` (+ enum в `app/Enums`, если есть статус) | [model.stub](model.stub) |
| 3 | `database/factories/{{EntityName}}Factory.php` | [factory.stub](factory.stub) |
| 4 | `database/seeders/{{EntityName}}Seeder.php` + регистрация в `DatabaseSeeder::run()` в порядке зависимостей | — (по образцу `CompanySeeder`) |
| 5 | `app/Exceptions/{{EntityName}}NotFoundException.php` (+ бизнес-ошибки) | [exception.stub](exception.stub) |
| 6 | Строки в `bootstrap/app.php` + ключи в `lang/ru`, `lang/tg`, `lang/en` | [exception-mapping.stub](exception-mapping.stub) |
| 7 | `app/Repositories/{{EntityName}}Repository.php` | [repository.stub](repository.stub) |
| 8 | `app/Services/{{EntityName}}Service.php` | [service.stub](service.stub) |
| 9 | `app/Actions/…Action.php` — только если операцию вызывают ≥2 сервиса/точки входа | [action.stub](action.stub) |
| 10 | `app/Http/Requests/{{EntityName}}/Store…`, `Update…`, `Index…` | [request-store.stub](request-store.stub), [request-update.stub](request-update.stub), [request-index.stub](request-index.stub) |
| 11 | `app/Http/Resources/Client/{{EntityName}}Resource.php` | [resource.stub](resource.stub) |
| 12 | `app/Http/Controllers/{{EntityName}}Controller.php` | [controller.stub](controller.stub) |
| 13 | `routes/api.php` | [routes.stub](routes.stub) |
| 14 | `tests/Feature/{{EntityName}}/…Test.php`, `tests/Unit/Models/{{EntityName}}Test.php` | [feature-test.stub](feature-test.stub), [unit-test.stub](unit-test.stub) |
| 15 | Новые env-настройки → `config/custom.php` + блок `#CUSTOM#` в `.env.example` | — |

Тестовое окружение (один раз на проект): [phpunit.xml.stub](phpunit.xml.stub), [test-case.stub](test-case.stub),
шаги — в [references/testing.md](../references/testing.md).

## Варианты, которые меняют шаблон

**Сущность с переводами** (`name`, `address`…):
- в миграции переводимых колонок нет;
- в модели `use HasTranslations`, `protected array $translatable = [...]`, переводимые поля перечислены в `DEFAULT_SELECT_COLUMNS`;
- в FormRequest — блок `translations.*` (`language_code` — `required|distinct|size:2`);
- в сервисе — проверка существования языков (одна общая реализация, см. [data-layer.md](../references/data-layer.md#translations));
- в репозитории `create`/`update` — своя `transaction()`, `syncTranslations($data)` и перечитывание через `findById`;
- для админки — чтение с `->with('translations')` (полный список языков) и отдельный Resource.

**Публичное чтение для мобильного приложения:** маршрут вне `auth:sanctum`, Resource без служебных полей,
жёсткие лимиты выборки (`limit`, константа `MAX_*`).

**Действие пользователя над своим ресурсом:** `auth()->id()` в контроллере → аргумент сервиса → ownership-проверка
в сервисе с ответом `NotFound`.

**Денежная операция:** `transaction()` + `lockForUpdate` пользователя, `round(..., 2)`, запись в `transactions`,
log-and-rethrow в канал `charging`, пуши — через `MobileCommandRepository` (внутри `defer`).

## После генерации

```bash
php vendor/bin/pint
php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon
php artisan test
php artisan swagger      # необязательно локально (CI делает сам); проверить, что новые эндпоинты и коды ответов попали в docs
```
