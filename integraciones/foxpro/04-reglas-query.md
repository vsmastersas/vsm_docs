# Operación 4: ejecutar reglas de consulta

## Objetivo

La operación `"4"` se utiliza únicamente para ejecutar el modelo de reglas de consulta que ya existe en `nominasimple_bk` y devolver su resultado.

La operación contempla dos formas de ejecución:

1. **Modelo completo:** ejecuta las reglas habilitadas y construye la estructura jerárquica usando `id_tag_padre` y `nom_tag`.
2. **Query puntual:** busca una sola regla por su código, ejecuta únicamente su `query_dato` y devuelve el resultado de acuerdo con `tipo_dato` (`F`, `Q`, `A`, `B` o `C`).

Esta operación no inserta, actualiza ni elimina información. Tampoco recibe instrucciones SQL desde FoxPro. Las consultas deben estar definidas previamente en la tabla `nreglas`.

## Estructura del JSON

Se conserva la misma estructura base de la integración:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "docum": "1001",
    "codigo": "90001"
  }
}
```

### Descripción de los campos

| Campo | Descripción |
|---|---|
| `operacion` | Siempre debe ser `"4"`. |
| `tabla` | Nombre del modelo de reglas que se ejecutará. Para el modelo actual se usa `nreglas`. |
| `nit_empresa` | Empresa para la cual se ejecutan las consultas. Es obligatorio. |
| `campos` | Parámetros disponibles para resolver las consultas. En la ejecución puntual también contiene `codigo_query`. |

`campos` no representa columnas para insertar o modificar. En esta operación contiene únicamente los parámetros requeridos por las consultas.

## Funcionamiento del modelo existente

El flujo actual de `nominasimple_bk` es el siguiente:

1. `getReglas()` consulta las reglas habilitadas de `nreglas`.
2. Las reglas se ordenan usando `orden` e `id_tag_padre`.
3. `armarElementosHijoPadre()` construye la estructura de salida según `id_tag_padre` y `nom_tag`.
4. Si la regla contiene una consulta, esta se toma de `query_dato`.
5. `solveParams()` conserva solamente los parámetros usados en la consulta.
6. La consulta se ejecuta con parámetros nombrados.
7. `tipo_dato` define la forma del resultado.

## Ejecución de un query puntual

Esta forma permite ejecutar una sola consulta configurada sin construir toda la estructura de `nreglas`.

FoxPro envía:

- el código de la consulta que desea ejecutar;
- los parámetros que necesita esa consulta.

El código identifica una regla previamente registrada. No contiene SQL y no puede reemplazar `query_dato`.

### Estructura de entrada

Se mantiene la estructura base. `codigo_query` se incluye dentro de `campos` junto con los parámetros:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "EMPLEADO_NOMBRE",
    "codigo": "90001"
  }
}
```

Interpretación:

- `codigo_query`: identifica la regla puntual que se ejecutará;
- `codigo`: parámetro requerido por el `query_dato` de esa regla;
- `nit_empresa`: parámetro general de empresa.

El servicio debe retirar `codigo_query` antes de llamar a `solveParams()`, porque identifica la regla pero no es un parámetro SQL.

### Búsqueda de la regla

El servicio busca una sola regla habilitada usando `codigo_query`:

```text
codigo_query = EMPLEADO_NOMBRE
ind_habilita = 1
```

La regla encontrada contiene como mínimo:

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "nom_tag": "nombreEmpleado",
  "tipo_dato": "Q",
  "query_dato": "SELECT nombre FROM empleado WHERE nit_empresa = :nit_empresa AND codigo = :codigo"
}
```

Si la tabla `nreglas` todavía no tiene una columna de código estable, puede usarse inicialmente su `id`. Sin embargo, para la integración con FoxPro es preferible un código funcional como `EMPLEADO_NOMBRE`, porque no depende del identificador interno de cada ambiente.

### Flujo puntual

1. Validar que `operacion` sea `"4"`.
2. Leer `codigo_query` desde `campos`.
3. Buscar una única regla habilitada con ese código.
4. Obtener `tipo_dato` y `query_dato` desde la regla.
5. Unir `nit_empresa` con los demás valores de `campos`.
6. Retirar `codigo_query` de los parámetros de consulta.
7. Aplicar `solveParams()` para conservar únicamente los placeholders usados.
8. Ejecutar solamente esa regla.
9. Transformar el resultado según `F`, `Q`, `A`, `B` o `C`.
10. Devolver el código, el tipo y el resultado.

### Respuesta base de una ejecución puntual

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "tipo_dato": "Q",
  "resultado": "Carlos Pérez"
}
```

