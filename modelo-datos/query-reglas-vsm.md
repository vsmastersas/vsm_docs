# Modelo `query_reglas` adaptado para VSM

## Objetivo

Definir el catálogo de consultas puntuales que utilizará VSM para atender la operación `"4"` de la integración FoxPro.

Este modelo toma como referencia la ejecución de `query_dato` y los tipos `F`, `Q`, `A`, `B` y `C` de `nreglas`, pero se separa del modelo jerárquico. `query_reglas` ejecuta una consulta puntual por código; no arma estructuras padre-hijo.

## Alcance

`query_reglas` permite:

- registrar una consulta con un código estable;
- recibir parámetros desde FoxPro;
- ejecutar únicamente consultas habilitadas;
- reutilizar el filtrado de parámetros de `solveParams()`;
- transformar el resultado según `F`, `Q`, `A`, `B` o `C`;
- devolver una respuesta uniforme.

No permite que FoxPro envíe SQL. FoxPro solo envía el código de la regla y sus parámetros.

## Mapeo del modelo anterior a VSM

| `nreglas` | `query_reglas` VSM | Observación |
|---|---|---|
| `id` | `id` | Identificador interno. |
| No existe | `codigo` | Código funcional y único usado por la integración. |
| `nom_tag` | `nombre_salida` | Nombre descriptivo del resultado. |
| `tipo_dato` | `tipo_resultado` | Conserva los valores `F`, `Q`, `A`, `B` y `C`. |
| `query_dato` | `consulta` | SQL parametrizado o valor fijo para tipo `F`. |
| `ind_habilita` | `activo` | Determina si puede ejecutarse. |
| `id_tag_padre` | No aplica | La operación 4 no construye jerarquías. |
| `orden` | No aplica | Solo se ejecuta una regla. |
| `regla_valor` | No aplica | No se evalúan condiciones del árbol completo. |

## Estructura propuesta

| Campo | Tipo sugerido | Nulo | Descripción |
|---|---|---:|---|
| `id` | bigint | No | Llave primaria interna. |
| `codigo` | varchar(100) | No | Identificador funcional único, por ejemplo `CARGO_DETALLE`. |
| `nombre_salida` | varchar(100) | No | Nombre legible del resultado. |
| `tipo_resultado` | char(1) | No | `F`, `Q`, `A`, `B` o `C`. |
| `consulta` | text | No | Consulta parametrizada o valor fijo. |
| `activo` | boolean/tinyint | No | Indica si la regla puede ejecutarse. |

Este es el modelo mínimo. `nit_empresa` y los demás parámetros no se almacenan en la tabla; se reciben en cada solicitud.

## SQL de referencia

```sql
CREATE TABLE query_reglas (
    id BIGINT NOT NULL AUTO_INCREMENT,
    codigo VARCHAR(100) NOT NULL,
    nombre_salida VARCHAR(100) NOT NULL,
    tipo_resultado CHAR(1) NOT NULL,
    consulta TEXT NOT NULL,
    activo TINYINT(1) NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uq_query_reglas_codigo (codigo)
);
```

La sintaxis puede ajustarse al motor de base de datos usado por VSM. Las restricciones funcionales deben mantenerse aunque cambie el tipo físico.

## Restricciones del modelo

### Código único

`codigo` debe identificar exactamente una regla. Debe ser estable entre desarrollo, pruebas y producción.

Ejemplos:

```text
CARGO_DETALLE
CARGOS_EMPRESA
CARGO_DATOS
CODIGOS_CARGO
AMBIENTE_ACTUAL
```

### Tipo de resultado

`tipo_resultado` solo admite:

| Tipo | Comportamiento |
|---|---|
| `F` | Devuelve `consulta` como valor fijo; no ejecuta SQL. |
| `Q` | Devuelve el primer campo de la primera fila. |
| `A` | Devuelve todas las filas como objetos. |
| `B` | Devuelve todas las columnas de la primera fila como arreglo. |
| `C` | Devuelve la primera columna de todas las filas como arreglo. |

### Consulta

Para `Q`, `A`, `B` y `C`, `consulta` debe contener una sola sentencia de lectura y utilizar parámetros nombrados:

```sql
SELECT detalle
FROM ncargo
WHERE nit_empresa = :nit_empresa
  AND codca = :codca;
```

No se permiten instrucciones de escritura ni varias sentencias en una misma regla.

## Registros de ejemplo

### Tipo `F`

```sql
INSERT INTO query_reglas
    (codigo, nombre_salida, tipo_resultado, consulta, activo)
VALUES
    ('AMBIENTE_ACTUAL', 'ambiente', 'F', 'PRODUCCION', 1);
```

### Tipo `Q`

```sql
INSERT INTO query_reglas
    (codigo, nombre_salida, tipo_resultado, consulta, activo)
VALUES
    (
        'CARGO_DETALLE',
        'detalleCargo',
        'Q',
        'SELECT detalle FROM ncargo WHERE nit_empresa = :nit_empresa AND codca = :codca',
        1
    );
```

### Tipo `A`

```sql
INSERT INTO query_reglas
    (codigo, nombre_salida, tipo_resultado, consulta, activo)
VALUES
    (
        'CARGOS_EMPRESA',
        'cargos',
        'A',
        'SELECT codca, detalle, porarp FROM ncargo WHERE nit_empresa = :nit_empresa ORDER BY detalle',
        1
    );
```

### Tipo `B`

