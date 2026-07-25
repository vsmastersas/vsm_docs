# Modelo IARK parametrizado en JSON

Esta implementación traslada a VSM el motor de formularios, secciones, grillas,
eventos y acciones usado por `vs_appt_frontend`, `vst_appt_backend` y
`saequi_VS_TEMPLATE_APP.sql`. Los catálogos existentes continúan trabajando con
`vmcatalogo` y `vmcatalogo_campo`; los procesos transaccionales nuevos se
renderizan a partir de un documento JSON compilado.

## Resultado

- Los trece procesos activos encontrados en el modelo anterior: Cotización,
  Proyectos, Gastos, Selección de productos, Selección APU, Guardar accesorios,
  Adjuntos de cotización, Consulta de adjuntos, Consulta de cotizaciones,
  Agenda, Visitas, Registrar visita y Seguimiento.
- La parametrización completa del dump: 27 secciones, 144 controles (incluye
  los 4 controles hijos condicionales), 95
  eventos, 68 fuentes de datos, 50 enlaces de guardado y 4 acciones de fila
  distribuidas en los trece procesos.
- Cotización conserva sus seis secciones reales: Tipo documento, Cliente,
  Método de pago, Detalle, Productos y Consulta. Incluye sus grillas internas,
  columnas editables, selección de filas, adjuntos y controles de apertura.
- Parametrización administrable desde cuatro catálogos dinámicos.
- Aislamiento obligatorio por la empresa seleccionada en el token.
- Altas, edición, borrado lógico y auditoría reutilizando el repositorio seguro
  existente.
- Un único refresh compila tablas y procesos en `storage/schema.json`.

## Equivalencia con el modelo anterior

| Modelo anterior | Modelo VSM |
|---|---|
| `ct_procesos` | `iak_proceso` y `configuracion_json` |
| `ct_secciones` | `config.form.sections[]` |
| `ct_formulario` | `config.form` |
| `ct_detalle_campo_form` | `config.form.fields[]` |
| `ct_evento` | `fields[].events[]` |
| `cnf_tipo_control` | `iak_tipo_control` |
| `cnf_tipo_evento` | `iak_tipo_evento` |
| `cnf_tipo_accion` | `iak_tipo_accion` |
| `cnf_query` | `config.dataSources[]` y `grids[].source` |
| `cnf_query_accion_reg` | `grids[].actions[]` |
| `vst_menu` | `vmmenu` |

No se copian JavaScript ni SQL arbitrarios desde la base de datos. Es una mejora
intencional: el JSON declara tablas, campos, filtros, eventos y acciones; el
backend valida identificadores contra el esquema compilado y usa parámetros
MySQL para todos los valores.

## Tablas

| Tabla | Uso |
|---|---|
| `iak_proceso` | Documento JSON, ruta, tabla destino, versión y empresa. |
| `iak_tipo_control` | Catálogo de controles admitidos. |
| `iak_tipo_evento` | Catálogo de disparadores admitidos. |
| `iak_tipo_accion` | Catálogo de acciones admitidas. |
| `iak_categoria` | Catálogo relacionado usado para demostrar búsqueda y creación rápida parametrizable. |
| `iak_registro_demo` | Adaptador CRUD de referencia de la primera versión. Se mantiene por compatibilidad. |
| `iak_proceso_registro` | Persistencia multiempresa del formulario completo en `data_json`; evita perder campos mientras se adaptan las tablas transaccionales legacy. |
| `iak_beneficiario` | Clientes/proveedores creados desde la modal parametrizada de Cotización. |
| `iak_contacto` | Contactos creados desde la modal parametrizada de Cotización. |
| `iak_crm_cliente` | Clientes de CRM creados desde Agenda. |
| `iak_crm_contacto` | Contactos de CRM creados desde Agenda. |
| `vmmenu` | Menús de administración y ejecución. |
| `vmcatalogo` | Catálogos para administrar la parametrización. |
| `vmcatalogo_campo` | Campos de esos catálogos. `visible_formulario` permite ocultar JSON en la grilla sin ocultarlo al editar. |
| `vmauditoria` | Cambios y eliminaciones lógicas. La migración la crea si falta. |

Las tablas transaccionales nuevas usan el prefijo `iak_`. Las tablas compartidas
se adaptan sin duplicarlas.

## Documento JSON

Ejemplo reducido:

```json
{
  "schemaVersion": 2,
  "presentation": { "type": "accordion", "columns": 12 },
  "storage": { "mode": "json", "table": "iak_proceso_registro", "column": "data_json" },
  "form": {
    "primaryKey": "id",
    "submitLabel": "Guardar",
    "fields": [
      { "name": "codigo", "label": "Código", "control": "text", "required": true },
      {
        "name": "nombre",
        "label": "Nombre",
        "control": "text",
        "events": [{ "trigger": "blur", "type": "transform", "transform": "trim" }]
      }
    ],
    "sections": [
      { "id": "datos", "title": "Información general", "fields": ["codigo", "nombre"] }
    ]
  },
  "grids": [{
    "id": "registros",
    "title": "Registros del proceso",
    "pageSize": 20,
    "source": {
      "type": "table",
      "table": "iak_registro_demo",
      "fields": ["id", "codigo", "nombre", "ind_estado"],
      "orderBy": [{ "field": "id", "direction": "desc" }]
    },
    "columns": [
      { "field": "codigo", "label": "Código" },
      { "field": "nombre", "label": "Nombre" }
    ],
    "actions": [
      { "id": "edit", "label": "Editar", "type": "load_form" },
      { "id": "delete", "label": "Eliminar", "type": "soft_delete", "confirm": "¿Desea eliminar?" }
    ]
  }]
}
```

Controles implementados: `text`, `number`, `decimal`, `date`, `datetime`,
`time`, `email`, `select`, `multiselect`, `radio`, `checkbox`, `textarea`,
`json`, `richtext`, `file`, `html`, `map`, `address`, `data_grid`, `button`,
`open_catalog`, `open_process`, `open_process_modal`, `save_action`,
`query_action` y `location_action`. La distribución usa las doce columnas de
`dist` del modelo original y se repliega sin desbordarse en móvil.

Las grillas internas conservan `editableFields`, `numericFields`, columna de
selección, columna de adjunto y alineación. Las filas editadas y seleccionadas
se incluyen en el JSON guardado. La fórmula legacy de la grilla de Cotización
fue convertida a un AST declarativo (`multiply` + `set_grid_cell`), por lo que
ya no requiere `eval`.

Eventos ejecutables iniciales: `trim`, `uppercase` y `lowercase`, disparados en
`change` o `blur`. Acciones de grilla: `load_form`, `soft_delete`, `refresh` y
`navigate`.

### Relaciones buscables y creación rápida

Un campo se convierte en búsqueda relacionada mediante `searchable` y
`relation`. El botón `+` depende exclusivamente de `allowCreate`; por lo tanto,
una relación puede ser buscable sin permitir altas rápidas.

```json
{
  "name": "categoria",
  "label": "Categoría",
  "control": "select",
  "searchable": true,
  "relation": {
    "table": "iak_categoria",
    "valueField": "codigo",
    "labelField": "nombre",
    "filterCompany": true,
    "filterState": true,
    "allowCreate": true
  }
}
```

Al crear, el backend lee el esquema real de la tabla relacionada y devuelve los
campos de la modal. Si el código ya existe se recupera sin permitir modificarlo;
`Guardar y seleccionar` devuelve la opción al formulario principal. En los
catálogos tradicionales, como Personas/Área, el equivalente sigue siendo
`vmcatalogo_campo.relacion_permite_crear`.

Los controles IARK `abrirCatalogoTabla` también se mantienen como modales. El
compilador enlaza explícitamente el botón y el selector, sin depender de su
posición en pantalla:

```json
{
  "name": "cmd_cliente_agregar",
  "control": "open_catalog",
  "modalMode": "create_relation",
  "createTarget": "txtnit",
  "openCatalog": "ct_beneficiarios"
}
```

El campo indicado por `createTarget` contiene `relation.createFields`, que
parametriza orden, título, obligatoriedad, tipo de control y opciones de cada
campo de la modal. Esto permite reproducir selectores como Ciudad, Tipo de
beneficiario y Cliente sin escribir esos campos directamente en React. Al
guardar, se cierra sólo la modal, se conserva abierto el proceso y la opción
nueva queda seleccionada. El mismo flujo recupera códigos existentes en modo
de sólo lectura y cambia el botón a `Seleccionar`.

Cada selector de la modal puede habilitar búsqueda local con
`"searchable": true` dentro de `relation.createFields`. La búsqueda ignora
mayúsculas y minúsculas y compara tanto la etiqueta como el valor. Si se omite
o se configura en `false`, se mantiene el `select` convencional.

Cuando el catálogo relacionado es grande, el campo de creación puede declarar
su propia `relation`. En ese caso el selector consulta el backend con debounce,
trae máximo 25 coincidencias y no precarga el catálogo completo:

