# 6. Trazabilidad y pendientes

## Matriz de trazabilidad

| Requisito | Evidencia | Estado |
|---|---|---|
| Login y sesión | `App.tsx`, `LoginUseCase.ts`, `HttpAuthRepository.ts` | Confirmado |
| Menú por permisos/ruta | `DashboardPlaceholder.tsx`, `AuthTypes.ts` | Confirmado |
| Catálogos parametrizados | `CatalogWorkspace`, `mysql_catalog_repository.py` | Confirmado |
| CRUD genérico | `insert_records.py`, `update_records.py`, `delete_records.py` | Confirmado |
| Query parametrizado | `execute_query_rule.py`, `mysql_query_rule_repository.py` | Confirmado, resultado no universalmente paginado |
| Formularios IARK | `DynamicProcessWorkspace.tsx`, `DynamicModelTypes.ts` | Confirmado |
| Grillas paginadas | `DynamicGridView`, repositorio de modelo dinámico | Confirmado |
| Exportación CSV | `DashboardPlaceholder.tsx` | Confirmado |
| Exportación PDF | flujo de exportación de catálogos | Confirmado como salida imprimible |
| Excel `.xlsx` nativo | No se encontró generador XLSX | No confirmado |
| Pantalla automática desde SQL custom | No existe contrato específico | Pendiente de desarrollo |
| Alta disponibilidad | No evidenciada en repositorios | Pendiente de infraestructura |

## Recomendaciones

1. Publicar un contrato OpenAPI generado desde las rutas Flask.
2. Agregar pruebas de contrato para cada endpoint y respuesta de error.
3. Documentar la versión de cada migración aplicada por ambiente.
4. Separar explícitamente exportación de página visible frente a exportación completa filtrada.
5. Definir un contrato formal para consultas custom con filtros, columnas, paginación y exportación.

## Nota de exactitud

Este paquete documenta el comportamiento del código fuente; no sustituye la validación de configuración activa, variables de entorno, permisos reales, datos de producción ni infraestructura desplegada.
