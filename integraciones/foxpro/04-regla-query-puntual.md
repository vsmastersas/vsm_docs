# Operación 4: ejecutar una regla de consulta puntual

## Objetivo

La operación `"4"` ejecuta una sola regla de consulta previamente configurada y devuelve únicamente su resultado.

FoxPro envía el código de la regla y sus parámetros. El servicio obtiene internamente `query_dato` y `tipo_dato`. FoxPro no envía SQL ni decide cómo interpretar la respuesta.

Esta operación es únicamente de lectura. No inserta, actualiza ni elimina información.

## Estructura del JSON

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

### Campos

| Campo | Descripción |
|---|---|
| `operacion` | Siempre debe ser `"4"`. |
| `tabla` | Para el modelo actual se usa `nreglas`. |
| `nit_empresa` | Empresa para la cual se ejecutará la consulta. |
| `campos.codigo_query` | Código estable de la regla que se ejecutará. |
| `campos` | Además de `codigo_query`, contiene los parámetros requeridos por la consulta. |

`codigo_query` identifica la regla. No se envía como parámetro SQL.

## Configuración de la regla

La regla identificada debe contener como mínimo:

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "nom_tag": "nombreEmpleado",
  "tipo_dato": "Q",
  "query_dato": "SELECT nombre FROM empleado WHERE nit_empresa = :nit_empresa AND codigo = :codigo",
  "ind_habilita": 1
}
```

Si `nreglas` aún no tiene `codigo_query`, puede localizarse inicialmente por `id`. Para integrar FoxPro se recomienda un código funcional estable, porque el `id` puede cambiar entre ambientes.

## Flujo de ejecución

1. Validar que `operacion` sea `"4"`.
2. Leer `codigo_query` desde `campos`.
3. Buscar una única regla habilitada con ese código.
4. Obtener `query_dato` y `tipo_dato` desde la regla.
5. Combinar `nit_empresa` con los demás valores de `campos`.
6. Retirar `codigo_query` de los parámetros.
7. Aplicar `solveParams()` para conservar solo los parámetros usados en `query_dato`.
8. Validar que no falte ningún parámetro requerido.
9. Ejecutar únicamente esa regla.
10. Transformar y devolver el resultado según `tipo_dato`.

No se ejecutan otras reglas y no se llama a `armarElementosHijoPadre()`.

## Resolución de parámetros

Entrada:

```json
{
  "operacion": "4",
  "tabla": "nreglas",
  "nit_empresa": "123456",
  "campos": {
    "codigo_query": "EMPLEADO_NOMBRE",
    "codigo": "90001",
    "docum": "1001"
  }
}
```

Consulta configurada:

```sql
SELECT nombre
FROM empleado
WHERE nit_empresa = :nit_empresa
  AND codigo = :codigo;
```

Parámetros enviados a la consulta después de aplicar `solveParams()`:

```json
{
  "nit_empresa": "123456",
  "codigo": "90001"
}
```

`codigo_query` se retira porque identifica la regla y `docum` se descarta porque la consulta no lo utiliza.

## Respuesta base

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "tipo_dato": "Q",
  "resultado": "Carlos Pérez"
}
```

El cliente no envía `tipo_dato`. El servicio lo obtiene de la regla.

## Tipo `F`: valor fijo

No ejecuta SQL. Devuelve directamente el contenido de `query_dato`.

### Entrada completa

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

### Respuesta esperada

```json
{
  "codigo_query": "AMBIENTE_ACTUAL",
  "tipo_dato": "F",
  "resultado": "PRODUCCION"
}
```

## Tipo `Q`: primer valor

Ejecuta la consulta y devuelve el primer campo de la primera fila.

### Entrada completa

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

### Respuesta esperada

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "tipo_dato": "Q",
  "resultado": "Carlos Pérez"
}
```

## Tipo `A`: arreglo de objetos

Ejecuta la consulta y devuelve todas las filas conservando el nombre de las columnas.

### Entrada completa

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

### Respuesta esperada

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

## Tipo `B`: columnas de la primera fila

Ejecuta la consulta y devuelve como arreglo los valores de todas las columnas de la primera fila.

### Entrada completa

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

### Respuesta esperada

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

## Tipo `C`: primera columna de todas las filas

Ejecuta la consulta y devuelve el primer valor de cada fila.

### Entrada completa

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

### Respuesta esperada

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

## Respuesta cuando la regla no existe

```json
{
  "codigo_query": "EMPLEADO_NOMBRE",
  "tipo_dato": null,
  "resultado": null,
  "error": "No existe una regla habilitada con el código enviado"
}
```

## Reglas de implementación

- `codigo_query` es obligatorio para la operación `"4"`.
- Solo puede encontrarse una regla habilitada por código.
- FoxPro no envía `query_dato` ni `tipo_dato`.
- FoxPro no envía SQL.
- `codigo_query` se retira antes de resolver los parámetros.
- Los valores se enlazan mediante parámetros nombrados.
- `solveParams()` elimina parámetros sobrantes.
- Deben validarse los parámetros faltantes antes de ejecutar.
- `nit_empresa` debe utilizarse en consultas que accedan a datos de una empresa.
- Se admiten únicamente los tipos `F`, `Q`, `A`, `B` y `C`.
- La operación no ejecuta el árbol completo de reglas.
- La operación no modifica información.

