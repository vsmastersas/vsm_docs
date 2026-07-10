# Operación 1: insertar

Permite insertar uno o varios registros en una misma tabla.

## Escenario normal

Para insertar un registro, `campos` se envía como objeto.

### JSON de entrada completo

```json
{
  "operacion": "1",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "campos": {
    "codca": "001",
    "detalle": "Analista",
    "porarp": 0.522
  }
}
```

El servicio agrega `nit_empresa` a los datos que se insertarán.

### Respuesta esperada

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
      "id": 101,
      "exito": true
    }
  ]
}
```

## Escenario cuando se envían varios

Para insertar varios registros, `campos` se envía como arreglo. Cada elemento es un registro independiente.

### JSON de entrada completo

```json
{
  "operacion": "1",
  "tabla": "ncargo",
  "nit_empresa": "123456",
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
}
```

### Respuesta esperada completa

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
      "id": 101,
      "exito": true
    },
    {
      "indice": 1,
      "id": 102,
      "exito": true
    }
  ]
}
```

## Ejemplo real completo

La tabla `ncargo` es usada en `nominasimple_bk`. El código actual inserta mediante:

```php
DB::table('ncargo')->insertGetId($data);
```

El JSON completo que representa esa operación es:

```json
{
  "operacion": "1",
  "tabla": "ncargo",
  "nit_empresa": "900123456",
  "campos": {
    "codca": "AUX01",
    "detalle": "Auxiliar administrativo",
    "porarp": 0.522
  }
}
```

Respuesta:

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
      "id": 245,
      "exito": true
    }
  ]
}
```

## Reglas

- `campos` es obligatorio.
- `camposllave` no se usa en inserts.
- `nit_empresa` se incorpora internamente en cada registro.
- Cada elemento debe contener únicamente columnas permitidas para la tabla.
- El `id` devuelto corresponde al generado por `insertGetId()` cuando la tabla usa una llave autoincremental.

