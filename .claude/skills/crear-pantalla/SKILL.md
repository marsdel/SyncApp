---
name: crear-pantalla
description: Create a new routed screen or component in the RPSync frontend (Vue 3 + PrimeVue + Tailwind). Use when asked to "crear una pantalla", "nueva pantalla", "crear componente", "add a screen/view" under resources/js/src/views/pages. Covers the artisan generator, file placement, i18n, Pinia store wiring and the PrimeVue/Tailwind rules.
---

# Create a screen in RPSync

Source of truth: `docs/Coding Styles.md` (Frontend section). This skill is the
operational version of it — read the doc when a case is not covered here.

## 0. Decide what you are building

| Case | Goes to |
| --- | --- |
| Routed screen (the router mounts it) | `resources/js/src/views/pages/<modulo>/<seccion>/<Pantalla>.vue` — **always**, no exceptions |
| Child component of one screen | `views/pages/<modulo>/<seccion>/<pantalla>/components/` (or `.../components/`, `.../modals/` as the module already does) |
| Cross-cutting UI infra component | `@core/components/` — restricted-use ones **must** be named `{Name}.ngl.vue` |
| Non-trivial business logic pulled out of a screen | `resources/js/src/features/<nombre-feature>/` (`composables/`, `lib/`, `model/`) |
| Domain state shared between screens | `store/<modulo>/domain/` + `store/<modulo>/repository/` (Pinia) |

Do **not** create `module/<dominio>/` — only `module/shared/` exists and the pattern
is not to be repeated.

## 1. Generate the file

There is an interactive generator:

```bash
php artisan make:screen
```

It asks for: module (from the existing dirs in `views/pages`), screen type
(`componente` | `transaccion` | `mantenimiento` | `configuracion` | `informe`),
component name, and whether it is "separado" (its own folder with `Index.vue`).
Type → directory mapping is `componente→component`, `transaccion→transacciones`,
`informe→informes`; `mantenimiento` and `configuracion` map to themselves.

Its template is minimal (`<script setup lang="ts">` + `useI18n` + `defineOptions`).
Writing the file by hand is equally fine — the generator only saves the mkdir.

## 2. Naming rules (enforced, `vue/component-definition-name-casing` is `error`)

- Component files: `PascalCase.vue` (`UnidadNegocio.vue`, never `unidad-negocio.vue`).
- Non-component `.ts`/`.js`: `camelCase.ts`.
- Route `path`: `kebab-case`, RFC3986 (`/administracion-pg/mantenimiento/unidad-negocio`).
- Module folder slug must be identical across `router/routes/<modulo>/`,
  `navigation/vertical/<modulo>/`, `store/<modulo>/`, `views/pages/<modulo>/` and
  `auth/permissions/modules/<modulo>.ts`. For **new** modules use `kebab-case`.

## 3. Skeleton

```vue
<template>
    <div class="t:flex t:flex-col t:gap-4">
        <PVCard>
            <template #content>
                <h4 class="t:m-0 t:font-semibold">{{ t(`${T_PREFIX}.titulo`) }}</h4>
            </template>
        </PVCard>
    </div>
</template>

<script setup lang="ts">
import type { MiEntidad } from '@/store/<modulo>/domain/miEntidad'

import { computed, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { useToastService } from '@vue-toastification'
import { useMiEntidadStore } from '@/store/<modulo>/repository/miEntidad'

defineOptions({ name: 'MiPantalla' })

const T_PREFIX = '<modulo>.<seccion>.<pantalla>'

const { t } = useI18n()
const toast = useToastService()
const { fetchMiEntidades } = useMiEntidadStore()
</script>
```

Real reference to copy from: `resources/js/src/views/pages/sistemas/ParametrosSalem.vue`
(PrimeVue + Tailwind + Pinia + i18n, no bootstrap-vue).

## 4. UI stack — PrimeVue 4 + Tailwind v4

- PrimeVue components auto-import with the `PV` prefix: `<PVCard>`, `<PVButton>`,
  `<PVDataTable>`. **Never import them manually.**
- Tailwind utilities carry the `t` prefix: `t:flex`, `t:items-center`, `t:gap-4`.
- Brand color: `t:text-primary` / `t:bg-primary` / `t:border-primary`, **no numeric
  step** (`t:text-primary-500` is unreadable in dark mode).