No se recorren las demás reglas y no se usa `armarElementosHijoPadre()`.

## Query puntual según el tipo de resultado

### Query puntual tipo `F`

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "AMBIENTE_ACTUAL"
  }
}
```

La regla tiene `tipo_dato = "F"`. No ejecuta SQL y devuelve directamente `query_dato`:

```json
{
  "codigo_query": "AMBIENTE_ACTUAL",
  "tipo_dato": "F",
  "resultado": "PRODUCCION"
}
```

### Query puntual tipo `Q`

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "EMPLEADO_NOMBRE",
    "codigo": "90001"
  }
}
```

Respuesta con el primer campo de la primera fila:

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "tipo_dato": "Q",
  "resultado": "Carlos Pérez"
}
```

### Query puntual tipo `A`

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "EMPLEADO_CONCEPTOS",
    "codigo": "90001"
  }
}
```

Respuesta con todas las filas como objetos:

```json
{
  "codigo_query": "EMPLEADO_CONCEPTOS",
  "tipo_dato": "A",
  "resultado": [
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
```

### Query puntual tipo `B`

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "CONCEPTO_DATOS",
    "codigo": "001"
  }
}
```

Respuesta con todas las columnas de la primera fila:

```json
{
  "codigo_query": "CONCEPTO_DATOS",
  "tipo_dato": "B",
  "resultado": [
    "001",
    "Salario"
  ]
}
```

### Query puntual tipo `C`

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "CONCEPTOS_CODIGOS"
  }
}
```

Respuesta con la primera columna de todas las filas:

```json
{
  "codigo_query": "CONCEPTOS_CODIGOS",
  "tipo_dato": "C",
  "resultado": [
    "001",
    "002",
    "003"
  ]
}
```

El cliente no necesita enviar `tipo_dato`. El servicio lo obtiene de la regla identificada por `codigo_query`.

## Parámetros de las consultas

Los parámetros disponibles se construyen con `nit_empresa` y los valores recibidos en `campos`.

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "docum": "1001",
    "codigo": "90001",
    "prefijo": "NE"
  }
}
```

Parámetros disponibles internamente:

```json
{
  "nit_empresa": "123456",
  "docum": "1001",
  "codigo": "90001",
  "prefijo": "NE"
}
```

Una regla puede utilizar solamente algunos de ellos:

```sql
SELECT nombre
FROM empleado
WHERE nit_empresa = :nit_empresa
  AND codigo = :codigo;
```

Para esta consulta, `solveParams()` enviará únicamente:

```json
{
  "nit_empresa": "123456",
  "codigo": "90001"
}
```

Los parámetros `docum` y `prefijo` se descartan para esa consulta porque no aparecen en `query_dato`.

## Estructura de una regla

Los campos principales usados por el modelo son:

| Campo | Descripción |
|---|---|
| `id` | Identificador de la regla. |
| `id_tag_padre` | Identificador de la regla padre. Permite construir resultados anidados. |
| `nom_tag` | Nombre de la propiedad que aparecerá en el JSON resultante. |
| `tipo_dato` | Indica cómo obtener y devolver el valor. |
| `query_dato` | Consulta parametrizada o valor fijo. |
| `ind_habilita` | Indica si la regla debe ejecutarse. |
| `orden` | Define el orden de procesamiento. |
| `regla_valor` | Condición configurada para la regla, cuando aplica. |

## Tipos de resultado

El campo `tipo_dato` determina la forma de interpretar `query_dato`.

### Tipo `F`: valor fijo

No ejecuta una consulta. Devuelve directamente el contenido de `query_dato`.

Regla conceptual:

```json
{
  "nom_tag": "ambiente",
  "tipo_dato": "F",
  "query_dato": "PRODUCCION"
}
```

Resultado:

```json
{
  "ambiente": "PRODUCCION"
}
```

### Tipo `Q`: valor único

Ejecuta la consulta y devuelve el primer campo de la primera fila.

Consulta configurada:

```sql
SELECT nombre
FROM empleado
WHERE nit_empresa = :nit_empresa
  AND codigo = :codigo;
