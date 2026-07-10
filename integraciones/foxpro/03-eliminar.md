# Operación 3: eliminar

Permite eliminar uno o varios registros. Toda eliminación debe indicar `camposllave`.

## Escenario normal

Para eliminar un registro no se envía `campos`; solamente se envía `camposllave`.

### JSON de entrada completo

```json
{
  "operacion": "3",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "camposllave": {
    "id": 101
  }
}
```

### Respuesta esperada

```json
{
  "exito": true,
  "operacion": "3",
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

## Escenario cuando se envían varios

Para eliminar varios registros, `camposllave` se envía como arreglo. Cada elemento es un criterio independiente.

### JSON de entrada completo

```json
{
  "operacion": "3",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "camposllave": [
    {
      "id": 101
    },
    {
      "codca": "002"
    }
  ]
}
```

### Respuesta esperada completa

```json
{
  "exito": true,
  "operacion": "3",
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

El código actual de `nominasimple_bk` elimina cargos por `id`:

```php
DB::table('ncargo')->where(['id' => $data['id']])->delete();
```

El contrato permite mantener esa forma y también definir una llave compuesta:

```json
{
  "operacion": "3",
  "tabla": "ncargo",
  "nit_empresa": "900123456",
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
  "operacion": "3",
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
DELETE FROM ncargo
WHERE codca = :codca
  AND porarp = :porarp
  AND nit_empresa = :nit_empresa;
```

## Respuesta cuando un registro falla

```json
{
  "exito": false,
  "operacion": "3",
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

- `camposllave` es obligatorio.
- `campos` no se envía.
- `camposllave` puede tener una llave simple o compuesta.
- Para varios registros, `camposllave` es un arreglo.
- Los campos de cada criterio se combinan con `AND`.
- `nit_empresa` siempre se agrega al criterio.
- No se permite un delete con `camposllave` vacío.

