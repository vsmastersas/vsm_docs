---
title: Integración FoxPro para migración de datos
description: Contrato JSON para insertar, actualizar y eliminar datos de forma individual o masiva
published: true
date: 2026-07-10T00:00:00.000Z
tags: foxpro, migracion, api, json
editor: markdown
dateCreated: 2026-07-09T18:54:45.994Z
---

# Integración FoxPro para migración de datos

## 1. Objetivo

Definir un contrato JSON para migrar datos ordenadamente desde un sistema legacy FoxPro hacia la base de datos interna. El contrato permite:

- insertar uno o varios registros;
- actualizar uno o varios registros, cada uno con criterios propios;
- eliminar uno o varios registros, cada uno con criterios propios;
- reutilizar el modelo de reglas y parámetros nombrados existente en `nominasimple_bk`;
- conocer el resultado de cada elemento enviado, incluso cuando un lote contiene errores parciales;
- ejecutar validaciones o simulaciones antes de modificar datos.

Cada solicitud opera sobre una sola tabla y usa una sola operación. No se permite enviar SQL libre desde FoxPro.

---

## 2. Operaciones

| Valor | Operación | Contenido requerido |
|---|---|---|
| `"1"` | Insertar | `registros[].campos` |
| `"2"` | Actualizar | `registros[].campos` y `registros[].criterio` |
| `"3"` | Eliminar | `registros[].criterio` |

Aunque por compatibilidad se aceptará un objeto único, el formato canónico siempre usa `registros` como arreglo. Esto simplifica el procesamiento y la respuesta de lotes.

---

## 3. Esquema general de la solicitud

```json
{
  "id_solicitud": "MIG-20260710-0001",
  "operacion": "1",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "atomico": true,
  "registros": [
    {
      "id_origen": "FOX-CLI-0001",
      "campos": {
        "nombre": "Juan",
        "apellido": "Pérez"
      }
    }
  ]
}
```

### Atributos de nivel superior

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---:|---|
| `id_solicitud` | string | Sí | Identificador único e idempotente generado por el sistema legacy. |
| `operacion` | string | Sí | `"1"`, `"2"` o `"3"`. |
| `tabla` | string | Sí | Alias de una tabla registrada en la lista blanca. |
| `nit_empresa` | string | Sí | Contexto obligatorio de empresa. |
| `modo` | string | No | `validar` o `ejecutar`. Por defecto, `ejecutar`. |
| `atomico` | boolean | No | Si es `true`, cualquier error revierte todo el lote. Recomendado para migraciones. |
| `registros` | array | Sí | Uno o varios elementos de la misma operación. |

### Atributos de cada registro

| Campo | Insertar | Actualizar | Eliminar | Descripción |
|---|---:|---:|---:|---|
| `id_origen` | Obligatorio | Obligatorio | Obligatorio | Identificador del registro en FoxPro para trazabilidad. |
| `campos` | Obligatorio | Obligatorio | No permitido | Valores que se insertarán o modificarán. |
| `criterio` | No requerido | Obligatorio | Obligatorio | Condiciones estructuradas para localizar registros. |
| `esperados` | No | Recomendado | Recomendado | Cantidad esperada de filas afectadas; evita cambios accidentales. |

`nit_empresa` no debe repetirse dentro de `campos` ni de `criterio`: el servicio lo agrega y valida automáticamente según la configuración de la tabla.

---

## 4. Esquema general de la respuesta

Todas las operaciones devuelven el mismo formato:

```json
{
  "id_solicitud": "MIG-20260710-0001",
  "operacion": "1",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "estado": "exitoso",
  "atomico": true,
  "confirmado": true,
  "resumen": {
    "recibidos": 1,
    "exitosos": 1,
    "fallidos": 0,
    "filas_afectadas": 1
  },
  "resultados": [
    {
      "indice": 0,
      "id_origen": "FOX-CLI-0001",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": {
        "id": 901
      },
      "error": null
    }
  ],
  "errores": []
}
```

### Estados