```

Regla conceptual:

```json
{
  "nom_tag": "nombreEmpleado",
  "tipo_dato": "Q",
  "query_dato": "consulta parametrizada configurada en nreglas"
}
```

Resultado:

```json
{
  "nombreEmpleado": "Carlos Pérez"
}
```

### Tipo `A`: arreglo de objetos

Ejecuta la consulta y devuelve todas las filas conservando el nombre de sus columnas.

Resultado:

```json
{
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
```

Los campos cuyo valor sea `null` o una cadena vacía se retiran de cada objeto, de acuerdo con el comportamiento actual de `armarElementosHijoPadre()`.

### Tipo `B`: columnas de la primera fila

Ejecuta la consulta y devuelve como arreglo los valores de todas las columnas de la primera fila.

Si la primera fila de la consulta es:

```json
{
  "codigo": "001",
  "descripcion": "Salario"
}
```

El resultado será:

```json
{
  "concepto": [
    "001",
    "Salario"
  ]
}
```

### Tipo `C`: primera columna de todas las filas

Ejecuta la consulta y devuelve el primer valor de cada fila.

Si la consulta devuelve varios códigos, el resultado será:

```json
{
  "codigos": [
    "001",
    "002",
    "003"
  ]
}
```

## Estructura jerárquica

Las reglas pueden tener relaciones padre-hijo mediante `id_tag_padre`.

Ejemplo conceptual de reglas:

| `id` | `id_tag_padre` | `nom_tag` | `tipo_dato` |
|---:|---:|---|---|
| 1 | 0 | `empleado` | — |
| 2 | 1 | `nombre` | `Q` |
| 3 | 1 | `documento` | `Q` |
| 4 | 1 | `conceptos` | `A` |

Resultado construido por el modelo:

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

Este escenario corresponde a la ejecución del **modelo completo**. No se envía `codigo_query`, por lo que el servicio recorre las reglas habilitadas y construye la estructura jerárquica.

### JSON de entrada completo

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo": "90001",
    "docum": "1001"
  }
}
```

### Respuesta esperada

La respuesta corresponde directamente a la estructura construida por las reglas:

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

## Ejecución con distintos parámetros

La misma estructura de reglas puede reutilizarse enviando valores diferentes en `campos`.

Primera solicitud:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo": "90001",
    "docum": "1001"
  }
}
```

Segunda solicitud:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo": "90002",
    "docum": "1002"
  }
}
```

Cada solicitud ejecuta el mismo modelo, pero las consultas reciben parámetros diferentes. No se mezclan ambos resultados en una operación de escritura ni se modifican registros.

## Respuesta cuando una consulta falla

El modelo actual genera información de error mediante `stackErrors()`:

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

La consulta y sus parámetros son útiles para diagnóstico interno. Si la respuesta se expone fuera del sistema, se recomienda no devolver información sensible.

## Reglas de implementación

- La operación `"4"` es exclusivamente de consulta.
- FoxPro envía parámetros en `campos`; no envía SQL.
- Si `campos` contiene `codigo_query`, se ejecuta únicamente esa regla.
- Si `campos` no contiene `codigo_query`, se ejecuta el modelo completo y se arma la estructura jerárquica.
- `codigo_query` identifica la regla, pero no se envía como parámetro a la consulta.
- El cliente no envía `tipo_dato`; se utiliza el tipo configurado en la regla.
- Las consultas se obtienen de `query_dato` en las reglas habilitadas.
- Los valores se enlazan mediante parámetros nombrados.
- `solveParams()` elimina los parámetros que una consulta particular no usa.
- Antes de ejecutar se debe validar que estén presentes todos los parámetros requeridos por la consulta.
- `nit_empresa` debe utilizarse en las consultas que accedan a información de una empresa.
- Se conservan las formas de resultado `F`, `Q`, `A`, `B` y `C` del modelo existente.
- `id_tag_padre` y `nom_tag` determinan la estructura final del JSON.
- La ejecución no debe producir inserts, updates ni deletes.
