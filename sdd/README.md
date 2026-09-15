# SDD — VSM ERP Web

Paquete de Software Design Description levantado del código fuente de `vsm_web_frontend` y `vsm_web_backend`.

## Documentos

- [01. Diseño general y arquitectura](./01-diseno-general.md)
- [02. Especificación del frontend](./02-especificacion-frontend.md)
- [03. Especificación del backend y API](./03-especificacion-backend-api.md)
- [04. Modelo de datos y parametrización](./04-modelo-datos-parametrizacion.md)
- [05. Seguridad, operación y despliegue](./05-seguridad-operacion-despliegue.md)
- [06. Trazabilidad y pendientes](./06-trazabilidad-pendientes.md)

## Alcance y criterio

La especificación describe el comportamiento observado en el repositorio al **12 de septiembre de 2026**. Los puntos marcados como `Pendiente` requieren confirmación en base de datos, infraestructura o ambiente desplegado; no se presentan como capacidades garantizadas.

## Fuentes principales

- Frontend: `vsm_web_frontend/src/`
- Backend: `vsm_web_backend/app/`
- Migraciones y parametrización: `vsm_web_backend/sql/`
- Contratos y ejemplos: `vsm_web_backend/README.md`, `vsm_docs/`
