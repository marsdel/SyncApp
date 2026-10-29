---
name: crear-rutas
description: Create routes in RPSync — frontend vue-router routes (defineModuleRoutes, permissions, breadcrumbs, menu) and/or backend API routes (routes/api/<modulo>.php, prefixes, middleware). Use when asked to "crear una ruta", "agregar ruta", "registrar la pantalla en el router", "nuevo endpoint", or when a screen or controller needs to be reachable.
---

# Create routes in RPSync

Two different route systems. Decide which one the task needs — a new screen usually
needs both (a frontend route to reach it, a backend route for its data).

- **Frontend** (`vue-router`) → section A
- **Backend** (Laravel API) → section B

---

# A. Frontend routes

## A1. Where things live

| File | Purpose |
| --- | --- |
| `resources/js/src/router/routes/<modulo>/index.ts` | The module's route array |
| `resources/js/src/router/routes/<modulo>/nombres/` | `ROUTE_*` name constants, split per section (`mantenimiento.ts`, `configuracion.ts`) + an `index.ts` barrel re-exporting them |
| `resources/js/src/router/index.ts` | Imports the module and spreads it into `routes: [...]` |
| `resources/js/src/navigation/vertical/<modulo>/` | Sidebar menu entries |

Route names follow `'<modulo>:<pantalla>'`, e.g.
`export const ROUTE_UNIDAD_NEGOCIO = 'administracion-pg:unidad-negocio'`.

`path` must be `kebab-case` and RFC3986-compliant:
`/administracion-pg/mantenimiento/unidad-negocio`.

## A2. `defineModuleRoutes` — use this for new or migrated modules

Available from `@/@core/utils/router`. It nests module → section → screen and builds
`path`, i18n `title` and `breadcrumb` for you; no manual `buildBreadcrumb`, no
repeating `resource` per screen.

```ts
import { defineModuleRoutes } from '@/@core/utils/router'
import permisos, { RESOURCE_ADMINISTRACION_PG } from '@/auth/permissions/modules/administracionPg'
import { ROUTE_UNIDAD_NEGOCIO } from './nombres'

const { mantenimiento } = permisos

export default defineModuleRoutes({
    base: 'administracion-pg',
    title: 'administracion.titulo_v2',
    resource: RESOURCE_ADMINISTRACION_PG,
    permissions: permisos, // optional but strongly recommended — see A3
    mantenimiento: [
        {
            path: 'unidad-negocio',
            name: ROUTE_UNIDAD_NEGOCIO,
            component: () => import('@/views/pages/administracion-pg/mantenimiento/UnidadNegocio.vue'),
            title: 'administracion.mantenimiento.unidad_negocio.titulo',
            action: mantenimiento.unidadNegocio,
        },
    ],
})
```

- The five standard sections have shortcut keys: `dashboards`, `mantenimiento`,
  `configuracion`, `transacciones`, `informes`.
- A non-standard section goes in `sections: [{ path, title, routes }]` (nested after
  the standard ones). Screens hanging directly off the module go in `routes:`.
- Loose `sections`/`routes` have no associated permission category, so they lose the
  `action` type-checking and relative `{.hoja}` references below. That is the only
  reason to prefer the five standard sections when the module fits them.

## A3. `permissions` gives you two free checks

Pass the same tree you already import and you get:

1. **Type-level** — a screen's `action` under `mantenimiento: [...]` is narrowed to
   the leaves of the `mantenimiento` category. Pasting a `configuracion` permission
   there, or misspelling one, is a compile error, not a runtime surprise.
2. **Runtime, DEV only** — the same check against real values, covering what the type
   cannot see (dynamically built arrays, `as any`).

Without `permissions`, `action` is an unconstrained `string`.

## A4. Combining permissions — `$or` / `$and`, not a hand-written `accessIf`

```ts
action: {
    $or: [
        transacciones.solicitudesCompra.creacionSolicitud,
        {
            $and: [
                transacciones.requisiciones.formularioRequisicion,
                { $or: ['{.formularioRequisicion}::crear', '{.formularioRequisicion}::ver'] },
            ],
        },
    ],
}
```