- `exitoso`: todos los registros fueron procesados.
- `parcial`: hubo éxitos y errores; solo es posible cuando `atomico` es `false`.
- `fallido`: ningún cambio fue confirmado o la transacción fue revertida.
- `validado`: la solicitud es válida, pero no se modificaron datos porque `modo` es `validar`.

`confirmado` indica si la transacción quedó aplicada. Cuando `atomico` sea `true` y falle un elemento, `confirmado` será `false`, incluso si algún elemento alcanzó a ejecutarse antes del rollback.

### Esquema de error

```json
{
  "codigo": "CRITERIO_CANTIDAD_INESPERADA",
  "mensaje": "El criterio encontró 2 filas y se esperaba 1",
  "campo": "registros[0].criterio",
  "detalle": {
    "esperados": 1,
    "encontrados": 2
  }
}
```

No se debe devolver SQL, credenciales, stack traces ni datos sensibles. El modelo actual de `stackErrors()` incluye `query` y `params`; para este endpoint deben registrarse únicamente en logs internos y sanearse en la respuesta pública.

---

## 5. Operación 1: insertar uno o varios registros

### Solicitud

```json
{
  "id_solicitud": "MIG-20260710-INS-001",
  "operacion": "1",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "atomico": true,
  "registros": [
    {
      "id_origen": "FOX-CLI-001",
      "campos": {
        "nombre": "Ana",
        "apellido": "Gómez",
        "telefono": "3001234567"
      }
    },
    {
      "id_origen": "FOX-CLI-002",
      "campos": {
        "nombre": "Luis",
        "apellido": "Pérez",
        "telefono": null
      }
    }
  ]
}
```

### Operación parametrizada equivalente

```sql
INSERT INTO cliente (nombre, apellido, telefono, nit_empresa)
VALUES (:nombre, :apellido, :telefono, :nit_empresa);
```

Se ejecuta una vez por elemento dentro de una transacción. Nunca se concatenan valores en el SQL.

### Respuesta esperada para la solicitud anterior

```json
{
  "id_solicitud": "MIG-20260710-INS-001",
  "operacion": "1",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "estado": "exitoso",
  "atomico": true,
  "confirmado": true,
  "resumen": {
    "recibidos": 2,
    "exitosos": 2,
    "fallidos": 0,
    "filas_afectadas": 2
  },
  "resultados": [
    {
      "indice": 0,
      "id_origen": "FOX-CLI-001",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": { "id": 901 },
      "error": null
    },
    {
      "indice": 1,
      "id_origen": "FOX-CLI-002",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": { "id": 902 },
      "error": null
    }
  ],
  "errores": []
}
```

### Reglas de inserción

- La tabla y los campos deben estar registrados en una lista blanca.
- El servicio agrega `nit_empresa` cuando la tabla está aislada por empresa.
- `id_origen` no necesariamente se inserta; se usa para trazabilidad y conciliación.
- Los campos omitidos conservan el valor predeterminado de la base de datos; `null` significa insertar `NULL`.
- La respuesta devuelve la llave generada cuando la tabla la tenga.
- `id_solicitud` no puede ejecutarse dos veces con un contenido diferente.

---

## 6. Operación 2: actualizar con criterios configurables

Cada elemento contiene los campos que cambiarán y su propio criterio. Esto permite actualizar varios registros distintos en una solicitud.

### Solicitud con criterios simples y compuestos

```json
{
  "id_solicitud": "MIG-20260710-UPD-001",
  "operacion": "2",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "atomico": true,
  "registros": [
    {
      "id_origen": "FOX-CLI-001",
      "campos": {
        "telefono": "3101112233"
      },
      "criterio": {
        "id": 901
      },
      "esperados": 1
    },
    {
      "id_origen": "FOX-CLI-002",
      "campos": {
        "estado": "A"
      },
      "criterio": {
        "codigo": "CLI002",
        "tipo_documento": "CC"
      },
      "esperados": 1
    }
  ]
}
```

