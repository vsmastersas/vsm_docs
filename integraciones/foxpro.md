---
title: Integración FoxPro
description: Contrato JSON para migrar datos desde FoxPro
published: true
date: 2026-07-10T00:00:00.000Z
tags: foxpro, migracion, json
editor: markdown
dateCreated: 2026-07-09T18:54:45.994Z
---

# Integración FoxPro

## Objetivo

Migrar datos desde FoxPro mediante mensajes JSON simples para insertar, actualizar o eliminar uno o varios registros.

La estructura conserva los campos base definidos originalmente:

```json
{
  "operacion": "1",
  "tabla": "ncargo",
  "nit_empresa": "123456",
  "campos": {}
}
```

## Tipo de operación

Los comandos disponibles para probar cada escenario están centralizados en [Ejemplos cURL de la integración FoxPro](./foxpro/curls.md).

| Valor | Operación | Documentación |
|---|---|---|
| `"1"` | Insertar | [01 - Insertar](./foxpro/01-insertar.md) |
| `"2"` | Actualizar | [02 - Actualizar](./foxpro/02-actualizar.md) |
| `"3"` | Eliminar | [03 - Eliminar](./foxpro/03-eliminar.md) |
| `"4"` | Ejecutar una regla de consulta puntual | [04 - Query puntual](./foxpro/04-regla-query-puntual.md) |
| `"5"` | Ejecutar el modelo completo de reglas | [05 - Modelo completo](./foxpro/05-modelo-completo-reglas.md) |

Las operaciones `"4"` y `"5"` son únicamente de consulta y no modifican información.

## Campos base

| Campo | Descripción |
|---|---|
| `operacion` | `"1"`, `"2"`, `"3"`, `"4"` o `"5"`. |
| `tabla` | Tabla sobre la cual se ejecutará la operación. |
| `nit_empresa` | Empresa a la cual pertenecen los registros. Es obligatorio. |
| `campos` | Datos de insert/update o parámetros de consulta en las operaciones `"4"` y `"5"`. |
| `camposllave` | Criterio utilizado para actualizar o eliminar. |

## Reglas generales

- Cada mensaje contiene una sola operación y una sola tabla.
- `nit_empresa` siempre se agrega internamente al insert o al criterio de update/delete.
- Para un solo registro, `campos` es un objeto.
- Para varios registros, `campos` es un arreglo.
- Las consultas deben ejecutarse con parámetros; no se concatenan valores en SQL.
- Las tablas y campos recibidos deben validarse contra una lista permitida.
- No se permite actualizar o eliminar sin `camposllave`.
