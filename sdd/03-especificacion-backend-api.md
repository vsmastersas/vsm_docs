# 3. Especificación del backend y API

## Tecnologías y capas

- Python 3.11+, Flask y MySQL Connector.
- Entrada HTTP: `app/infrastructure/http/flask_app.py`.
- Casos de uso: `app/application/use_cases/`.
- Servicios de dominio: `app/domain/services/`.
- Adaptadores MySQL: `app/infrastructure/database/`.
- Cliente externo: `app/infrastructure/external_api_client.py`.

## Endpoints confirmados

| Método | Endpoint | Función |
|---|---|---|
| POST | `/api/auth/login` | Autenticación |
| POST | `/api/schema/refresh` | Actualiza catálogo de esquema y parametrización |
| POST | `/api/catalogos/consulta` | Consulta catalogada con filtros y paginación |
| POST | `/api/catalogos/lookup` | Consulta puntual de registro |
| POST | `/api/catalogos/opciones-relacion` | Opciones para relaciones |
| POST | `/api/catalogos/importar` | Importación CSV |
| POST | `/api/integraciones/foxpro` | Operaciones dinámicas de insertar, actualizar, eliminar y query |
| POST | `/api/modelo/definicion` | Definición de proceso IARK |
| POST | `/api/modelo/grilla` | Datos de grilla parametrizada |
| POST | `/api/modelo/fuente-datos` | Fuente de datos de proceso |

La existencia y contrato exacto de rutas adicionales debe verificarse contra la versión desplegada.

## Operaciones FoxPro

- Operación `1`: insertar.
- Operación `2`: actualizar.
- Operación `3`: eliminar lógico.
- Operación `4`: ejecutar regla de consulta.

Las operaciones validan tabla y columnas contra el catálogo de esquema. Las modificaciones registran auditoría según el caso de uso y restricciones configuradas.

## Query rules

`ExecuteQueryRule` admite reglas activas con tipos `F`, `Q`, `A`, `B` y `C`. Las consultas SQL deben ser una única sentencia `SELECT`, sin `;`, y usan parámetros nombrados convertidos a parámetros del driver MySQL. No se observó ejecución de SQL libre enviado directamente por el navegador.

## Respuestas y errores

Las respuestas usan JSON y el indicador `exito`. Los errores de validación se devuelven normalmente con HTTP 400; errores de persistencia con HTTP 500 en endpoints de consulta de catálogos. El frontend convierte respuestas no exitosas en mensajes visibles.

## Integraciones externas

El cliente externo resuelve configuración por empresa/ruta/fuente, interpola payloads y mapea respuestas. También conserva respuestas individuales de documentos XML/JSON cuando el campo configurado aparece directamente o dentro de `data`.
