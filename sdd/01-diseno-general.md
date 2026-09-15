# 1. Diseño general y arquitectura

## Propósito

VSM ERP Web expone una interfaz React para autenticación, navegación, catálogos, procesos IARK, dashboards e integraciones externas. El backend Flask funciona como API, aplica validaciones y adapta MySQL, parametrización IARK y APIs externas de VisualTech.

## Vista de componentes

```text
Navegador
  └─ React + TypeScript + Vite
      ├─ autenticación y sesión
      ├─ navegación y rutas
      ├─ catálogos parametrizados
      ├─ procesos dinámicos IARK
      └─ dashboards/widgets
              │ HTTP / JSON / Bearer
              ▼
          Flask API
      ├─ casos de uso
      ├─ validación de esquema
      ├─ motor declarativo IARK
      ├─ repositorios MySQL
      └─ cliente de APIs externas
              │
              ├─ MySQL / parametrización
              └─ VisualTech u otros proveedores
```

## Principios confirmados

- Backend organizado como arquitectura hexagonal: `domain`, `application`, `infrastructure`.
- Frontend separado por capas `domain`, `application`, `infrastructure` y `presentation`.
- La composición de dependencias del backend se realiza en `app/infrastructure/http/flask_app.py`.
- La configuración de procesos, campos, eventos, grillas y acciones se almacena en JSON dentro del modelo IARK.
- El esquema de tablas se mantiene en `storage/schema.json` y se refresca mediante `/api/schema/refresh`.

## Norma obligatoria de parametrización

El ERP debe operar bajo una regla 100% obligatoria y no negociable: las reglas de negocio no se queman en el código de una pantalla ni en endpoints específicos. La base de datos y el modelo IARK son la fuente de parametrización para:

- opciones y estados de controles;
- validaciones y valores por defecto;
- relaciones y bindings;
- acciones y sus parámetros;
- eventos y refrescos dependientes;
- consultas, filtros y columnas visibles;
- cálculos declarativos y fórmulas;
- visibilidad, permisos y navegación.

Frontend y backend solo deben aportar el motor genérico que interpreta esa configuración, controles técnicos comunes, seguridad, autorización, límites de ejecución y contratos HTTP. Una excepción debe ser exclusivamente técnica, no contener lógica de negocio, y quedar registrada con justificación, alcance y pruebas.

Una implementación no cumple el SDD si agrega un `if`, `switch`, catálogo de opciones, nombre de acción, binding, filtro, cálculo o flujo específico de un proceso directamente en React, Flask o una migración de datos de negocio cuando ese comportamiento puede declararse en BD.

## Flujo principal de navegación

1. El usuario inicia sesión.
2. El frontend obtiene sesión, empresa y menú.
3. La ruta activa se resuelve como dashboard, catálogo o proceso dinámico.
4. El frontend solicita la definición y/o datos al backend.
5. La respuesta se normaliza y se presenta según la configuración.

## Decisiones y límites

- No se observó un gateway independiente ni un bus de eventos.
- No se debe asumir alta disponibilidad, colas, cache distribuido o ejecución asíncrona: quedan `Pendiente` de infraestructura.
- El sistema soporta integración externa parametrizada, pero el contrato concreto depende de la configuración activa en base de datos.
