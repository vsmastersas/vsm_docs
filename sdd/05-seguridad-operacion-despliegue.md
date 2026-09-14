# 5. Seguridad, operación y despliegue

## Seguridad observada

- El frontend envía `Authorization: Bearer` en las llamadas protegidas.
- El backend autentica al usuario y resuelve empresa activa.
- Las operaciones de escritura validan esquema, tabla, columnas y valores.
- Query rules bloquean múltiples sentencias y exigen `SELECT`.
- `vmauditoria` tiene protección específica frente a operaciones genéricas.
- Las credenciales y variables sensibles viven en `.env`, no en la parametrización versionada.

## Riesgos a controlar

- No exponer edición de SQL a usuarios sin autorización.
- Validar permisos por empresa y ruta en cada endpoint.
- Revisar CORS y secretos antes de desplegar.
- Limitar tamaño de páginas, importaciones y respuestas externas.
- No asumir que una respuesta externa siempre es lista; validar objetos, errores y documentos.

## Desarrollo

Backend: `python scripts/debug_flask.py` en `vsm_web_backend`.

Frontend: `npm run dev` en `vsm_web_frontend`, con proxy `/api` hacia Flask.

## Contenedores

Ambos proyectos contienen `Dockerfile` y `docker-compose.yml`. El frontend se sirve detrás de nginx según las instrucciones de su README. La topología productiva, réplicas, monitoreo y backups no están completamente descritos en el código y quedan `Pendiente`.

## Verificación

- Frontend: `npm run build` y `npx tsc --noEmit`.
- Backend: pruebas bajo `vsm_web_backend/tests/` y compilación sintáctica de Python.
- Integración: validar login, refresh de esquema, catálogo, proceso, relaciones, acciones y respuestas externas.