Las propiedades del objeto abreviado `criterio` se combinan con `AND` y equivalen al operador `eq`.

### Operación parametrizada equivalente del primer elemento

```sql
UPDATE cliente
SET telefono = :set_telefono
WHERE id = :where_id
  AND nit_empresa = :nit_empresa;
```

### Respuesta esperada

```json
{
  "id_solicitud": "MIG-20260710-UPD-001",
  "operacion": "2",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "estado": "exitoso",
  "atomico": true,
  "confirmado": true,
  "resumen": {
    "recibidos": 2,
    "exitosos": 2,
    "fallidos": 0,
    "filas_afectadas": 2
  },
  "resultados": [
    {
      "indice": 0,
      "id_origen": "FOX-CLI-001",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": null,
      "error": null
    },
    {
      "indice": 1,
      "id_origen": "FOX-CLI-002",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": null,
      "error": null
    }
  ],
  "errores": []
}
```

### Criterio avanzado

Cuando se requieran rangos, listas o agrupaciones, `criterio` usa la forma explícita:

```json
{
  "and": [
    { "campo": "estado", "operador": "eq", "valor": "A" },
    {
      "or": [
        { "campo": "codigo", "operador": "in", "valor": ["CLI001", "CLI002"] },
        { "campo": "fecha_modificacion", "operador": "between", "valor": ["2026-01-01", "2026-06-30"] }
      ]
    }
  ]
}
```

Operadores permitidos:

| Operador | Significado | Forma de `valor` |
|---|---|---|
| `eq`, `ne` | Igual, diferente | escalar o `null` |
| `gt`, `gte`, `lt`, `lte` | Comparación | escalar |
| `in`, `not_in` | Pertenece/no pertenece | arreglo no vacío |
| `between` | Rango inclusivo | arreglo de dos valores |
| `like` | Patrón permitido | string |
| `is_null`, `is_not_null` | Verificación de nulo | no usa `valor` |

Los nombres de campo y operador se validan contra configuración; todos los valores se envían como parámetros enlazados.

### Reglas de actualización

- `criterio` nunca puede estar vacío.
- El servicio agrega `nit_empresa = :nit_empresa` fuera de cualquier grupo `OR`.
- Un campo no puede aparecer a la vez en `campos` y como campo protegido de la tabla.
- Antes de actualizar se cuenta o bloquea el conjunto objetivo y se compara con `esperados`.
- Para migración ordinaria se recomienda `esperados: 1`; una actualización masiva deliberada debe declarar la cantidad exacta.
- Una actualización que no cambia valores puede devolver `filas_afectadas: 0` y `filas_encontradas: 1`; ambas métricas deben diferenciarse si el motor lo permite.

---

## 7. Operación 3: eliminar con criterios configurables

### Solicitud para eliminar varios registros por criterios independientes

```json
{
  "id_solicitud": "MIG-20260710-DEL-001",
  "operacion": "3",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "atomico": true,
  "registros": [
    {
      "id_origen": "FOX-CLI-001",
      "criterio": {
        "id": 901
      },
      "esperados": 1
    },
    {
      "id_origen": "FOX-CLI-002",
      "criterio": {
        "codigo": "CLI002",
        "tipo_documento": "CC"
      },
      "esperados": 1
    }
  ]
}
```

### Operación parametrizada equivalente del primer elemento

```sql
DELETE FROM cliente
WHERE id = :where_id
  AND nit_empresa = :nit_empresa;
```

### Respuesta esperada

```json
{
  "id_solicitud": "MIG-20260710-DEL-001",
  "operacion": "3",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "estado": "exitoso",
  "atomico": true,
  "confirmado": true,
  "resumen": {
    "recibidos": 2,
    "exitosos": 2,
    "fallidos": 0,
    "filas_afectadas": 2
  },
  "resultados": [
    {
      "indice": 0,
      "id_origen": "FOX-CLI-001",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": null,
      "error": null
    },
    {
      "indice": 1,
      "id_origen": "FOX-CLI-002",
      "estado": "exitoso",
      "filas_afectadas": 1,
      "llave_generada": null,
      "error": null
    }
  ],
  "errores": []
}
```

