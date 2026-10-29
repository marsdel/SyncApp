---
name: crear-controlador
description: Create an HTTP Controller in the RPSync Laravel backend. Use when asked to "crear un controlador", "nuevo controller", "agregar un endpoint", or when exposing a Service over the API. Covers make:clean-arch, Service injection, FormRequests, the try/catch + response()->json contract and controller traits.
---

# Create a Controller in RPSync

A controller does three things: receive a validated request, call its Service, and
shape the HTTP response. Nothing else. Source of truth:
`docs/Coding Styles.md#cómo-hacer-controladores`.

## 1. Generate

```bash
php artisan make:clean-arch <Modulo>/<Entidad>
```

Creates the whole slice:

| File | Path |
| --- | --- |
| Interface | `app/Interfaces/<Modulo>/Repository/I<Entidad>Repository.php` |
| Repository | `app/Repositories/<Modulo>/<Entidad>Repository.php` |
| Service | `app/Services/<Modulo>/<Entidad>Service.php` |
| Controller | `app/Http/Controllers/<Modulo>/<Entidad>Controller.php` |

> **Two things the template gets wrong for current standards — fix them after
> generating:**
> 1. The controller template does `$this->service = new <Entidad>Service(new <Entidad>Repository)`.
>    Replace it with constructor property promotion of the Service so the container
>    resolves the repository binding.
> 2. The repository template `implements` the interface but does not extend
>    `ConnectableRepository`. Add the `extends`.
>
> Also note the generator flattens to `<Modulo>/<Entidad>`; most modules use a
> `<Modulo>/<Seccion>/` sub-namespace (`Compras/Transacciones/…`). Move the files if
> the module is organized that way.

## 2. Shape

```php
<?php

namespace App\Http\Controllers\<Modulo>\<Seccion>;

use App\Exceptions\<Modulo>\<Entidad>AlreadyExists;
use App\Http\Controllers\Controller;
use App\Http\Requests\<Modulo>\<Seccion>\Index<Entidad>Request;
use App\Http\Requests\<Modulo>\<Seccion>\Store<Entidad>Request;
use App\Services\<Modulo>\<Seccion>\<Entidad>Service;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Support\Facades\Log;
use Throwable;

class <Entidad>Controller extends Controller
{
    public function __construct(protected <Entidad>Service $service) {}

    public function index(Index<Entidad>Request $request)
    {
        try {
            return response()->json(['data' => $this->service->search($request->validated())], 200);
        } catch (Throwable $th) {
            Log::error($th);

            return response()->json(['message' => $th->getMessage()], 500);
        }
    }

    public function store(Store<Entidad>Request $request)
    {
        try {
            return response()->json(['data' => $this->service->create($request->validated())], 201);
        } catch (<Entidad>AlreadyExists $e) {
            return response()->json(['message' => $e->getMessage()], 422);
        } catch (ModelNotFoundException $e) {
            return response()->json(['message' => $e->getMessage()], 404);
        } catch (Throwable $th) {
            Log::error($th);

            return response()->json(['message' => $th->getMessage()], 500);
        }
    }
}
```

Reference: `app/Http/Controllers/Compras/Transacciones/OrdenEntregaController.php`.

## 3. Hard rules

- **Inject the Service by constructor property promotion.** A new controller must not
  import `App\Models\...` nor build Eloquent queries. This holds even when you are
  adding an endpoint inside a legacy module whose other 15,000 lines do exactly that —
  new code follows Controller → Service → Repository regardless of its neighbours.
- **All input validation lives in a `FormRequest`** (`app/Http/Requests/<Modulo>/…`,
  named `Store<Entidad>Request`, `Update<Entidad>Request`, `Index<Entidad>Request`).
  Type it as the action parameter and use `$request->validated()`. There is not a
  single inline `$request->validate([...])` in the project — do not be the first.
  Reusable rules go in `app/Rules/` (`ValidationRule` implementations); rule sets
  shared across requests of one domain go in a trait.
- **`booleable` validates but does not cast.** `"false"`/`"off"` are truthy in PHP, so
  `paginated=false` reads as `true`. Cast each such field with `to_boolean()` in
  `prepareForValidation()` — [`.ai/rules/requests.md`](../../../.ai/rules/requests.md).
- **Error handling: one `try`/`catch` per action.** Catch specific domain exceptions
  first with a precise HTTP status, then a final `catch (Throwable $th)` that does
  `Log::error($th)` and returns a 500 JSON. Do not let an uncaught exception reach the
  global handler without context.
- **Response format:** `response()->json([...], $status)` with `'data'` for success
  payloads and `'message'` for errors. There is no project-wide `JsonResource`
  convention — a `JsonResource` is acceptable when you genuinely need to reshape the
  response, but do not introduce it as a requirement if the module does not use it.

## 4. Base class, traits and context attributes

- `App\Http\Controllers\Controller` is intentionally almost empty. Do **not** add
  generic utility methods to it.
- Shared controller behaviour goes in a trait under `app/Traits/Controllers/`
  (`HasAuthData`, `HasGuard`, `HasInjectableEmpresa`, `CanThrowEmergency`), pulled in
  explicitly with `use`.
- To get the current user / connection, prefer the container attributes in
  `app/Container/Attributes/` — `#[CurrentUser]`, `#[CurrentConnection]`,
  `#[CurrentConnectionName]` — over calling the `user()` / `nconn()` helpers inline.
  The mechanism is real but still lightly adopted; new code should use it absent a
  reason not to.
- Some `Compras`/`ContabilidadPg` controllers set a `trace_id` in the constructor
  (`context(['trace_id' => static::createTraceId()])`) and append it to error messages
  so support can correlate a user-reported error with the log. Worth replicating for
  transaction-heavy controllers.

## 5. Wiring

The controller is not reachable until its route exists — see the `crear-rutas` skill.
Routes for new endpoints go in `routes/api/<modulo>.php` (or
`routes/api/v2/resource/<modulo>.php`), never appended to the historical
`routes/api.php`.

If the endpoint must be permission-protected, attach
`->middleware('acceso.permiso:<action>,<resource>')` with strings identical to the
frontend permission tree — see `crear-permiso`.

## 6. Checklist

- [ ] File at `app/Http/Controllers/<Modulo>/<Seccion>/<Entidad>Controller.php`,
      module folder consistent with `Services/`, `Repositories/`, `Interfaces/`,
      `Models/` and `routes/api/<modulo>.php`.
- [ ] Service injected via constructor property promotion; no `new Service(...)`.
- [ ] No `App\Models\...` import, no Eloquent, no `DB::` in the controller.
- [ ] Every action takes a `FormRequest` and uses `validated()`.
- [ ] Every `booleable` field is cast with `to_boolean()` in `prepareForValidation()`.
- [ ] `try`/`catch` per action: domain exceptions first, `Throwable` + `Log::error`
      last, JSON 500.
- [ ] Success responses use `['data' => ...]`; errors use `['message' => ...]`.
- [ ] Route registered and, if needed, protected with `acceso.permiso`.
- [ ] `./vendor/bin/pint --dirty` reports nothing pending.