```sql
INSERT INTO query_reglas
    (codigo, nombre_salida, tipo_resultado, consulta, activo)
VALUES
    (
        'CARGO_DATOS',
        'cargo',
        'B',
        'SELECT codca, detalle, porarp FROM ncargo WHERE nit_empresa = :nit_empresa AND codca = :codca',
        1
    );
```

### Tipo `C`

```sql
INSERT INTO query_reglas
    (codigo, nombre_salida, tipo_resultado, consulta, activo)
VALUES
    (
        'CODIGOS_CARGO',
        'codigosCargo',
        'C',
        'SELECT codca FROM ncargo WHERE nit_empresa = :nit_empresa ORDER BY codca',
        1
    );
```

## Entrada de la operación 4

La estructura pública de la integración se mantiene:

```json
{
  "operacion": "4",
  "tabla": "query_reglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "CARGO_DETALLE",
    "codca": "001"
  }
}
```

### Correspondencia con el modelo

| Entrada | Uso interno |
|---|---|
| `campos.codigo_query` | Busca `query_reglas.codigo`. |
| `nit_empresa` | Parámetro disponible como `:nit_empresa`. |
| `campos.codca` | Parámetro disponible como `:codca`. |

El nombre externo `codigo_query` se conserva para que el JSON sea explícito. Internamente se compara con la columna `codigo`.

## Consulta para obtener la regla

```sql
SELECT
    id,
    codigo,
    nombre_salida,
    tipo_resultado,
    consulta
FROM query_reglas
WHERE codigo = :codigo_query
  AND activo = 1;
```

La búsqueda debe devolver exactamente un registro.

## Resolución de parámetros

Para esta entrada:

```json
{
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "CARGO_DETALLE",
    "codca": "001",
    "otro_valor": "NO_USADO"
  }
}
```

El contexto inicial de ejecución es:

```json
{
  "nit_empresa": "123456",
  "codca": "001",
  "otro_valor": "NO_USADO"
}
```

`codigo_query` se retira porque identifica la regla y no es un parámetro de la consulta.

Después de aplicar la lógica de `solveParams()` a la consulta del ejemplo:

```json
{
  "nit_empresa": "123456",
  "codca": "001"
}
```

`otro_valor` se descarta porque la consulta no contiene `:otro_valor`.

## Flujo VSM propuesto

```text
recibir JSON
  -> validar operacion = 4
  -> validar tabla = query_reglas
  -> extraer campos.codigo_query
  -> consultar query_reglas por codigo y activo
  -> obtener consulta y tipo_resultado
  -> combinar nit_empresa con campos
  -> retirar codigo_query
  -> filtrar parámetros no utilizados
  -> validar placeholders faltantes
  -> ejecutar según F/Q/A/B/C
  -> devolver codigo_query, tipo_dato y resultado
```

## Correspondencia con los métodos existentes

| Tipo VSM | Lógica reutilizable de `nominasimple_bk` |
|---|---|
| `F` | Retornar directamente el valor fijo. |
| `Q` | `executeQuery()` y `getSingleValueQuery()`. |
| `A` | `executeQueryArray()`. |
| `B` | `executeQueryArray()` y `getQueryValue()`. |
| `C` | `executeQueryArray()` y `getQueryValue2()`. |
| Todos | Filtrado de parámetros equivalente a `solveParams()`. |

VSM puede implementar los mismos comportamientos con sus propios servicios o repositorios; no es necesario copiar literalmente los traits de Laravel.

## Respuesta tipo `Q`

```json
{
  "codigo_query": "CARGO_DETALLE",
  "tipo_dato": "Q",
  "resultado": "Analista"
}
```

## Respuesta tipo `A`

```json
{
  "codigo_query": "CARGOS_EMPRESA",
  "tipo_dato": "A",
  "resultado": [
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

## Validaciones requeridas

- `codigo_query` es obligatorio.
- `codigo_query` debe existir una sola vez en `query_reglas`.
- La regla debe estar activa.
- `tipo_resultado` debe ser `F`, `Q`, `A`, `B` o `C`.
- Para tipos distintos de `F`, `consulta` debe ser una única sentencia de lectura.
- No se admiten `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `DROP` ni múltiples sentencias.
- Todos los placeholders deben tener un valor disponible.
- Los valores se enlazan como parámetros; nunca se concatenan en el SQL.
- Las consultas de información empresarial deben incluir `:nit_empresa`.
- Deben aplicarse límites de tiempo y cantidad de filas.

## Errores mínimos

### Código inexistente o inactivo

```json
{
  "codigo_query": "CARGO_DETALLE",
  "tipo_dato": null,
  "resultado": null,
  "error": "No existe una regla activa con el código enviado"
}
```

### Parámetro faltante

```json
{
  "codigo_query": "CARGO_DETALLE",
  "tipo_dato": "Q",
  "resultado": null,
  "error": "Falta el parámetro requerido: codca"
}
```

## Recomendación para VSM

Usar `query_reglas` como catálogo independiente para la operación 4. El modelo completo de `nreglas`, con relaciones padre-hijo, debe mantenerse separado para la operación 5.

La primera versión no necesita una tabla adicional de parámetros. Los placeholders pueden detectarse desde `consulta` y validarse contra los valores recibidos. Si posteriormente VSM necesita tipar parámetros, documentarlos o asignar valores predeterminados, podrá agregarse un modelo hijo `query_reglas_parametros` sin cambiar el contrato JSON.
