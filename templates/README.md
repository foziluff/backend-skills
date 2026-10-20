# Templates

Templates are ready to copy: replace the placeholders, go through every `// DECISION:` comment, then delete those comments
(production code in the project has almost no comments).

## Placeholders

| Placeholder | Example (`Tariff`, belongs to `Point`) |
|---|---|
| `{{EntityName}}` | `Tariff` |
| `{{entityName}}` | `tariff` (camelCase: variables, properties) |
| `{{entity_name}}` | `tariff` (snake_case: lang keys, log fields) |
| `{{entity_names}}` | `tariffs` (table) |
| `{{entity-names}}` | `tariffs` (URL, kebab-case plural: `connector-types`) |
| `{{ParentName}}` / `{{parentName}}` / `{{parent_name}}` / `{{PARENT_NAME}}` | `Point` / `point` / `point` / `POINT` |
| `{{parent_id}}` | `point_id` |
| `{{Verb}}` / `{{verb}}` | `Stop` / `stop` |
| `{{Problem}}` / `{{problem}}` | `LimitReached` / `limit_reached` |
| `{{Aspect}}` / `{{aspect}}` | `Status` / `status` |
| `{{YYYY_MM_DD_HHMMSS}}` | `2026_10_20_120000` |
| `{{Name}}` / `{{secret_key}}` / `{{Header-Name}}` | `Gateway` / `ws_secret` / `Ws-Secret` (secret-middleware only) |

## Order for a new CRUD

| # | File | Template |
|---|---|---|
| 1 | `database/migrations/…_create_{{entity_names}}_table.php` (+ an index for every planned query) | [migration.stub](migration.stub) |
| 2 | `app/Enums/{{EntityName}}{{Aspect}}.php` — for **every** field with a fixed set of values (status, type, role) | [enum.stub](enum.stub) |
| 3 | `app/Models/{{EntityName}}.php` | [model.stub](model.stub) |
| 4 | `database/factories/{{EntityName}}Factory.php` | [factory.stub](factory.stub) |
| 5 | `database/seeders/{{EntityName}}Seeder.php` + registration in `DatabaseSeeder::run()` in dependency order | — (follow `CompanySeeder`) |
| 6 | `app/Exceptions/{{EntityName}}NotFoundException.php` (+ business errors) | [exception.stub](exception.stub) |
| 7 | Lines in `bootstrap/app.php` + keys in `lang/ru`, `lang/tg`, `lang/en` | [exception-mapping.stub](exception-mapping.stub) |
| 8 | `app/Repositories/{{EntityName}}Repository.php` — the only place for its queries; check each with `EXPLAIN` | [repository.stub](repository.stub) |
| 9 | `app/Services/{{EntityName}}Service.php` | [service.stub](service.stub) |
| 10 | `app/Actions/…Action.php` — only if the operation is called by ≥2 services/entry points | [action.stub](action.stub) |
| 11 | `app/Http/Requests/{{EntityName}}/Store…`, `Update…`, `Index…` | [request-store.stub](request-store.stub), [request-update.stub](request-update.stub), [request-index.stub](request-index.stub) |
| 12 | `app/Http/Resources/Client/{{EntityName}}Resource.php` | [resource.stub](resource.stub) |
| 13 | `app/Http/Controllers/{{EntityName}}Controller.php` | [controller.stub](controller.stub) |
| 14 | `routes/api.php` (right group, throttle where needed) | [routes.stub](routes.stub) |
| 15 | `tests/Feature/{{EntityName}}/…Test.php`, `tests/Unit/Models/{{EntityName}}Test.php` | [feature-test.stub](feature-test.stub), [unit-test.stub](unit-test.stub) |
| 16 | New env settings → `config/custom.php` + the `#CUSTOM#` block of `.env.example` (empty values for secrets) | — |

Service-to-service endpoint protected by a shared secret: [secret-middleware.stub](secret-middleware.stub).
Test environment (once per project): [phpunit.xml.stub](phpunit.xml.stub), [test-case.stub](test-case.stub),
steps in [references/testing.md](../references/testing.md).

## Variants that change the template

**Entity with translations** (`name`, `address`…):
- no translatable columns in the migration;
- in the model: `use HasTranslations`, `protected array $translatable = [...]`, translatable fields listed in `DEFAULT_SELECT_COLUMNS`;
- in the FormRequest: a `translations.*` block (`language_code` — `required|distinct|size:2`);
- in the service: a language-existence check (one shared implementation, see [data-layer.md](../references/data-layer.md#translations));
- in the repository `create`/`update`: its own `transaction()`, `syncTranslations($data)` and a re-read via `findById`;
- for the admin panel: read with `->with('translations')` (all languages) and a separate Resource.

**Public read for the mobile app:** route outside `auth:sanctum` with `throttle`, a Resource without internal fields,
hard read caps (`limit`, a `MAX_*` constant), an index that serves the query.

**A user action on their own resource:** `auth()->id()` in the controller → service argument → ownership check in the service
answering `NotFound`; a test that another user's id returns 404.

**Money operation:** `transaction()` + `lockForUpdate` of the user, `round(..., 2)`, a record in `transactions` in the same
transaction as the balance change, log-and-rethrow to the `charging` channel, pushes via `MobileCommandRepository` (inside `defer`).

**A stored derived value (denormalization) — only if unavoidable:** one write path, the same transaction as the source of truth,
a row lock, reconstructible + a reconciliation check ([performance.md](../references/performance.md#denormalization)).

## After generating

```bash
php vendor/bin/pint
php vendor/bin/phpstan analyse app config database routes bootstrap lang --level=5 -c vendor/larastan/larastan/extension.neon
php artisan test
php artisan swagger      # optional locally (CI does it); check the new endpoints and response codes made it into the docs
```