- **Respect the user preferences.** The UI has four (typeface, palette, density, size). Use
  the spacing scale and the text utilities, never arbitrary px (`t:p-[12px]`, `t:text-[11px]`),
  never a `font-family` in a component, never a literal hex for state colour, and never a fixed
  px height around text. Full rule in `.ai/rules/js.md`.
- **Banned in new code:** `bootstrap-vue` (and never reinstall the removed `bootstrap-vue-next`), `element-plus`,
  `vuex`, `moment`, `vue-cleave-component`, `portal-vue`, base `vee-validate`.

The full ruleset (dark-mode contrast math, the Vuexy SCSS override quirk, the
`isDesktop`-vs-Tailwind-breakpoint mismatch, and the complete deprecated-library →
replacement table) is the same one `review-code` checks against — read
`../review-code/rules/frontend.md` (sections "Deprecated libraries" and "PrimeVue +
Tailwind") instead of duplicating it here; that file is the canonical version.

## 5. Auto-imports — do not write these imports

`unplugin-auto-import` already provides: everything from `vue` (`ref`, `computed`,
`watch`…), `useI18n` from `vue-i18n`, `useRoute`/`useRouter` from `vue-router`, and
`useAbility` from `@casl/vue`. `unplugin-vue-components` auto-registers every `.vue`
under `@core/components/` (except `*.ngl.vue`) and `layouts/`, plus PrimeVue (`PV`),
`@vueuse/core`, `FontAwesomeIcon`, `FeatherIcon`, `Cleave`, the `v-maska` directive
and others. Check `vite.config.ts` before adding an import "just in case".

`resources/js/auto-imports.d.ts` and `components.d.ts` are gitignored and regenerated
by `pnpm dev` / `pnpm build` — never edit them.

## 6. i18n

- Translate with `useI18n().t()`. There is **no** `useSalemI18n`.
- Keys live per module in `resources/js/src/libs/i18n/locales/<modulo>/{es,en}.json`.
  Fill **both** files.
- React to locale changes (syncing an external lib) with `useOnLocaleChange` from
  `@lib-i18n/composables`; change the active locale with `useLocaleController()`.
- If you create the locales folder for a **new** module, it must be added by hand to
  `i18n-ally.localesPaths` in `.vscode/settings.json` (gitignored, so each dev does it
  locally) or keys will not autocomplete for anyone.

## 7. State

New shared state uses **Pinia**, never Vuex:

- `store/<modulo>/domain/<entidad>.ts` — types only, no logic.
- `store/<modulo>/repository/<entidad>.ts` — one setup-store `use<Entidad>Store`
  wrapping the axios calls (`fetch*`, `crear*`, `actualizar*`, `desactivar*`), with
  `acceptHMRUpdate` registered.
- `store/<modulo>/index.ts` — barrel re-exporting `domain/*` and `repository/*`.

Reference: `store/compras/repository/servicio.ts`.

## 8. Wire the screen up

A screen is not done until it is reachable and protected. In order:

1. Permission in `auth/permissions/modules/<modulo>.ts` → skill `crear-permiso`.
2. Route + route-name const → skill `crear-rutas`.
3. Menu entry in `navigation/vertical/<modulo>/` (`defineMenu([...])` with `title`,
   `resource`, `action`, `route`).
4. i18n keys for title, breadcrumb and menu.

## 9. Before finishing

Run the verification skill `verificar-frontend`, or at minimum:

```bash
pnpm exec oxlint  -c oxlint.config.ts        <your files>
pnpm exec eslint  -c eslint.config.ts        <your files>
pnpm exec oxfmt   -c oxfmt.config.ts --check <your files>
```

Only `pnpm` is allowed in this repo. Scope the commands to your own files: the unscoped
`pnpm lint` / `pnpm fmt:check` cover all of `resources/js/src` and are red on legacy code,
and `pnpm fmt` would rewrite the whole tree.

Note that the pre-commit `lint-staged` glob does **not** cover pure `.ts` under
`features/`, `reports/`, `data/` or `types/` — if you touched those, running the commands
above by hand is mandatory.