### Reglas de eliminación

- No se permite `campos`.
- `criterio` y `esperados` son obligatorios.
- El servicio agrega siempre el criterio de empresa.
- Se debe validar la cantidad antes de eliminar.
- Para tablas configuradas con borrado lógico, la operación cambia el campo de estado en vez de ejecutar `DELETE`.
- Las dependencias y el orden de eliminación deben definirse por tabla para no violar llaves foráneas.

---

## 8. Validación sin modificar datos

Antes de ejecutar un lote se recomienda enviarlo con `"modo": "validar"`. El sistema valida esquema, tabla, campos, tipos, criterios, reglas y cantidad esperada, pero hace rollback o no inicia escrituras.

```json
{
  "id_solicitud": "MIG-20260710-VAL-001",
  "operacion": "2",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "validar",
  "atomico": true,
  "registros": [
    {
      "id_origen": "FOX-CLI-001",
      "campos": { "telefono": "3101112233" },
      "criterio": { "id": 901 },
      "esperados": 1
    }
  ]
}
```

Respuesta:

```json
{
  "id_solicitud": "MIG-20260710-VAL-001",
  "operacion": "2",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "validar",
  "estado": "validado",
  "atomico": true,
  "confirmado": false,
  "resumen": {
    "recibidos": 1,
    "exitosos": 1,
    "fallidos": 0,
    "filas_afectadas": 0
  },
  "resultados": [
    {
      "indice": 0,
      "id_origen": "FOX-CLI-001",
      "estado": "validado",
      "filas_encontradas": 1,
      "filas_afectadas": 0,
      "llave_generada": null,
      "error": null
    }
  ],
  "errores": []
}
```

---

## 9. Reutilización del modelo de reglas de `nominasimple_bk`

### 9.1. Comportamiento existente

El repositorio contiene un motor de reglas basado en la tabla `nreglas`:

1. `getReglas()` obtiene reglas habilitadas y ordenadas.
2. Cada regla puede contener una consulta en `query_dato`.
3. Las consultas usan parámetros nombrados, por ejemplo `:nit_empresa`, `:docum` y `:codigo`.
4. `solveParams($params, $qry)` conserva únicamente los parámetros que aparecen en la consulta particular.
5. `executeQuery()` obtiene la primera fila y `executeQueryArray()` obtiene todas las filas.
6. `armarElementosHijoPadre()` construye una salida jerárquica usando `id_tag_padre` y `nom_tag`.

El campo `tipo_dato` admite varias formas:

| Tipo | Comportamiento actual | Resultado lógico |
|---|---|---|
| `F` | Usa `query_dato` sin ejecutar consulta | valor fijo |
| `Q` | Ejecuta consulta y toma el primer campo de la primera fila | escalar |
| `A` | Ejecuta consulta y conserva todas las filas | arreglo de objetos |
| `B` | Toma todas las columnas de la primera fila | arreglo de valores |
| `C` | Toma la primera columna de todas las filas | arreglo de valores |

### 9.2. Modelo propuesto para escrituras

Se propone reutilizar la idea de reglas configuradas y parámetros nombrados, no ejecutar directamente `query_dato` recibido desde FoxPro. Cada tabla se registra internamente en un catálogo `reglas_query` o configuración equivalente:

```json
{
  "alias": "cliente",
  "tabla_fisica": "cliente",
  "campo_empresa": "nit_empresa",
  "campo_llave": "id",
  "operaciones": ["1", "2", "3"],
  "campos_insertables": ["nombre", "apellido", "telefono", "codigo", "tipo_documento"],
  "campos_actualizables": ["nombre", "apellido", "telefono", "estado"],
  "campos_criterio": ["id", "codigo", "tipo_documento", "estado", "fecha_modificacion"],
  "campos_protegidos": ["id", "nit_empresa", "created_at"],
  "borrado": "fisico",
  "reglas_antes": ["validar_cliente"],
  "reglas_despues": ["registrar_auditoria"]
}
```

