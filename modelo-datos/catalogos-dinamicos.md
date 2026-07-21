# Catálogos dinámicos: tablas, permisos y configuración

Esta guía explica cómo se construye un catálogo dinámico en VSM usando **Cargos** como ejemplo. Incluye todas las tablas que participan en la autenticación, autorización, menú, configuración, datos, empresa activa y auditoría.

## 1. Resumen del flujo

Para que `Cargos` aparezca y funcione, el sistema recorre esta cadena:

```text
vmusers
   │
   ├── vmusuempresa ── vmempresa ── vmsucursal
   │
   └── vmusuperfil ── vmperfil ── vmmenu ── vmcatalogo
                                             │
                                             ├── vmcatalogo_campo
                                             └── ncargo

Las actualizaciones y eliminaciones generan registros en vmauditoria.
```

El frontend no tiene una pantalla desarrollada específicamente para `ncargo`. Lee la definición retornada por el backend y construye dinámicamente la grilla, los filtros, el formulario y sus acciones.

## 2. Inventario completo de tablas

### 2.1 Tablas obligatorias del catálogo

| Tabla | Responsabilidad |
|---|---|
| `vmusers` | Identifica al usuario autenticado, su empresa predeterminada, sucursal y condición de administrador. |
| `vmperfil` | Define los perfiles que pueden recibir opciones de menú. |
| `vmusuperfil` | Relaciona usuarios con perfiles. |
| `vmmenu` | Define la opción visible, ruta, icono, orden, padre y perfil autorizado. |
| `vmcatalogo` | Relaciona una ruta del menú con una tabla física y parametriza las acciones del catálogo. |
| `vmcatalogo_campo` | Configura cada columna: título, orden, visibilidad, búsqueda, ordenamiento, control y relaciones. |
| `ncargo` | Tabla física de negocio del catálogo Cargos. Para otro catálogo se reemplaza por su tabla correspondiente. |

### 2.2 Contexto multiempresa

| Tabla | Responsabilidad |
|---|---|
| `vmempresa` | Contiene las empresas disponibles y su información visual, incluido `imagen`. |
| `vmusuempresa` | Relaciona cada usuario con sus empresas y determina si la asignación está activa. |
| `vmsucursal` | Contiene las sucursales de una empresa. Forma parte del contexto empresarial, aunque actualmente no decide si aparece Cargos. |
| `vmusers` | Conserva `nit_empresa` y `cod_sucursal` predeterminados como respaldo. |

La empresa elegida por el usuario se envía al backend como contexto activo. Para usuarios no administradores, las consultas del catálogo se limitan normalmente mediante:

```sql
WHERE tb.nit_empresa = ?
```

### 2.3 Auditoría

| Tabla | Responsabilidad |
|---|---|
| `vmauditoria` | Registra actualizaciones y eliminaciones lógicas ejecutadas por el repositorio genérico. |

Actualmente la implementación registra `ACTUALIZAR` y `ELIMINAR`. No registra la creación de filas en esta tabla.

### 2.4 Metadata interna de MySQL

El backend consulta también:

- `INFORMATION_SCHEMA.COLUMNS`
- `INFORMATION_SCHEMA.STATISTICS`

Estas vistas no se pueblan. MySQL las mantiene y el usuario de conexión necesita poder consultar la metadata de la base activa. Se utilizan para descubrir columnas, tipos, tamaños, llaves e índices y para validar los formularios dinámicos.

## 3. Permisos y acciones

No existe una tabla independiente como `vmaccion`, `vmpermisoaccion` o `vmmenuaccion`.

El acceso a la pantalla se determina con:

```text
usuario → perfil → menú → catálogo
```

La visibilidad de cada acción se parametriza directamente en `vmcatalogo`:

| Campo | Acción |
|---|---|
| `permite_buscar` | Mostrar búsqueda contra la base de datos. |
| `permite_crear` | Mostrar y habilitar Nuevo. |
| `permite_editar` | Mostrar y habilitar Editar. |
| `permite_eliminar` | Mostrar y habilitar Eliminar. |
| `permite_exportar` | Mostrar y habilitar Exportar. |
| `icono_buscar` | Icono de Buscar. |
| `icono_crear` | Icono de Nuevo. |
| `icono_editar` | Icono de Editar. |
| `icono_eliminar` | Icono de Eliminar. |
| `icono_exportar` | Icono de Exportar. |

Un valor `0` en `permite_*` oculta o deshabilita la capacidad correspondiente; un valor `1` la habilita.

## 4. Configuración exacta de Cargos

### 4.1 Menú padre

`Cargos` es hijo del menú `Catálogos`:

```sql
SELECT *
FROM vmmenu
WHERE id = 5;
```

