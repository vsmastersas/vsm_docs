# Ejemplos cURL de la integración FoxPro

Esta página reúne los comandos `curl` disponibles para probar la integración FoxPro contra el backend local.

## Preparación

Iniciar los servicios desde el repositorio `vsm_web_backend`:

```bash
docker compose up --build
```

La API queda disponible en:

```text
http://localhost:5000
```

Actualizar el catálogo local de tablas y columnas antes de ejecutar las operaciones, y nuevamente cuando cambie la estructura de MySQL:

```bash
curl --request POST 'http://localhost:5000/api/schema/refresh'
```

Respuesta esperada:

```json
{
  "exito": true,
  "tablas_actualizadas": 1,
  "columnas_actualizadas": 5
}
```

## Operación 1: insertar

Endpoint:

```text
POST /api/integraciones/foxpro
```

### Escenario 1: insertar un registro

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{
    "operacion": "1",
    "tabla": "ncargo",
    "nit_empresa": "123456",
    "nom_usuario": "usuario.pruebas",
    "campos": {
      "codca": "001",
      "detalle": "Analista",
      "porarp": 0.522
    }
  }'
```

Respuesta esperada:

```json
{
  "exito": true,
  "operacion": "1",
  "tabla": "ncargo",
  "registros_recibidos": 1,
  "registros_afectados": 1,
  "resultados": [
    {
      "indice": 0,
      "id": 1,
      "exito": true
    }
  ]
}
```

El valor de `id` depende del estado actual de la base de datos.

### Escenario 2: insertar varios registros

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{
    "operacion": "1",
    "tabla": "ncargo",
    "nit_empresa": "123456",
    "nom_usuario": "usuario.pruebas",
    "campos": [
      {
        "codca": "001",
        "detalle": "Analista",
        "porarp": 0.522
      },
      {
        "codca": "002",
        "detalle": "Coordinador",
        "porarp": 1.044
      }
    ]
  }'
```

Respuesta esperada:

```json
{
  "exito": true,
  "operacion": "1",
  "tabla": "ncargo",
  "registros_recibidos": 2,
  "registros_afectados": 2,
  "resultados": [
    {
      "indice": 0,
      "id": 2,
      "exito": true
    },
    {
      "indice": 1,
      "id": 3,
      "exito": true
    }
  ]
}
```

Los valores de `id` dependen del estado actual de la base de datos. La inserción múltiple se ejecuta dentro de una sola transacción.

`nom_usuario` es obligatorio y admite máximo 50 caracteres. Cada registro puede enviar `ind_estado` con valor `1` o `0`; cuando se omite, el backend asigna `1`. `created_at` y `updated_at` son generados por el servidor.

## Operación 2: actualizar

### Escenario 1: actualizar un registro

```bash
curl --request PUT 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{
    "operacion": "2",
    "tabla": "ncargo",
    "nit_empresa": "123456",
    "nom_usuario": "usuario.actualiza",
    "campos": {
      "detalle": "Analista senior",
      "porarp": 1.044
    },
    "camposllave": {
      "id": 1
    }
  }'
```

Respuesta esperada:

```json
{
  "exito": true,
  "operacion": "2",
  "tabla": "ncargo",
  "registros_recibidos": 1,
  "registros_afectados": 1,
  "resultados": [
    {"indice": 0, "registros_afectados": 1, "exito": true}
  ]
}
```

### Escenario 2: actualizar varios registros

```bash
curl --request PUT 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{
    "operacion": "2",
    "tabla": "ncargo",
    "nit_empresa": "123456",
    "nom_usuario": "usuario.actualiza",
    "campos": [
      {
        "detalle": "Analista senior",
        "porarp": 1.044,
        "camposllave": {"id": 1}
      },
      {
        "detalle": "Coordinador de nómina",
        "camposllave": {"codca": "002"}
      }
    ]
  }'
```

Respuesta esperada:

```json
{
  "exito": true,
  "operacion": "2",
  "tabla": "ncargo",
  "registros_recibidos": 2,
  "registros_afectados": 2,
  "resultados": [
    {"indice": 0, "registros_afectados": 1, "exito": true},
    {"indice": 1, "registros_afectados": 1, "exito": true}
  ]
}
```

Cada criterio incorpora internamente `nit_empresa`. La actualización múltiple se ejecuta dentro de una sola transacción.

La operación es idempotente: repetir la misma actualización sobre un registro que ya tiene esos valores devuelve nuevamente una respuesta exitosa con `registros_afectados: 1`.

`nom_usuario` es obligatorio en el nivel principal, admite máximo 50 caracteres y se registra en todas las filas actualizadas. `ind_estado` solo cambia cuando se incluye explícitamente en `campos`, siempre con valor `1` o `0`. Toda actualización válida renueva `updated_at`.

## Operación 3: eliminar

### Escenario 1: eliminar un registro

```bash
curl --request DELETE 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{
    "operacion": "3",
    "tabla": "ncargo2",
    "nit_empresa": "123456",
    "nom_usuario": "usuario.elimina",
    "camposllave": {
      "id": 1
    }
  }'
```

Respuesta esperada:

```json
{
  "exito": true,
  "operacion": "3",
  "tabla": "ncargo2",
  "registros_recibidos": 1,
  "registros_afectados": 1,
  "resultados": [
    {"indice": 0, "registros_afectados": 1, "exito": true}
  ]
}
```

### Escenario 2: eliminar varios registros

```bash
curl --request DELETE 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{
    "operacion": "3",
    "tabla": "ncargo2",
    "nit_empresa": "123456",
    "nom_usuario": "usuario.elimina",
    "camposllave": [
      {"id": 1},
      {"codca": "002"}
    ]
  }'
```

Respuesta esperada:

```json
{
  "exito": true,
  "operacion": "3",
  "tabla": "ncargo2",
  "registros_recibidos": 2,
  "registros_afectados": 2,
  "resultados": [
    {"indice": 0, "registros_afectados": 1, "exito": true},
    {"indice": 1, "registros_afectados": 1, "exito": true}
  ]
}
```

Cada criterio incorpora internamente `nit_empresa`. La eliminación múltiple usa una sola transacción y revierte todo si algún criterio no encuentra registros.

`nom_usuario` es obligatorio. La operación realiza borrado lógico: conserva los registros y asigna `ind_estado = 0`, además de actualizar `updated_at`. El cambio queda registrado en `vmauditoria`.

## Operación 4: consulta puntual

Todas las consultas usan `POST`, la tabla `query_reglas` y un `codigo_query` configurado previamente.

### Tipo F: valor fijo

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{"operacion":"4","tabla":"query_reglas","nit_empresa":"123456","campos":{"codigo_query":"AMBIENTE_ACTUAL"}}'
```

### Tipo Q: primer valor

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{"operacion":"4","tabla":"query_reglas","nit_empresa":"123456","campos":{"codigo_query":"CARGO_DETALLE","codca":"002"}}'
```

### Tipo A: arreglo de objetos

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{"operacion":"4","tabla":"query_reglas","nit_empresa":"123456","campos":{"codigo_query":"CARGOS_EMPRESA"}}'
```

### Tipo B: columnas de la primera fila

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{"operacion":"4","tabla":"query_reglas","nit_empresa":"123456","campos":{"codigo_query":"CARGO_DATOS","codca":"002"}}'
```

### Tipo C: primera columna de todas las filas

```bash
curl --request POST 'http://localhost:5000/api/integraciones/foxpro' \
  --header 'Content-Type: application/json' \
  --data '{"operacion":"4","tabla":"query_reglas","nit_empresa":"123456","campos":{"codigo_query":"CODIGOS_CARGO"}}'
```
