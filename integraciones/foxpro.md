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

| Valor | Operación | Documentación |
|---|---|---|
| `"1"` | Insertar | [01 - Insertar](./foxpro/01-insertar.md) |
| `"2"` | Actualizar | [02 - Actualizar](./foxpro/02-actualizar.md) |
| `"3"` | Eliminar | [03 - Eliminar](./foxpro/03-eliminar.md) |

La lógica reutilizable de consultas y parámetros está documentada en [04 - Reglas de query](./foxpro/04-reglas-query.md).

## Campos base

| Campo | Descripción |
|---|---|
| `operacion` | `"1"`, `"2"` o `"3"`. |
| `tabla` | Tabla sobre la cual se ejecutará la operación. |
| `nit_empresa` | Empresa a la cual pertenecen los registros. Es obligatorio. |
| `campos` | Datos que se insertan o actualizan. En operaciones múltiples es un arreglo. |
| `camposllave` | Criterio utilizado para actualizar o eliminar. |

## Reglas generales

- Cada mensaje contiene una sola operación y una sola tabla.
- `nit_empresa` siempre se agrega internamente al insert o al criterio de update/delete.
- Para un solo registro, `campos` es un objeto.
- Para varios registros, `campos` es un arreglo.
- Las consultas deben ejecutarse con parámetros; no se concatenan valores en SQL.
- Las tablas y campos recibidos deben validarse contra una lista permitida.
- No se permite actualizar o eliminar sin `camposllave`.