### 4.2 Opción de menú

```sql
INSERT INTO vmmenu (
  id, orden, nombre, ruta, icono, id_mpadre, id_perfil, ind_activo
)
VALUES (
  15, 6, 'Cargos', 'catalogos/cargos', 'jobs', 5, 1, 1
)
ON DUPLICATE KEY UPDATE
  orden = VALUES(orden),
  nombre = VALUES(nombre),
  ruta = VALUES(ruta),
  icono = VALUES(icono),
  id_mpadre = VALUES(id_mpadre),
  id_perfil = VALUES(id_perfil),
  ind_activo = VALUES(ind_activo);
```

La ruta `catalogos/cargos` debe coincidir exactamente entre el menú, las solicitudes del frontend y la consulta del backend.

### 4.3 Definición del catálogo

```sql
INSERT INTO vmcatalogo (
  codigo,
  titulo,
  tabla,
  id_menu,
  orden_campo,
  orden_direccion,
  permite_crear,
  permite_editar,
  permite_eliminar,
  permite_exportar,
  permite_buscar,
  ind_activo
)
VALUES (
  'CAT_NCARGO',
  'Cargos',
  'ncargo',
  15,
  'codca',
  'asc',
  1, 1, 1, 1, 1, 1
)
ON DUPLICATE KEY UPDATE
  titulo = VALUES(titulo),
  tabla = VALUES(tabla),
  id_menu = VALUES(id_menu),
  orden_campo = VALUES(orden_campo),
  orden_direccion = VALUES(orden_direccion),
  permite_crear = VALUES(permite_crear),
  permite_editar = VALUES(permite_editar),
  permite_eliminar = VALUES(permite_eliminar),
  permite_exportar = VALUES(permite_exportar),
  permite_buscar = VALUES(permite_buscar),
  ind_activo = VALUES(ind_activo);
```

### 4.4 Campos de la grilla y formulario

```sql
SET @catalogo_cargos_id = (
  SELECT id
  FROM vmcatalogo
  WHERE codigo = 'CAT_NCARGO'
  LIMIT 1
);

INSERT INTO vmcatalogo_campo (
  id_catalogo,
  campo,
  titulo,
  orden,
  visible,
  searchable,
  sortable,
  ancho,
  tipo_control,
  ind_activo
)
VALUES
  (@catalogo_cargos_id, 'id',          'ID',                 1, 0, 0, 1, '90px',  'number',  1),
  (@catalogo_cargos_id, 'codca',       'Código',             2, 1, 1, 1, '140px', 'input',   1),
  (@catalogo_cargos_id, 'detalle',     'Nombre del cargo',   3, 1, 1, 1, '320px', 'input',   1),
  (@catalogo_cargos_id, 'porarp',      'Porcentaje ARP',     4, 1, 0, 1, '150px', 'decimal', 1),
  (@catalogo_cargos_id, 'nit_empresa', 'NIT de la empresa', 5, 0, 0, 1, '160px', 'input',   1),
  (@catalogo_cargos_id, 'ind_estado',  'Estado',             6, 1, 0, 1, '110px', 'number',  1)
ON DUPLICATE KEY UPDATE
  titulo = VALUES(titulo),
  orden = VALUES(orden),
  visible = VALUES(visible),
  searchable = VALUES(searchable),
  sortable = VALUES(sortable),
  ancho = VALUES(ancho),
  tipo_control = VALUES(tipo_control),
  ind_activo = VALUES(ind_activo);
```

## 5. Significado de `vmcatalogo_campo`

| Campo | Significado |
|---|---|
| `campo` | Nombre exacto de la columna física. |
| `titulo` | Etiqueta presentada al usuario. |
| `orden` | Posición en grilla y formulario. |
| `visible` | Indica si aparece en la grilla. |
| `searchable` | Permite usar el campo en la búsqueda contra la base de datos. |
| `sortable` | Permite ordenar la grilla por la columna. |
| `ancho` | Ancho sugerido para la columna. |
| `tipo_control` | Control usado en el formulario: `input`, `number`, `decimal`, `select`, etc. |
| `relacion_tabla` | Tabla consultada cuando el campo es una relación. |
| `relacion_campo_valor` | Columna relacionada que se guarda. |
| `relacion_campo_etiqueta` | Columna relacionada que se muestra. |
| `relacion_filtra_empresa` | Aplica la empresa activa a las opciones relacionadas. |
| `relacion_filtra_estado` | Limita las opciones relacionadas a registros activos. |
| `buscar_al_perder_foco` | Busca un registro existente al salir del campo. |
| `relacion_permite_crear` | Muestra el botón para crear un elemento relacionado desde una ventana modal. |
| `ind_activo` | Activa o desactiva la configuración del campo. |

