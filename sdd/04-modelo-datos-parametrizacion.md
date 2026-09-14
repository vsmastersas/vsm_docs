# 4. Modelo de datos y parametrización

## Familias de configuración

- Catálogos: `vmcatalogo`, `vmcatalogo_campo` y tablas relacionadas.
- Modelo IARK: procesos, campos, secciones, fuentes de datos, eventos, bindings, acciones y grillas.
- Query rules: `query_reglas`.
- Dashboards: widgets, filtros, orígenes y reglas de consulta.
- Integraciones externas: configuración de API, fuentes y acciones.
- Auditoría: `vmauditoria`.

## Parametrización declarativa

La configuración activa define qué se renderiza y cómo se consulta. El frontend no requiere una pantalla específica por cada proceso: interpreta la definición recibida, incluyendo controles, relaciones, acciones, resultados y grillas.

## Relaciones y bindings

Las relaciones definen tabla, campo de valor, campo de etiqueta, filtros por empresa/estado y creación opcional. Los bindings conectan valores de respuesta con campos destino y permiten cascadas entre consultas.

## Paginación

- Catálogos: soportan paginación tradicional y cursor, según configuración y repositorio.
- Grillas IARK: soportan página, tamaño, búsqueda y total devuelto por backend.
- Query rules generales: retornan resultados transformados, pero no exponen por sí mismas un contrato universal de paginación.

## Migraciones relevantes

La evolución se encuentra en `vsm_web_backend/sql/`, incluyendo scripts del modelo IARK, catálogos, dashboards, query rules, acciones, integraciones y Consulta FE. Los scripts deben ejecutarse en orden del proyecto y después realizar `POST /api/schema/refresh` cuando cambie esquema o parametrización.

## Pendiente de confirmar

- Estado exacto de las migraciones aplicadas en cada ambiente.
- Valores activos por empresa en la base desplegada.
- Convención formal de versionado de parametrizaciones.
