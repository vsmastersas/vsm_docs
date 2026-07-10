# Reglas de query y parámetros

## Objetivo

Reutilizar el modelo existente de `nominasimple_bk` para ejecutar consultas configuradas con parámetros, sin cambiar la estructura base del JSON de FoxPro.

## Modelo existente

Las reglas se consultan desde la tabla `nreglas`. Sus consultas están almacenadas en `query_dato` y usan parámetros nombrados:

```sql
SELECT detalle
FROM ncargo
WHERE nit_empresa = :nit_empresa
  AND codca = :codca;
```

El método existente `solveParams()` recibe todos los parámetros disponibles y conserva únicamente los usados por la consulta:

```php
public function solveParams($params, $qry)
{
    foreach ($params as $key => $value) {
        if (!strstr($qry, ":" . $key)) {
            unset($params[$key]);
        }
    }
    return $params;
}
```

## Parámetros según la operación

### Insertar

Entrada:

```json
{
  "operacion": "1",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "campos": {
    "codca": "001",
    "detalle": "Analista"
  }
}
```

Parámetros disponibles para una regla:

```json
{
  "operacion": "1",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "codca": "001",
  "detalle": "Analista"
}
```

### Actualizar

Entrada:

```json
{
  "operacion": "2",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "campos": {
    "detalle": "Analista senior"
  },
  "camposllave": {
    "id": 101
  }
}
```

Para evitar que un nombre de `campos` choque con uno de `camposllave`, los criterios se exponen con el prefijo `llave_`:

```json
{
  "operacion": "2",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "detalle": "Analista senior",
  "llave_id": 101
}
```

Consulta de regla:

```sql
SELECT COUNT(*) AS total
FROM ncargo
WHERE nit_empresa = :nit_empresa
  AND id = :llave_id;
```

### Eliminar

Entrada:

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

Parámetros disponibles:

```json
{
  "operacion": "3",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "llave_id": 101
}
```

## Formas de resultado existentes

El motor actual usa `tipo_dato` para decidir cómo interpretar `query_dato`:

| Tipo | Forma |
|---|---|
| `F` | Valor fijo; no ejecuta una consulta. |
| `Q` | Primer campo de la primera fila. |
| `A` | Todas las filas como arreglo de objetos. |
| `B` | Todas las columnas de la primera fila como arreglo. |
| `C` | Primera columna de todas las filas como arreglo. |

### Ejemplo tipo `Q`

Consulta:

```sql
SELECT COUNT(*) AS total
FROM ncargo
WHERE nit_empresa = :nit_empresa;
```

Resultado de la regla:

```json
1
```

### Ejemplo tipo `A`

Consulta:

```sql
SELECT codca, detalle
FROM ncargo
WHERE nit_empresa = :nit_empresa;
```

Resultado de la regla:

```json
[
  {
    "codca": "001",
    "detalle": "Analista"
  },
  {
    "codca": "002",
    "detalle": "Coordinador"
  }
]
```

### Ejemplo tipo `B`

Resultado de la primera fila:

```json
[
  "001",
  "Analista"
]
```

### Ejemplo tipo `C`

Resultado de la primera columna de todas las filas:

```json
[
  "001",
  "002"
]
```

## Reglas para reutilizar el modelo

- Las consultas deben definirse internamente; FoxPro no envía SQL.
- `tabla`, los nombres de campos y `camposllave` deben validarse contra listas permitidas.
- Los valores se enlazan como parámetros.
- `solveParams()` puede reutilizarse para quitar parámetros sobrantes.
- Antes de ejecutar también debe validarse que no falte ningún parámetro requerido por la consulta.
- `nit_empresa` debe formar parte de toda consulta que trabaje con información de una empresa.
- Las formas `F`, `Q`, `A`, `B` y `C` se conservan porque ya existen en el modelo actual.
- Las reglas de comparación deben resolverse con operadores conocidos; no se debe ejecutar código dinámico mediante `eval()`.
