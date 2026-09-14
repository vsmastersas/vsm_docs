# 2. Especificación del frontend

## Tecnologías

- React, TypeScript y Vite.
- Entrada: `src/main.tsx`.
- Composición principal: `src/presentation/App.tsx`.
- Cliente HTTP común: `src/infrastructure/http/apiFetch.ts`.

## Módulos

| Módulo | Implementación | Responsabilidad |
|---|---|---|
| Autenticación | `LoginPage`, `LoginUseCase`, `HttpAuthRepository` | Inicio de sesión y sesión activa |
| Shell | `DashboardPlaceholder` | Topbar, sidebar, menú y selección de ruta |
| Catálogos | `CatalogWorkspace`, `HttpCatalogRepository` | Consulta, búsqueda, CRUD, importación y exportación |
| Procesos | `DynamicProcessWorkspace` | Renderizado declarativo de formularios, eventos y acciones |
| Grillas | `DynamicGridView` | Consulta paginada dentro de procesos |
| Dashboards | `DynamicHomeDashboard`, `DashboardWidgets` | Widgets y filtros de dashboard |
| VisualTech | `VisualtechInvoiceDetail`, acciones dinámicas | Resultado y detalle de factura |

## Catálogos

El frontend consume `/api/catalogos/consulta` con ruta, página, tamaño, búsqueda, campo, modo, orden, estado y cursor. Presenta columnas parametrizadas, filtros, paginación, permisos y acciones de crear/editar/eliminar/importar/exportar.

La exportación actual se realiza en el navegador sobre los registros visibles. El formato denominado `Excel / CSV` genera CSV; no se confirma generación de `.xlsx` nativo.

## Procesos IARK

`DynamicProcessWorkspace` interpreta controles como texto, número, fecha, select, checkbox, JSON, rich text, archivo, mapa, acciones y `data_grid`. También soporta eventos `init`, `change`, `blur` y `submit`, bindings, relaciones, modales y acciones externas.

Las grillas dinámicas consultan `/api/modelo/grilla` con `page`, `size` y `search`; mantienen paginación local de la respuesta del servidor.

## Presentación

Los estilos globales están en `src/presentation/styles/global.css`. El layout global incluye sidebar, topbar, encabezados homologados, tarjetas, tablas, estados vacíos y bloques de código XML/JSON con resaltado.

## Validación observada

```sh
npm run build
npx tsc --noEmit
```