Para un campo relacionado se deben configurar, como mínimo, `relacion_tabla`, `relacion_campo_valor` y `relacion_campo_etiqueta`. `relacion_permite_crear` debe habilitarse únicamente en las relaciones donde se autorice crear registros nuevos.

## 6. Requisitos de una tabla de negocio

Para soportar creación, edición y eliminación mediante el backend genérico, una tabla de catálogo debe contener:

```sql
id
nit_empresa
nom_usuario
ind_estado
created_at
updated_at
```

`id` debe ser preferiblemente la llave primaria simple. Los cinco campos de sistema comprobados obligatoriamente por el backend son:

```text
nit_empresa
nom_usuario
ind_estado
created_at
updated_at
```

Ejemplo mínimo:

```sql
CREATE TABLE IF NOT EXISTS nmi_catalogo (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  codigo VARCHAR(20) NOT NULL,
  detalle VARCHAR(100) NOT NULL,
  nit_empresa VARCHAR(20) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  ind_estado TINYINT NULL DEFAULT 1,
  nom_usuario VARCHAR(100) NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_nmi_catalogo_empresa_codigo (nit_empresa, codigo),
  KEY idx_nmi_catalogo_empresa_estado (nit_empresa, ind_estado)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

## 7. Orden recomendado para registrar otro catálogo

1. Crear la tabla física de negocio.
2. Confirmar que contiene las columnas obligatorias del backend.
3. Crear índices compatibles con empresa, estado, ordenamiento y búsquedas.
4. Crear o identificar el perfil en `vmperfil`.
5. Relacionar el usuario con el perfil en `vmusuperfil`.
6. Crear la opción en `vmmenu`, vinculándola con el menú padre y el perfil.
7. Crear la definición en `vmcatalogo`.
8. Crear las definiciones de columnas en `vmcatalogo_campo`.
9. Verificar que la empresa exista en `vmempresa`.
10. Verificar la asignación del usuario en `vmusuempresa`.
11. Verificar que exista `vmauditoria` si se permitirán edición y eliminación.
12. Iniciar sesión nuevamente o esperar la expiración de caché de metadata antes de probar.

## 8. Consultas de diagnóstico

### Verificar toda la cadena de autorización

```sql
SELECT
  u.id AS usuario_id,
  u.email,
  p.id AS perfil_id,
  p.nombre AS perfil,
  m.id AS menu_id,
  m.nombre AS menu,
  m.ruta,
  c.id AS catalogo_id,
  c.codigo AS catalogo,
  c.tabla
FROM vmusers u
INNER JOIN vmusuperfil up
  ON up.id_user = u.id
 AND up.ind_activo = 1
INNER JOIN vmperfil p
  ON p.id = up.id_perfil
 AND p.ind_activo = 1
INNER JOIN vmmenu m
  ON m.id_perfil = p.id
 AND m.ind_activo = 1
INNER JOIN vmcatalogo c
  ON c.id_menu = m.id
 AND c.ind_activo = 1
WHERE u.id = 1
  AND u.ind_activo = 1
  AND m.ruta = 'catalogos/cargos';
```

### Verificar las columnas configuradas

```sql
SELECT cc.*
FROM vmcatalogo_campo cc
INNER JOIN vmcatalogo c ON c.id = cc.id_catalogo
WHERE c.codigo = 'CAT_NCARGO'
ORDER BY cc.orden, cc.id;
```

### Verificar empresa y registros visibles

```sql
SELECT e.nit, e.nombre, ue.id_user, ue.ind_activo
FROM vmempresa e
INNER JOIN vmusuempresa ue ON ue.nit_empresa = e.nit
WHERE ue.id_user = 1;

SELECT *
FROM ncargo
WHERE nit_empresa = '00000000'
ORDER BY codca, id
LIMIT 20;
```

## 9. Archivos de referencia

- `vsm_web_backend/sql/004_auth_multiempresa.sql`: usuarios, perfiles, empresas y menú base.
- `vsm_web_backend/sql/005_catalogos_ncargo.sql`: implementación completa del catálogo Cargos.
- `vsm_web_backend/sql/009_catalogos_iconos_acciones.sql`: iconos de Editar y Eliminar.
- `vsm_web_backend/sql/010_catalogos_acciones_toolbar.sql`: acciones e iconos de la barra superior.
- `vsm_web_backend/sql/011_catalogos_creacion_relacion.sql`: creación parametrizable de registros relacionados.
- `vsm_web_backend/sql/015_empresas_sucursales_usuario.sql`: empresas, sucursales y asignación por usuario.
- `vsm_web_backend/sql/manual/vsmauditoria.sql`: tabla de auditoría.