```json
{
  "field": "cargo",
  "title": "Cargo",
  "controlType": "select",
  "searchable": true,
  "relation": {
    "table": "ncargo",
    "valueField": "codca",
    "labelField": "detalle",
    "filterCompany": true,
    "filterState": true,
    "searchMode": "fulltext_boolean",
    "storeMode": "label"
  }
}
```

Los modos disponibles son `contains`, `starts_with` y `fulltext_boolean`. Este
último requiere un índice `FULLTEXT` sobre el campo de etiqueta y es el modo
usado por Cargo para mantener una respuesta rápida con millones de registros.
`storeMode` define si se persiste el `value` técnico o la `label` visible. Cargo
usa `label` porque la columna histórica `iak_contacto.cargo` almacena el nombre,
no un código foráneo.

Los selectores encadenados pueden declarar `disabledWhenEmpty` y los eventos de
cambio pueden incluir filtros sobre los datos compilados. Por ejemplo, en
Selección APU el ítem queda deshabilitado hasta elegir el material y luego sólo
muestra productos de ese tipo:

```json
{
  "name": "combo_item_principal",
  "control": "select",
  "disabledWhenEmpty": "combo_tipo_material"
}
```

La metadata auxiliar usada para filtrar se publica con nombres iniciados en
`_`; participa en las reglas declarativas, pero nunca aparece como columna en
las grillas.

Las fórmulas legacy reconocidas se expresan como `event.calculation`. El motor
admite operaciones declarativas como `multiply` y `remaining_percentage`, y
recalcula sus dependencias en cadena. Así, Selección APU actualiza área
necesaria, porcentaje de desperdicio y total sin publicar ni evaluar
JavaScript almacenado en la base original.

Las grillas de resumen pueden combinar las filas compiladas con los valores
vigentes del formulario mediante `valueBindings`. El origen puede ser fijo
(`sourceField`) o estar indicado en cada fila (`sourceFieldFromRow`), y
`rowFormulas` calcula sus columnas derivadas. El Resumen APU usa estas reglas
para completar área, valor unitario y valor total al presionar **Agregar**.

Las modales también pueden heredar datos del formulario que las abre mediante
`relation.createContext`. Por ejemplo, Contacto recibe el cliente previamente
seleccionado y no permite cambiarlo:

```json
{
  "createContext": [
    {
      "sourceField": "txtnit",
      "targetField": "nit",
      "readonly": true
    }
  ]
}
```

Los eventos legacy `obtenerUbicacion` se compilan como controles
`location_action`. Al ejecutarlos, el frontend solicita autorización al
navegador, captura latitud, longitud, precisión y fecha, y entrega el resultado
al control `map` asociado por un evento `render`. Cuando la acción está marcada
como requerida, el formulario no permite guardar sin una ubicación válida.
En procesos con almacenamiento JSON las coordenadas quedan persistidas junto
con los demás valores del registro.

El motor copia el valor y la etiqueta visibles, agrega la opción al selector de
la modal si todavía no estaba en su listado y respeta `readonly`. El mismo
mecanismo se usa en Agenda para pasar `txtnitvisita` a `cod_convenio`; puede
reutilizarse con otros campos sin modificar el frontend.

## Validación dinámica de campos requeridos

Los guardados del formulario principal y de las modales de creación rápida
leen `required` directamente de cada campo parametrizado:

```json
{
  "name": "txtvigencia",
  "label": "Vigencia (Días)",
  "control": "number",
  "required": true
}
```

Si falta información, el frontend no envía la operación: conserva los valores
capturados y abre una modal que enumera todas las etiquetas requeridas
pendientes. Los campos condicionados por `visibleWhen` sólo se validan cuando
están visibles. La misma regla admite textos, relaciones, selectores, archivos,
listas, grillas y casillas sin codificar nombres de campos en React.

## JSON compilado y refresh

`POST /api/schema/refresh` ahora genera atómicamente la versión 2:

```json
{
  "version": 2,
  "generadoEn": "2026-07-21T23:59:23+00:00",
  "tablas": {},
  "parametrizacion": {
    "schemaVersion": 1,
    "procesos": [],
    "tipos": {}
  }
}
```

El cargador conserva compatibilidad de lectura con el archivo versión 1. Se
debe ejecutar el refresh después de modificar `configuracion_json`, crear una
tabla o alterar sus columnas.

## Compilación reproducible desde IARK