Este catálogo resuelve el alias a una tabla física conocida y define exactamente qué puede escribirse o filtrarse.

### 9.3. Parseo y reutilización de parámetros

El contexto completo de parámetros se construye combinando:

```json
{
  "nit_empresa": "123456",
  "id_solicitud": "MIG-20260710-UPD-001",
  "id_origen": "FOX-CLI-001",
  "operacion": "2",
  "tabla": "cliente",
  "campos": {
    "telefono": "3101112233"
  },
  "criterio": {
    "id": 901
  }
}
```

Para cada regla se filtran los parámetros como hace actualmente `solveParams()`: solo se enlazan los placeholders utilizados por esa regla. La versión nueva debe además comprobar que todo placeholder requerido tenga valor; el comportamiento existente solo elimina parámetros sobrantes y no detecta anticipadamente parámetros faltantes.

Ejemplo conceptual:

```text
consulta de regla:
  SELECT COUNT(*) total
  FROM cliente
  WHERE nit_empresa = :nit_empresa AND id = :criterio_id

contexto disponible:
  nit_empresa, id_solicitud, id_origen, operacion, tabla,
  campo_telefono, criterio_id

parámetros enlazados:
  nit_empresa, criterio_id
```

Los nombres anidados se normalizan con prefijos inequívocos:

- `campos.telefono` se convierte en `campo_telefono`;
- `criterio.id` se convierte en `criterio_id`;
- los parámetros de contexto conservan nombres como `nit_empresa` e `id_solicitud`.

### 9.4. Formas de resultado reutilizables en reglas

Las reglas previas y posteriores pueden conservar las formas `F`, `Q`, `A`, `B` y `C`. Su salida debe quedar separada del resultado principal:

```json
{
  "reglas": {
    "antes": [
      {
        "nombre": "validar_cliente",
        "tipo": "Q",
        "resultado": 1,
        "valida": true
      }
    ],
    "despues": [
      {
        "nombre": "consultar_cliente_migrado",
        "tipo": "A",
        "resultado": [
          { "id": 901, "codigo": "CLI001" }
        ]
      }
    ]
  }
}
```

Una regla puede:

- validar y detener la operación;
- transformar o normalizar valores;
- consultar datos relacionados;
- enriquecer la respuesta;
- generar auditoría interna.

No se recomienda trasladar sin cambios el uso actual de `eval()` sobre `regla_valor`. Las comparaciones deben representarse con operadores permitidos (`eq`, `ne`, `gt`, etc.) y evaluarse sin ejecutar código dinámico.

---

## 10. Compatibilidad con el formato anterior

Durante una transición se puede aceptar `campos` como objeto único y convertirlo internamente a `registros`:

```json
{
  "tabla": "cliente",
  "operacion": "1",
  "nit_empresa": "123456",
  "campos": {
    "nombre": "Juan"
  }
}
```

Normalización interna:

```json
{
  "tabla": "cliente",
  "operacion": "1",
  "nit_empresa": "123456",
  "registros": [
    {
      "id_origen": "LEGACY-0",
      "campos": {
        "nombre": "Juan"
      }
    }
  ]
}
```

El formato anterior no ofrece idempotencia ni trazabilidad completas. Debe considerarse temporal y quedar marcado como obsoleto una vez actualizado FoxPro.

---

## 11. Errores esperados

