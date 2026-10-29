# B. Backend API routes

## B1. Where things live

- One file per module: `routes/api/<modulo>.php` (`contabilidad.php`, `logistica.php`,
  `activo_fijo.php`, `importacion.php`, …), with subfolders for bigger areas
  (`admin/`, `controlCalidad/`, `sri/`, `v1/`, `v2/`, `v3/`).
- The historical `routes/api.php` still exists with mixed modules. **New routes never
  go there** — but that file is where a module's file gets mounted, inside the
  `prefix('auth')` + `['auth:sanctum', 'db.nconn', 'query.log']` group:

  ```php
  Route::prefix('contabilidad')->group(base_path('routes/api/contabilidad.php'))->name('contabilidad:f2:root');
  ```

  So a module mounted there answers under `/api/auth/<prefijo>/...`.
- Versioned resources live under `routes/api/v2/resource/<modulo>.php`, mounted from
  `routes/api/v2/api.php`:

  ```php
  Route::prefix('compras')->as('compras:')
      ->middleware(['auth:sanctum', 'db.nconn', 'query.log'])
      ->group(base_path('routes/api/v2/resource/compras.php'));
  ```

  `bootstrap/app.php` mounts `routes/api/v2/api.php` under the `api/v2` prefix with the
  `salem-api.core` middleware group, so these answer under `/api/v2/compras/...`.
- `routes/providers-api/` is a **separate** external supplier portal API
  (`/p/api/...`, versioned `v0`/`v1`). Never mix internal ERP routes into it.

## B2. Writing the routes

```php
<?php

use App\Http\Controllers\<Modulo>\<Submodulo>\<Entidad>Controller;

/**
 * <Qué agrupa este archivo>
 *
 * Prefix: /api/auth/v2/contabilidad
 */
Route::group([
    'prefix' => '<entidad-en-kebab-case>',
], function (): void {
    Route::get('{id_entidad}/detalle', [<Entidad>Controller::class, 'detalle'])
        ->whereNumber('id_entidad');
});
```

- Group by resource with a prefix; do not write a flat route per action.
- Constrain numeric route parameters with `->whereNumber(...)` (or `->where(...)`) —
  consistent practice in the `Pg`/`V2` modules.
- Put a block comment at the top of a new route file stating the **real** API prefix
  it ends up under. It is the fastest way for the next person to find the right file.
- `Route::apiResource(...)` and `Route::controller(...)->prefix(...)->group(...)` are
  both used in the codebase and are fine.

## B3. Auth and permissions

- User auth: Laravel Sanctum (`auth:sanctum`) — already applied by the mount groups
  above, don't re-add it per route.
- Permission check: `->middleware('acceso.permiso:<action>,<resource>')`. The strings
  must match the frontend permission tree exactly — see `crear-permiso`.
- Service-to-service integrations use a separate guard (`auth('services')`) with
  `service.perm` and `token.claim` middleware. Never reuse the user Sanctum guard for
  those, or vice versa.

Other useful aliases registered in `bootstrap/app.php`: `db.nconn`, `add.nconn`,
`add.user`, `add.empresa`, `es.admin`, `idempotency`, `query.log`, `track.salem`.