El artefacto íntegro está en
`vsm_web_backend/artifacts/iark_parameterization.json`. Se regenera contra una
copia de solo lectura de la base antigua con:

```sh
python scripts/compile_iark_legacy.py \
  --host HOST --port 3306 --database saequi_VS_TEMPLATE_APP \
  --user USUARIO_LECTURA --password CLAVE \
  --output artifacts/iark_parameterization.json
python scripts/generate_iark_migration.py
```

El primer archivo mantiene el SQL original únicamente para auditoría de la
migración. El generador lo retira del documento ejecutable antes de producir
`018_modelo_iark_completo.sql`. Ningún SQL ni JavaScript legacy se devuelve al
navegador.

## API

Todos los endpoints de ejecución exigen JWT y validan que la ruta exista en el
menú del perfil:

| Endpoint | Función |
|---|---|
| `POST /api/modelo/definicion` | Devuelve el proceso compilado de la ruta. |
| `POST /api/modelo/grilla` | Consulta paginada y filtrada de una grilla declarada. |
| `POST /api/modelo/guardar` | Inserta o actualiza sólo los campos declarados. |
| `POST /api/modelo/accion` | Ejecuta una acción declarativa permitida. |
| `POST /api/modelo/opciones-relacion` | Busca opciones por código o etiqueta. |
| `POST /api/modelo/esquema-creacion-relacion` | Construye la modal desde el esquema relacionado. |
| `POST /api/modelo/crear-relacion` | Crea y devuelve la opción seleccionable. |
| `POST /api/modelo/lookup-creacion-relacion` | Recupera un código existente sin editarlo. |
| `POST /api/schema/refresh` | Recompila tablas y parametrización en el mismo JSON. |

El NIT enviado por la semilla nunca decide el acceso a datos. En cada solicitud
se sustituye por la empresa activa firmada en el token.

## Instalación en producción

1. Respaldar la base y `storage/schema.json`.
2. Ejecutar primero la migración con conexión UTF-8 (el backend anterior ignora
   de forma segura las columnas y tablas nuevas):

   ```sh
   mysql --default-character-set=utf8mb4 -u USUARIO -p BASE \
     < vsm_web_backend/sql/017_modelo_iark_json.sql
   ```

3. Ejecutar la compilación completa:

   ```sh
   mysql --default-character-set=utf8mb4 -u USUARIO -p BASE \
     < vsm_web_backend/sql/018_modelo_iark_completo.sql
   ```

4. Desplegar backend y frontend de la misma versión.
5. Ejecutar `POST /api/schema/refresh`.
6. Verificar que la respuesta reporte `version_json: 2` y
   `procesos_actualizados: 13`.
7. Cerrar sesión e ingresar nuevamente para recargar el menú.
8. Abrir un proceso, crear un registro, editarlo y eliminarlo; comprobar la
   trazabilidad en `vmauditoria`.

La migración es idempotente: usa `CREATE TABLE IF NOT EXISTS`, claves únicas y
`ON DUPLICATE KEY UPDATE`. No elimina tablas ni datos existentes.

## Cómo adaptar una tabla real

La tabla destino debe tener llave primaria simple y las columnas estándar
`nit_empresa`, `nom_usuario`, `ind_estado`, `created_at` y `updated_at`.
Después:

1. Cambiar `tabla_destino` en `iak_proceso`.
2. Ajustar `form.fields`, `grids[].source.fields`, columnas, orden y filtros
   fijos dentro de `configuracion_json`.
3. Mantener únicamente controles, eventos y acciones admitidos.
4. Ejecutar el refresh.

No se requiere crear un componente React ni un endpoint por proceso.
# Paridad funcional con IARK

La réplica no ejecuta el JavaScript ni el SQL almacenado por la aplicación
anterior. El compilador abre la base IARK en una transacción de solo lectura,
convierte los comportamientos reconocidos a JSON declarativo y la aplicación
interpreta únicamente operaciones permitidas.

La inspección de `saequi_VS_TEMPLATE_APP` fue exclusivamente de lectura. Los
usuarios `djkav2009@gmail.com` y `hezuri2015@gmail.com` están activos y ambos
usan el perfil `SUPER ADMIN APP`; por tanto, la diferencia de comportamiento
no proviene de permisos distintos entre esos dos usuarios.

El inventario fuente revisado incluye los controles de texto, número, decimal,
fecha, fecha/hora, hora, email, select, multiselect, radio, editor enriquecido,
botones, apertura de catálogo/proceso/modal, grillas, guardar, consultar,
archivo, mapa, HTML, direcciones y geolocalización. Los eventos se conservan
como inicio, cambio, validación, renderizado y ubicación.