| Código | Situación |
|---|---|
| `SOLICITUD_INVALIDA` | El JSON no cumple el esquema. |
| `SOLICITUD_DUPLICADA` | El `id_solicitud` ya fue confirmado. |
| `SOLICITUD_CONFLICTIVA` | El mismo `id_solicitud` llegó con contenido diferente. |
| `TABLA_NO_PERMITIDA` | El alias no está registrado. |
| `OPERACION_NO_PERMITIDA` | La tabla no admite esa operación. |
| `CAMPO_NO_PERMITIDO` | Un campo no está autorizado para la operación. |
| `CRITERIO_REQUERIDO` | Update o delete no contiene criterio. |
| `CRITERIO_NO_PERMITIDO` | Campo u operador de criterio no autorizado. |
| `CRITERIO_CANTIDAD_INESPERADA` | La cantidad encontrada difiere de `esperados`. |
| `PARAMETRO_REQUERIDO` | Falta un placeholder utilizado por una regla. |
| `REGLA_NO_CUMPLIDA` | Una regla de negocio impidió la operación. |
| `CONFLICTO_INTEGRIDAD` | La operación viola una restricción de base de datos. |
| `ERROR_TRANSACCION` | La transacción no pudo confirmarse. |

Ejemplo de lote atómico fallido:

```json
{
  "id_solicitud": "MIG-20260710-UPD-002",
  "operacion": "2",
  "tabla": "cliente",
  "nit_empresa": "123456",
  "modo": "ejecutar",
  "estado": "fallido",
  "atomico": true,
  "confirmado": false,
  "resumen": {
    "recibidos": 2,
    "exitosos": 0,
    "fallidos": 1,
    "filas_afectadas": 0
  },
  "resultados": [
    {
      "indice": 0,
      "id_origen": "FOX-CLI-001",
      "estado": "revertido",
      "filas_afectadas": 0,
      "llave_generada": null,
      "error": null
    },
    {
      "indice": 1,
      "id_origen": "FOX-CLI-002",
      "estado": "fallido",
      "filas_afectadas": 0,
      "llave_generada": null,
      "error": {
        "codigo": "CRITERIO_CANTIDAD_INESPERADA",
        "mensaje": "El criterio encontró 0 filas y se esperaba 1",
        "campo": "registros[1].criterio",
        "detalle": {
          "esperados": 1,
          "encontrados": 0
        }
      }
    }
  ],
  "errores": [
    {
      "codigo": "ERROR_TRANSACCION",
      "mensaje": "El lote completo fue revertido"
    }
  ]
}
```

---

## 12. Reglas de seguridad e integridad

1. `nit_empresa` es obligatorio y se aplica en el servidor, nunca se confía en un criterio enviado por el cliente.
2. Tablas, campos, operadores y operaciones se resuelven desde listas blancas.
3. Todos los valores usan parámetros enlazados; no se concatena SQL.
4. No se aceptan nombres físicos de tabla, fragmentos SQL ni `query_dato` desde FoxPro.
5. Update y delete requieren criterio no vacío y cantidad `esperados`.
6. Los lotes de migración deben usar transacciones y, por defecto, `atomico: true`.
7. `id_solicitud` debe ser idempotente y su payload debe almacenarse mediante hash para detectar conflictos.
8. Cada resultado conserva `id_origen` para reconciliar origen y destino.
9. Deben registrarse usuario técnico, fecha, empresa, operación, tabla, llaves y resultado en una bitácora.
10. Los errores públicos no deben incluir consultas SQL ni parámetros sensibles.
11. Se deben limitar cantidad de registros, tamaño del cuerpo y profundidad de criterios.
12. El orden de carga debe respetar dependencias: catálogos y padres antes que detalles; detalles antes de eliminar padres.

---

## 13. Flujo recomendado de migración

1. Registrar las tablas y sus reglas en el catálogo interno.
2. Extraer desde FoxPro un lote pequeño con `id_origen` estable.
3. Enviar el lote con `modo: validar`.
4. Corregir errores de esquema, tipos, referencias o cantidades.
5. Reenviar el mismo contenido con un nuevo `id_solicitud` y `modo: ejecutar`.
6. Guardar la respuesta completa en el sistema legacy.
7. Conciliar `id_origen`, llaves generadas y filas afectadas.
8. Avanzar al siguiente lote únicamente cuando el anterior esté confirmado.
9. Al finalizar cada entidad, comparar conteos y totales de control entre origen y destino.

Este flujo permite reintentar de forma segura, localizar el registro exacto que falló y mantener el orden entre entidades relacionadas.
