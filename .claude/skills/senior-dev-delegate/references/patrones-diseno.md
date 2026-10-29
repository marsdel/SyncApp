# Patrones de diseño — Laravel (backend) y Vue 3 (frontend)

Anexa este archivo al prompt del subagente **solo cuando la tarea implica decidir dónde
va código nuevo** (una clase nueva, un tipo de comportamiento intercambiable, un composable
nuevo) — no para un fix o ajuste acotado a un archivo ya existente. Es reconocimiento de
patrón para tomar esa decisión, no una checklist a marcar: no fuerces un patrón donde no
aplica ni migres código estable existente solo para "encajarlo" en uno de estos.

## Patrones de diseño de Laravel (backend)

Reconoce qué patrón GoF subyace en Laravel para la pieza que estás tocando y sigue la
carpeta/convención que el proyecto ya usa para ese patrón — nunca la versión "de libro" si
`.ai/rules` dice otra cosa:

- **Dependency Injection / Facade**: resuelve colaboradores por constructor promovido
  (typed, normalmente detrás de interfaz), nunca `new Servicio()` a mano cuando ya existe
  o debería existir un binding. Los bindings de repositorios por módulo van en un
  `{Modulo}BindServiceProvider` bajo `app/Providers/Modules/Bindings/` (ver
  `.ai/rules/bindings.md`) — no registres bindings sueltos en `AppServiceProvider`.
- **Strategy**: comportamiento intercambiable dentro de un mismo flujo (registro contable
  de activos fijos, tipos de retención, cabeceras de guía de remisión) va en
  `app/Strategies/{Modulo}/`, siguiendo el patrón ya usado ahí — no metas un
  `switch`/`match` gigante en el servicio si el módulo ya tiene estrategias, y no crees
  una carpeta de estrategias nueva para un caso que aún no las necesita.
- **Command** (la idea de "acción de negocio como objeto de primera clase"): en este
  proyecto NO se modela como clases `Action` con `execute()/handle()` — no existe esa
  convención aquí. Un servicio single-purpose con un método verbo de dominio
  (`app/Services/**`) cumple el mismo rol; no introduzcas una carpeta `Actions/` nueva.
- **Observer**: `app/Observers/` existe pero está reservado para scopes/hooks globales de
  `booted()`, NO para reaccionar a eventos de negocio. Un side effect de negocio (sync con
  sistema legacy, notificación, fan-out) se dispara explícitamente como `App\Events\{Modulo}\...`
  desde el Service — nunca lo escondas en un Observer ni en `booted()`.
- **Iterator**: usa los métodos de `Collection` (`->map`, `->filter`, `->reduce`, `->each`)
  sobre resultados Eloquent/colecciones en vez de loops manuales con arrays.
- **Singleton / Factory** (contenedor de servicios): confía en que el container resuelve
  una sola instancia por request cuando corresponde (`singleton()` en un provider) — no
  implementes un singleton manual con propiedad estática.

## Patrones de Vue (frontend)

El catálogo clásico de patrones Vue (Component Communication, Renderless Component, HOC,
Provider/Consumer, etc.) nació en la era de Options API/Vue 2. Este proyecto exige
componentes nuevos en Vue 3 `<script setup>` + Composition API — usa la traducción de cada
patrón a ese idioma, no la sintaxis Vue 2 original:

- **Component Communication / Passing Props & Listeners**: `defineProps`/`defineEmits`
  tipados (props abajo, eventos arriba). Bajo `@vue/compat` MODE 2 evita `$parent`/`$root`
  para comunicarte con un ancestro — son parte de lo que la migración está retirando.
- **Composition / HOC / Functional / Renderless Component**: en Options API estos cuatro
  patrones existen para compartir lógica sin herencia. En Composition API el reemplazo
  directo de los cuatro es un **composable** (`use*.ts`) — ver
  `resources/js/src/@core/composables/` y los `composables/` locales junto a componentes
  complejos (`useSelectOption.ts`, `useFilterHolder.ts`). Si vas a extraer lógica
  reutilizable, escribe un composable primero; no envuelvas un componente "renderless" con
  scoped slots ni repliques `functional: true` de Vue 2 — ya no aplica en Vue 3.
- **Provider / Consumer / Dependency Injection**: `provide()`/`inject()` con una clave
  tipada (`InjectionKey<T>`), siguiendo el patrón ya usado en
  `resources/js/src/@core/auth/jwt/key.ts` + `plugin.ts` — no un objeto global mutable ni
  un store ad-hoc para algo que solo necesita un subárbol de componentes.
- **Dynamic Component**: `<component :is="...">` solo cuando el nombre del componente es
  realmente dinámico en runtime; si el proyecto ya auto-importa el componente vía
  `unplugin-vue-components`, no lo importes a mano ni lo registres localmente.
- **Handling Errors**: captura en el límite de la operación asíncrona (el composable o
  función que llama a la API), no con un `try/catch` en cada componente que lo consume —
  mismo principio de "validar solo en los límites del sistema" que en backend.
- **Productivity Tips**: aprovecha el auto-import de `unplugin-vue-components` y los macros
  de `<script setup>` (`defineProps`, `defineEmits`, `defineExpose`) en vez de la API de
  opciones (`props: {}`, `this.$emit`) salvo que estés editando un componente Vue 2/Options
  ya existente que todavía no se ha migrado — ahí sigue la convención del archivo, no la
  impongas a mitad de archivo.