## Consulta parametrizada

Los controles `searchFiltersQuery` se compilan como `query_action`. El origen,
el destino y la fuente se enlazan mediante `events`; los criterios quedan en
`dataSources[].filters`.

```json
{
  "queryMode": "filter_initial_data",
  "filters": [
    {
      "sourceField": "numero_cotiza",
      "resultField": "Número Cotización",
      "operator": "eq"
    },
    {
      "sourceField": "fecha_ini_cotiza",
      "resultField": "Fecha Cotización",
      "operator": "date_gte"
    }
  ]
}
```

Los operadores admitidos son `eq`, `contains`, `date_gte` y `date_lte`. El
modo `option_label` permite comparar la etiqueta visible de un combo en lugar
de su identificador.

## Cargue de archivos en catálogos

La tabla `iak_catalogo_cargue` habilita el botón **Importar** y conserva toda
la definición en `configuracion_json`: formato, encabezado, máximo de filas,
columnas por posición, obligatoriedad, llave y tratamiento de conflictos.
La migración
`vsm_web_backend/sql/019_paridad_iark_consultas_cargues.sql` parametriza el
catálogo Cargos. El archivo
`vsm_docs/ejemplos/cargos-importacion.csv` permite probarlo.

La parametrización fuente revisada también contiene los cargues de
beneficiarios (`nit`, `razon_social`, `direccion`), formas de pago
(`fp_cod`, `fp_nom`, `mp_cod`) y niveles ARP (`cod_nivarp`, `nom_nivarp`,
`por_nivarp`). El motor nuevo no quema esos nombres: cada catálogo se habilita
agregando su documento JSON a `iak_catalogo_cargue`.

El backend valida extensión, tamaño, número de filas, columnas requeridas e
identificadores permitidos. Las escrituras usan parámetros y una transacción
atómica; una fila inválida revierte todo el archivo.

## Validaciones de formularios

- Todo campo requerido muestra un asterisco rojo proveniente de `required`.
- Guardar con campos incompletos abre un modal con la lista completa y conserva
  los valores digitados.
- Los archivos respetan `accept` y `maxFileSizeMb`; cualquier incumplimiento se
  informa en modal.
- La misma validación funciona en el formulario principal y en modales de
  creación rápida.

## Datos reproducibles

En **Procesos IARK > Consulta Cotizaciones**:

- Número `2024002` retorna la cotización de KOBA COLOMBIA SAS.
- NIT `695` retorna cotizaciones de DUNA ARQUITECTURA E ILUMINACION.
- Rango `2024-12-13` a `2024-12-13` retorna los registros de esa fecha.
- Usuario `CESAR AUGUSTO VARGAS BENAVIDES` retorna siete registros del conjunto
  compilado.

En **Catálogos > Cargos**, use el archivo de ejemplo. El primer cargue inserta
`IARK-001` a `IARK-003`; repetirlo actualiza por `codca` sin duplicarlos.

## Inicio y gráficas parametrizadas

La tabla `iak_dashboard` guarda el tablero completo por ruta y empresa en
`configuracion_json`. La migración
`vsm_web_backend/sql/020_dashboard_iark_json.sql` instala el ejemplo de IARK
en la ruta `inicio` sin modificar los procesos ni los catálogos existentes.

El documento admite tarjetas `metric` y gráficas `bar`, `grouped_bar` y `pie`.
Cada widget parametriza ancho sobre doce columnas, título, subtítulo, datos,
series, colores, formato numérico, leyenda, texto auxiliar y ruta de
navegación. El frontend no contiene nombres de indicadores ni valores fijos:
consulta `POST /api/dashboard/definicion` y pinta el JSON autorizado para la
empresa seleccionada.

```json
{
  "type": "bar",
  "id": "barras-cotizaciones",
  "title": "Cotizaciones",
  "width": 6,
  "xKey": "periodo",
  "valueFormat": "currency",
  "series": [
    { "key": "valor", "label": "Valor", "color": "#1769d2" }
  ],
  "data": [
    { "periodo": "nov 2024", "valor": 38153847.58 }
  ]
}
```

Si existe una fila activa con el mismo código y `nit_empresa`, esta reemplaza
la definición global; de lo contrario se usa la fila con empresa nula. El
repositorio valida tipos, tamaños y límites antes de entregar el documento.
