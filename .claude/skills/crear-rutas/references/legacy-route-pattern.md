# Manual/legacy route pattern (flat arrays)

Most existing modules still look like this (`router/routes/administracion-pg/index.ts`).
Valid for un-migrated modules; **do not replicate in new code** — use `defineModuleRoutes`
(see the main skill file) instead.

```ts
export default [
    {
        path: '/administracion-pg/mantenimiento/unidad-negocio',
        name: ROUTE_UNIDAD_NEGOCIO,
        component: () => import('@/views/pages/administracion-pg/mantenimiento/UnidadNegocio.vue'),
        meta: {
            title: 'i18n:administracion.mantenimiento.unidad_negocio.titulo',
            action: permisos.mantenimiento.unidadNegocio,
            resource: RESOURCE_ADMINISTRACION_PG,
            breadcrumb: buildBreadcrumb(
                'a:administracion.titulo_v2',
                'a:global.menu.mantenimiento',
                'administracion.mantenimiento.unidad_negocio.titulo'
            ),
        },
    },
]
```

Both forms produce the same final `meta` (`title`, `action`, `resource`, `breadcrumb`).