Each term is a `string` (checked against the module's `resource`), a
`[resource, action]` pair (a permission from **another** module, e.g.
`[RESOURCE_RRHH, 'ver-estructura']`), or a nested `$or`/`$and`. This compiles to
`meta.accessIf` — identical result to writing the function by hand, but with no
per-screen function to maintain or get wrong.

## A5. Reference syntax `{...}`

`action` also accepts `'{ref}sufijo'`, resolved against `permissions`:

- `'{.unidadNegocio}'` — **relative**, resolves against the category of the section
  the screen is declared in. Works into nested groups too: `'{.grupo.hoja}'`.
- `'{mantenimiento.unidadNegocio}'` — **absolute**, any category of the same tree.
- Anything after `}` is concatenated verbatim: `'{.formularioRequisicion}::crear'`.
- Requires `permissions` to have been passed (otherwise it throws at runtime).
  Relative refs only work in the five standard sections.

> `{...}` does **not** autocomplete in the editor — a verified TypeScript limitation
> (tsserver gives up on sufficiently recursive types), not a pending bug. It still
> rejects invented or misspelled references at compile time. Use the raw form
> (`mantenimiento.unidadNegocio`) if you want autocomplete. Both coexist on purpose.

## A6. Manual/legacy pattern (flat arrays)

Most existing modules still look like this. Valid for un-migrated modules; **do not
replicate in new code** — if you're touching one of those modules and need to match its
existing shape, read `references/legacy-route-pattern.md`.

## A7. The nested-route authorization hole

`canNavigate` (`resources/js/src/libs/acl/routeProtection.js`) grants access if **any**
matched record grants it:

```js
to.matched.some(route =>
    route.meta.accessIf?.(ability) ?? ability.can(route.meta.action || 'ver-publico', route.meta.resource))
```

Because it is a `.some()`, a **structural parent** record with no explicit `meta` falls
back to `action: 'ver-publico'` and opens access to all its children.
`defineModuleRoutes` handles this internally (every structural record gets
`meta.accessIf: () => false`). If you declare nested routes by hand, you must replicate
it yourself.

## A8. Register the module and the menu

1. In `router/index.ts`: add the `import miModulo from './routes/<modulo>'` and spread
   `...miModulo` into the `routes` array.
2. In `navigation/vertical/<modulo>/`, add the entry with `defineMenu([...])`:

   ```ts
   export default defineMenu([
       {
           title: 'administracion.mantenimiento.unidad_negocio.titulo',
           resource: RESOURCE_ADMINISTRACION_PG,
           action: mantenimiento.unidadNegocio,
           route: ROUTE_UNIDAD_NEGOCIO,
       },
   ])
   ```

   The group node uses the category `_perm` (`permisos.mantenimiento._perm`) as its
   `action` — that is why `_perm` must never be hand-typed wrong (see `crear-permiso`).
3. Add the i18n keys for `title` and the menu label in
   `libs/i18n/locales/<modulo>/{es,en}.json`.

## A9. Dev-only routes

`router/routes/dev.ts` shows the pattern: `accessIf: () => true` instead of a
`resource`/`action` pair, and `export default import.meta.env.DEV ? DEV_ROUTING : []`
so they never reach the production bundle.

---

# B. Backend API routes

Read `references/backend-routes.md` for the full detail: where each route file lives
(`routes/api/<modulo>.php` vs `routes/api/v2/resource/<modulo>.php`, and why
`routes/api.php` itself is legacy-only), how to write and group the routes, numeric
param constraints, and the auth/permission middleware (`auth:sanctum`,
`acceso.permiso:<action>,<resource>`, the service-to-service guard).

## B4. Checklist

- [ ] Route added to `routes/api/<modulo>.php` (or `routes/api/v2/resource/<modulo>.php`),
      **not** to `routes/api.php`.
- [ ] New file carries the `Prefix: ...` header comment and is mounted from the right
      parent file.
- [ ] Grouped by resource prefix in `kebab-case`; numeric params constrained.
- [ ] `acceso.permiso` applied where authorization is required, with strings identical
      to the frontend tree.
- [ ] Frontend: `path` in `kebab-case`, `ROUTE_*` const exported from `nombres/`,
      module spread into `router/index.ts`, menu entry added, i18n keys present.
- [ ] Frontend: hand-written nested routes carry `meta.accessIf: () => false` on
      structural parents.
- [ ] `./vendor/bin/pint --dirty` (backend) and oxlint/ESLint/oxfmt scoped to the files
      you touched (frontend) are clean.
