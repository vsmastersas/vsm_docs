# Operación 2: actualizar

Permite actualizar uno o varios registros. Toda actualización debe indicar `camposllave`.

## Escenario normal

Para actualizar un registro, `campos` contiene solamente los valores que cambiarán y `camposllave` contiene el criterio.

### JSON de entrada completo

```json
{
  "operacion": "2",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "campos": {
    "detalle": "Analista senior",
    "porarp": 1.044
  },
  "camposllave": {
    "id": 101
  }
}
```

### Respuesta esperada

```json
{
  "exito": true,
  "operacion": "2",
  "tabla": "ncargo",
  "registros_recibidos": 1,
  "registros_afectados": 1,
  "resultados": [
    {
      "indice": 0,
      "registros_afectados": 1,
      "exito": true
    }
  ]
}
```

El criterio interno será `id = 101 AND nit_empresa = '123456'`.

## Escenario cuando se envían varios

Para actualizar varios registros, `campos` se envía como arreglo. Cada elemento contiene sus datos y su propio `camposllave`.

### JSON de entrada completo

```json
{
  "operacion": "2",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "campos": [
    {
      "detalle": "Analista senior",
      "porarp": 1.044,
      "camposllave": {
        "id": 101
      }
    },
    {
      "detalle": "Coordinador de nómina",
      "camposllave": {
        "codca": "002"
      }
    }
  ]
}
```

### Respuesta esperada completa

```json
{
  "exito": true,
  "operacion": "2",
  "tabla": "ncargo",
  "registros_recibidos": 2,
  "registros_afectados": 2,
  "resultados": [
    {
      "indice": 0,
      "registros_afectados": 1,
      "exito": true
    },
    {
      "indice": 1,
      "registros_afectados": 1,
      "exito": true
    }
  ]
}
```

## Ejemplo real completo con criterio compuesto

`camposllave` también puede contener varios campos. Todos se combinan con `AND`.

```json
{
  "operacion": "2",
  "tabla": "ncargo",
  "nit_empresa": "900123456",
  "campos": {
    "detalle": "Auxiliar de nómina"
  },
  "camposllave": {
    "codca": "AUX01",
    "porarp": 0.522
  }
}
```

Respuesta:

```json
{
  "exito": true,
  "operacion": "2",
  "tabla": "ncargo",
  "registros_recibidos": 1,
  "registros_afectados": 1,
  "resultados": [
    {
      "indice": 0,
      "registros_afectados": 1,
      "exito": true
    }
  ]
}
```

La consulta parametrizada equivalente es:

```sql
UPDATE ncargo
SET detalle = :detalle
WHERE codca = :codca
  AND porarp = :porarp
  AND nit_empresa = :nit_empresa;
```

## Respuesta cuando un registro falla

```json
{
  "exito": false,
  "operacion": "2",
  "tabla": "ncargo",
  "registros_recibidos": 1,
  "registros_afectados": 0,
  "resultados": [
    {
      "indice": 0,
      "registros_afectados": 0,
      "exito": false,
      "error": "No se encontró un registro con el criterio enviado"
    }
  ]
}
```

## Reglas

- `campos` y `camposllave` son obligatorios.
- `campos` contiene únicamente valores que cambiarán.
- `camposllave` puede tener una llave simple o compuesta.
- Los campos de `camposllave` se combinan con `AND`.
- `nit_empresa` siempre se agrega al criterio.
- No se permite un update con `camposllave` vacío.

