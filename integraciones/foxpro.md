---
title: foxpro
description: 
published: true
date: 2026-07-09T18:54:50.782Z
tags: 
editor: markdown
dateCreated: 2026-07-09T18:54:45.994Z
---

# Documentación técnica: Generación de JSON desde FoxPro para sincronización de datos

## 1. Objetivo

Desde FoxPro se debe generar un mensaje en formato JSON con la información necesaria para ejecutar operaciones sobre una base de datos interna.

El JSON debe indicar:

- La tabla sobre la cual se realizará la operación.
- El tipo de operación: agregar, actualizar o eliminar.
- El `nit_empresa`, que será usado como criterio obligatorio de seguridad.
- Los campos que se desean insertar o modificar.
- Los campos llave que identifican el registro o registros afectados.

Este mecanismo permitirá que FoxPro envíe instrucciones estructuradas para que otro sistema procese internamente los cambios y genere las operaciones SQL correspondientes.

---

## 2. Estructura general del JSON

```json
{
  "tabla": "cliente",
  "operacion": "1",
  "nit_empresa": "123456",
  "campos": {
    "nombre": "valor",
    "apellido": "valor",
    "camponombre": "valor2"
  },
  "camposllave": {
    "id": "1"
  }
}
```

---

## 3. Descripción de atributos principales

### `tabla`

Nombre de la tabla sobre la cual se realizará la operación.

Ejemplo:

```json
"tabla": "cliente"
```

---

### `operacion`

Indica el tipo de operación que debe ejecutarse.

| Valor | Operación |
|---|---|
| `"1"` | Agregar / Insertar |
| `"2"` | Actualizar |
| `"3"` | Eliminar |

---

### `nit_empresa`

Identificador de la empresa.

Este campo es obligatorio y debe enviarse en todas las operaciones.

Además, el sistema interno debe usar siempre este valor como parte del filtro de seguridad en operaciones de actualización y eliminación.

Ejemplo:

```json
"nit_empresa": "123456"
```

---

### `campos`

Objeto que contiene los campos y valores que serán insertados o actualizados.

Para inserciones, aquí deben enviarse los campos que se desean guardar.

Para actualizaciones, solo deben enviarse los campos que realmente van a cambiar. No es necesario enviar todos los campos de la tabla.

Ejemplo:

```json
"campos": {
  "nombre": "Juan",
  "apellido": "Pérez",
  "telefono": "3001234567"
}
```

---

### `camposllave`

Objeto que contiene los campos que serán usados como criterio de búsqueda en el `WHERE`.

Este atributo es obligatorio para operaciones de actualización y eliminación.

Llave simple:

```json
"camposllave": {
  "id": "1"
}
```

Llave compuesta:

```json
"camposllave": {
  "codigo": "1",
  "user": "1"
}
```

El sistema interno siempre debe agregar automáticamente el filtro por `nit_empresa`.

---

# 4. Operación 1: Agregar registro

## Descripción

Cuando `operacion` sea `"1"`, el sistema debe insertar un nuevo registro en la tabla indicada.

En este caso, el objeto `campos` contiene los nombres de las columnas y los valores que se desean insertar.

## JSON de ejemplo

```json
{
  "tabla": "cliente",
  "operacion": "1",
  "nit_empresa": "123456",
  "campos": {
    "nombre": "Juan",
    "apellido": "Pérez",
    "camponombre": "valor2"
  }
}
```

## SQL equivalente

```sql
INSERT INTO cliente 
(nombre, apellido, camponombre, nit_empresa)
VALUES 
('Juan', 'Pérez', 'valor2', '123456');
```

## Regla importante

Aunque `nit_empresa` venga fuera del objeto `campos`, debe ser incluido internamente en la inserción si la tabla lo requiere.

---

# 5. Operación 2: Actualizar registro con llave simple

## Descripción

Cuando `operacion` sea `"2"`, el sistema debe actualizar uno o varios campos de la tabla indicada.

El objeto `campos` debe contener únicamente los campos que serán modificados.

El objeto `camposllave` debe contener los campos usados para identificar el registro.

## JSON de ejemplo

```json
{
  "tabla": "cliente",
  "operacion": "2",
  "nit_empresa": "123456",
  "campos": {
    "nombre": "Carlos",
    "apellido": "Ramírez"
  },
  "camposllave": {
    "id": "1"
  }
}
```

## SQL equivalente

```sql
UPDATE cliente
SET 
  nombre = 'Carlos',
  apellido = 'Ramírez'
WHERE 
  id = '1'
  AND nit_empresa = '123456';
```

---

# 6. Operación 2: Actualizar registro con llave compuesta

## Descripción

En algunos casos, la actualización no se realiza con un único campo como `id`, sino con una combinación de varios campos.

Esto se conoce como llave compuesta.

## JSON de ejemplo

