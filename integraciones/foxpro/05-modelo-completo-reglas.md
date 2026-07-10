# Operación 5: ejecutar el modelo completo de reglas

## Objetivo

La operación `"5"` ejecuta el modelo completo existente en `nominasimple_bk` y construye una respuesta jerárquica a partir de todas las reglas habilitadas.

Esta operación utiliza `getReglas()` y `armarElementosHijoPadre()`. No ejecuta una regla puntual por código; para eso se usa la operación `"4"`.

La operación es únicamente de lectura. No inserta, actualiza ni elimina información.

## Estructura del JSON

```json
{
  "operacion": "5",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo": "90001",
    "docum": "1001",
    "prefijo": "NE"
  }
}
```

### Campos

| Campo | Descripción |
|---|---|
| `operacion` | Siempre debe ser `"5"`. |
| `tabla` | Para el modelo actual se usa `nreglas`. |
| `nit_empresa` | Empresa para la cual se ejecutarán las reglas. |
| `campos` | Parámetros disponibles para las consultas de todas las reglas. |

En esta operación no se envía `codigo_query`, porque no se selecciona una regla puntual.

## Flujo del modelo existente

1. `getReglas()` consulta las reglas habilitadas de `nreglas`.
2. Las reglas se ordenan por `orden` e `id_tag_padre`.
3. Para cada regla, `query_dato` contiene un valor fijo o una consulta parametrizada.
4. `solveParams()` conserva únicamente los parámetros usados por cada consulta.
5. `tipo_dato` determina cómo interpretar el resultado de cada regla.
6. `armarElementosHijoPadre()` relaciona las reglas mediante `id_tag_padre`.
7. `nom_tag` define el nombre de cada propiedad en el JSON.
8. `transformArrayParents()` consolida la estructura final.
9. Se eliminan valores vacíos, nulos y arreglos vacíos según el comportamiento actual.

## Estructura de una regla

| Campo | Descripción |
|---|---|
| `id` | Identificador de la regla. |
| `id_tag_padre` | Regla padre dentro de la estructura. |
| `nom_tag` | Nombre de la propiedad en el JSON resultante. |
| `tipo_dato` | Forma en que se obtiene y transforma el resultado. |
| `query_dato` | Consulta parametrizada o valor fijo. |
| `ind_habilita` | Indica si la regla debe ejecutarse. |
| `orden` | Orden de procesamiento. |
| `regla_valor` | Condición configurada para la regla, cuando aplica. |

## Tipos utilizados por el modelo

| Tipo | Resultado |
|---|---|
| `F` | Devuelve `query_dato` como valor fijo. |
| `Q` | Primer campo de la primera fila. |
| `A` | Todas las filas como arreglo de objetos. |
| `B` | Todas las columnas de la primera fila como arreglo. |
| `C` | Primera columna de todas las filas como arreglo. |

Cada regla puede usar un tipo diferente. El modelo combina todos sus resultados para construir el JSON final.

## Resolución de parámetros

Los parámetros disponibles se forman con `nit_empresa` y los valores de `campos`:

```json
{
  "nit_empresa": "123456",
  "codigo": "90001",
  "docum": "1001",
  "prefijo": "NE"
}
```

Si una regla contiene:

```sql
SELECT nombre
FROM empleado
WHERE nit_empresa = :nit_empresa
  AND codigo = :codigo;
```

`solveParams()` enviará solamente:

```json
{
  "nit_empresa": "123456",
  "codigo": "90001"
}
```

Los mismos parámetros de entrada se reutilizan en todas las reglas, pero cada consulta recibe únicamente los que necesita.

## Construcción jerárquica

Ejemplo conceptual:

| `id` | `id_tag_padre` | `nom_tag` | `tipo_dato` |
|---:|---:|---|---|
| 1 | 0 | `empleado` | — |
| 2 | 1 | `nombre` | `Q` |
| 3 | 1 | `documento` | `Q` |
| 4 | 1 | `conceptos` | `A` |

La relación padre-hijo produce:

```json
{
  "empleado": {
    "nombre": "Carlos Pérez",
    "documento": "1001",
    "conceptos": [
      {
        "codigo": "001",
        "valor": 1500000
      },
      {
        "codigo": "002",
        "valor": 162000
      }
    ]
  }
}
```

## Escenario normal y respuesta esperada

### JSON de entrada completo

```json
{
  "operacion": "5",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo": "90001",
    "docum": "1001",
    "prefijo": "NE"
  }
}
```

### Respuesta esperada

La respuesta es la estructura completa construida por las reglas:

```json
{
  "empleado": {
    "nombre": "Carlos Pérez",
    "documento": "1001",
    "conceptos": [
      {
        "codigo": "001",
        "descripcion": "Salario"
      },
      {
        "codigo": "002",
        "descripcion": "Auxilio de transporte"
      }
    ]
  }
}
```

## Reutilización con otros parámetros

La misma estructura se puede generar para otro registro cambiando únicamente `campos`:

```json
{
  "operacion": "5",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo": "90002",
    "docum": "1002",
    "prefijo": "NE"
  }
}
```

Se recorren las mismas reglas, pero las consultas reciben los nuevos parámetros.

## Respuesta cuando una regla falla

El modelo actual usa `stackErrors()` para describir el error:

```json
{
  "code": "HY093",
  "mensaje": "Falta un parámetro requerido por la consulta",
  "query": "consulta configurada en nreglas",
  "id_query": 15,
  "params": {
    "nit_empresa": "123456"
  }
}
```

La consulta y los parámetros deben conservarse para diagnóstico interno. Si la respuesta se expone fuera del sistema, debe evitarse devolver información sensible.

## Reglas de implementación

- La operación `"5"` siempre ejecuta el modelo completo.
- No recibe `codigo_query`.
- Ejecuta las reglas habilitadas en el orden configurado.
- FoxPro envía parámetros en `campos`; no envía SQL.
- Las consultas se obtienen de `query_dato`.
- Los valores se enlazan mediante parámetros nombrados.
- `solveParams()` elimina parámetros sobrantes para cada consulta.
- Deben validarse los parámetros faltantes antes de ejecutar.
- Se conservan los tipos `F`, `Q`, `A`, `B` y `C`.
- `id_tag_padre` y `nom_tag` determinan la estructura final.
- `nit_empresa` debe usarse en consultas que accedan a datos de una empresa.
- La operación no modifica información.