```json
{
  "tabla": "cliente",
  "operacion": "2",
  "nit_empresa": "123456",
  "campos": {
    "nombre": "Andrés",
    "camponombre": "valor_actualizado"
  },
  "camposllave": {
    "codigo": "1",
    "user": "1"
  }
}
```

## SQL equivalente

```sql
UPDATE cliente
SET 
  nombre = 'Andrés',
  camponombre = 'valor_actualizado'
WHERE 
  codigo = '1'
  AND user = '1'
  AND nit_empresa = '123456';
```

---

# 7. Operación 3: Eliminar registro

## Descripción

Cuando `operacion` sea `"3"`, el sistema debe eliminar el registro o registros que coincidan con los criterios enviados en `camposllave`.

Para eliminar, no es necesario enviar el objeto `campos`.

## JSON de ejemplo con llave simple

```json
{
  "tabla": "cliente",
  "operacion": "3",
  "nit_empresa": "123456",
  "camposllave": {
    "id": "1"
  }
}
```

## SQL equivalente

```sql
DELETE FROM cliente
WHERE 
  id = '1'
  AND nit_empresa = '123456';
```

---

## JSON de ejemplo con llave compuesta

```json
{
  "tabla": "cliente",
  "operacion": "3",
  "nit_empresa": "123456",
  "camposllave": {
    "codigo": "1",
    "user": "1"
  }
}
```

## SQL equivalente

```sql
DELETE FROM cliente
WHERE 
  codigo = '1'
  AND user = '1'
  AND nit_empresa = '123456';
```

---

# 8. Reglas generales para la implementación

## 8.1. El campo `nit_empresa` es obligatorio

Todas las operaciones deben enviar el campo `nit_empresa`.

Este campo debe usarse como filtro de seguridad para evitar que una operación afecte datos de otra empresa.

---

## 8.2. Para actualizar, no se deben enviar todos los campos

En operaciones de actualización, el objeto `campos` debe contener solo los campos que van a cambiar.

Ejemplo correcto:

```json
{
  "campos": {
    "telefono": "3001234567"
  }
}
```

---

## 8.3. Para eliminar, no se debe enviar `campos`

En operaciones de eliminación, solo se requiere:

- `tabla`
- `operacion`
- `nit_empresa`
- `camposllave`

Ejemplo:

```json
{
  "tabla": "cliente",
  "operacion": "3",
  "nit_empresa": "123456",
  "camposllave": {
    "id": "1"
  }
}
```

---

## 8.4. `camposllave` es obligatorio para actualizar y eliminar

Toda operación de actualización o eliminación debe tener criterios claros en `camposllave`.

No se debe permitir una actualización o eliminación sin `camposllave`.

Ejemplo incorrecto:

```json
{
  "tabla": "cliente",
  "operacion": "2",
  "nit_empresa": "123456",
  "campos": {
    "nombre": "Carlos"
  }
}
```

Esto podría generar una actualización masiva no deseada.

---

## 8.5. El sistema debe validar las tablas permitidas

Por seguridad, el sistema interno no debería ejecutar operaciones sobre cualquier nombre de tabla recibido.

Se recomienda tener una lista blanca de tablas permitidas.

Ejemplo:

```text
cliente
producto
factura
usuario
```

---

## 8.6. El sistema debe validar los campos permitidos

También se recomienda validar que los campos enviados en `campos` y `camposllave` existan realmente en la tabla correspondiente.

Esto evita errores o posibles instrucciones mal formadas.

---

## 8.7. El SQL debe generarse usando parámetros

Aunque en esta documentación se muestran ejemplos SQL con valores directamente en el texto, la implementación real debe usar consultas parametrizadas.

No se recomienda construir SQL concatenando texto directamente, porque puede generar riesgos de seguridad.

Ejemplo recomendado:

```sql
UPDATE cliente
SET nombre = ?
WHERE id = ?
AND nit_empresa = ?
```

---

# 9. Resumen de obligatoriedad por operación

| Campo | Agregar | Actualizar | Eliminar |
|---|---:|---:|---:|
| `tabla` | Obligatorio | Obligatorio | Obligatorio |
| `operacion` | Obligatorio | Obligatorio | Obligatorio |
| `nit_empresa` | Obligatorio | Obligatorio | Obligatorio |
| `campos` | Obligatorio | Obligatorio | No requerido |
| `camposllave` | No requerido | Obligatorio | Obligatorio |

---

# 10. Consideraciones finales

El JSON generado desde FoxPro debe ser lo más limpio y específico posible.

Cada operación debe representar una única instrucción lógica:

- Si se va a crear un registro, usar `operacion = "1"`.
- Si se va a modificar información, usar `operacion = "2"`.
- Si se va a eliminar información, usar `operacion = "3"`.

La implementación interna debe encargarse de convertir esta estructura JSON en la operación SQL correspondiente, aplicando siempre las reglas de seguridad, especialmente el uso obligatorio de `nit_empresa` en los filtros.
