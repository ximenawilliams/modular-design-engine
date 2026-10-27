Aquí va la HU-03 en el mismo formato que la HU-02. Separé en dos pasos lo que hace el empleado: primero **buscar** la empresa en COBIS y después **vincularla**.

---

## 🟢 HU-03 · Vincular empresa al grupo económico
**Microservicio:** `ms-grupos-economicos`
**Endpoints:** `GET /v1/empresas-core?identificacion={ruc}` (buscar) · `POST /v1/grupos-economicos/{id}/empresas` (vincular)

**Como** empleado de Servicio de Canales, **quiero** buscar una empresa en COBIS y asociarla a un grupo económico, **para** que después se vinculen sus productos y se invite a sus usuarios.

### Componentes
| Componente | Qué hace en esta historia |
|---|---|
| **BackOffice web** (pantalla del banco) | Busca por RUC o cédula, muestra lo que devuelve COBIS y el empleado confirma |
| **API Gateway** | Valida el token del empleado y enruta al microservicio |
| **ms-grupos-economicos** | Consulta COBIS, valida las reglas, guarda la empresa y registra el evento |
| **Integración con COBIS** (servicio del core, o un mock al inicio) | Devuelve el id, el nombre y el tipo de persona de la empresa. **Solo lectura** |
| **Base BELC (Aurora MySQL)** | Tablas `Grupo_economico`, `Empresas_asociadas`, `BITACORA_EVENTO` y `TIPO_EVENTO` |
| **Librería de bitácora** (HU-13) | Registra el evento en la misma transacción |
| ForgeRock | **No interviene** |

### Modelo de datos
**`Grupo_economico`** (solo lectura): se verifica que el `id_grupo_economico` de la URL exista.

**`Empresas_asociadas`** (inserta una fila)
| Campo | De dónde viene |
|---|---|
| `id_empresa_asociada` | Lo genera la base (PK) |
| `id_grupo_economico` | De la URL (FK → `Grupo_economico`) |
| `Nombre` | De COBIS (no lo escribe el empleado) |
| `Tipo_persona` | De COBIS: `PJ` (jurídica) o `PN` (natural) |
| `core_empresasoc_id` | El id de la empresa en COBIS |

**`BITACORA_EVENTO`** (inserta una fila)
| Campo | Valor |
|---|---|
| `id_tipo_evento` | → `TIPO_EVENTO` con código `EMPRESA_VINCULADA` y categoría `BACKOFFICE` |
| `Resultado` / `Canal` | `Exito` o `Fallo` / `BO` |
| `id_grupo_economico` | El grupo |
| `Usuario_backoffice` | El empleado |
| `Entidad`, `id_registro` | `Empresas_asociadas` y su id |
| `Valor_nuevo` | El JSON con nombre, tipo de persona y `core_empresasoc_id` |
| `Codigo_error` | Se llena si COBIS falló |
| `id_correlacion`, `Fecha_evento_utc` | El `correlationId` y la fecha en UTC |

### Contrato
**1. Buscar en COBIS:** `GET /v1/empresas-core?identificacion=155612345-2-2019`
```json
{ "coreEmpresaId": "COBIS-EMP-66001", "nombre": "Constructora Pacífico, S.A.", "tipoPersona": "PJ", "yaVinculada": false }
```

**2. Vincular:** `POST /v1/grupos-economicos/20/empresas`
```json
{ "coreEmpresaId": "COBIS-EMP-66001" }
```
**Respuesta 201**
```json
{ "idEmpresaAsociada": 201, "idGrupoEconomico": 20, "nombre": "Constructora Pacífico, S.A.", "tipoPersona": "PJ", "coreEmpresaId": "COBIS-EMP-66001" }
```
**Errores:** `400` datos inválidos · `403` sin permiso · `404` el grupo no existe, o la empresa no existe en COBIS · `409` la empresa ya está vinculada a un grupo · `502` COBIS respondió con error · `504` COBIS no respondió a tiempo.

### Flujo
1. El empleado busca la empresa por su identificación, y el microservicio consulta COBIS.
2. La pantalla muestra el nombre y el tipo de persona. Si la empresa ya está vinculada, lo indica.
3. El empleado confirma, y el microservicio **vuelve a consultar COBIS** con el `coreEmpresaId`. No se confía en lo que manda la pantalla.
4. Valida que el grupo exista y que la empresa no esté en otro grupo.
5. En **una sola transacción**: inserta en `Empresas_asociadas` y en `BITACORA_EVENTO`.
6. Responde 201.

### Criterios de aceptación
1. Guarda la empresa con su `core_empresasoc_id`, y su nombre y tipo de persona tal como vienen de COBIS.
2. Una empresa no puede estar en dos grupos: si ya está vinculada, responde 409.
3. Si COBIS no responde, devuelve 504 y no guarda nada. El intento queda en la bitácora con resultado `Fallo`.
4. Si el grupo no existe, responde 404.
5. BELC solo **lee** de COBIS; no crea ni modifica nada en el core.
6. Mientras el servicio real no esté disponible, funciona con un mock de COBIS que devuelve el mismo formato.

### Pendientes en tu modelo (revisados en las fotos)
- **Falta un UNIQUE en `Empresas_asociadas.core_empresasoc_id`.** Sin él, el criterio 2 depende solo del código.
- **`Empresas_asociadas` no tiene `Estado`, `Fecha_creacion` ni `Creado_por`.** Tus otras tablas sí los tienen. Sin `Estado` no hay forma de desvincular una empresa sin borrarla.
- **Agregar un CHECK en `Tipo_persona`** con los valores `PJ` y `PN`.
- **Agregar `EMPRESA_VINCULADA` a `TIPO_EVENTO`** en tus inserts.

Esta va con el formato recortado que acordamos: sin la sección de flujo, la bitácora referida a la HU-13 y los pendientes al final, como tareas aparte.

---

## 🟢 HU-04 · Vincular productos de la empresa
**Microservicio:** `ms-grupos-economicos`
**Endpoints:** `GET /v1/grupos-economicos/{id}/empresas/{idEmpresa}/productos-core` (listar desde COBIS) · `POST /v1/grupos-economicos/{id}/empresas/{idEmpresa}/productos` (vincular)

**Como** empleado de Servicio de Canales, **quiero** ver los productos que la empresa tiene en COBIS y elegir cuáles entran a BELC, **para** que después se puedan agrupar y asignar a roles.

### Componentes
| Componente | Qué hace en esta historia |
|---|---|
| **BackOffice web** | Muestra los productos de la empresa en COBIS con casillas para elegir |
| **API Gateway** | Valida el token del empleado y enruta al microservicio |
| **ms-grupos-economicos** | Consulta COBIS, valida y guarda los productos elegidos |
| **Integración con COBIS** (o un mock al inicio) | Lista los productos del cliente empresa. **Solo lectura** |
| **Base BELC** | `Empresas_asociadas`, `Producto_vinculacion`, `Tipo_producto`, `moneda` y `BITACORA_EVENTO` |
| **Librería de bitácora** (HU-13) | Registra el evento `PRODUCTO_VINCULADO` por cada producto |

### Modelo de datos
**Se leen:**
- `Empresas_asociadas`: que la empresa exista y pertenezca al grupo de la URL, y su `core_empresasoc_id` para consultar COBIS.
- `Tipo_producto` y `moneda`: para convertir los códigos de COBIS a los ids de tus catálogos.

**`Producto_vinculacion`** (una fila por producto elegido)
| Campo | De dónde viene |
|---|---|
| `id_producto_vinculado` | Lo genera la base (PK) |
| `id_empresa_asociada` | De la URL |
| `Core_module`, `Core_product_id`, `Core_cliente_id` | De COBIS |
| `id_tipo_producto` | Código de COBIS → `Tipo_producto` |
| `id_moneda` | Código de COBIS → `moneda` |
| `Numero_mascara` | Lo arma el microservicio, por ejemplo `****4521`. **Nunca se guarda el número completo** |
| `Estado_core` | El estado que trae COBIS |
| `Estado_Vinculacion` | `ACTIVO` |
| `Fecha_vinculacion`, `Fecha_ultima_sincronizacion` | Fecha del servidor |
| `Creado_por` | El empleado |

### Contrato
**1. Listar desde COBIS:** `GET .../empresas/201/productos-core`
```json
{
  "productos": [
    { "coreModule": "CTE", "coreProductId": "PRD-778120", "tipoProducto": "CUENTA_CORRIENTE", "moneda": "USD", "numeroMascara": "****4521", "estadoCore": "ACTIVA", "yaVinculado": false },
    { "coreModule": "DPF", "coreProductId": "PRD-778131", "tipoProducto": "DEPOSITO_PLAZO_FIJO", "moneda": "USD", "numeroMascara": "****0031", "estadoCore": "ACTIVA", "yaVinculado": true }
  ]
}
```
**2. Vincular:** `POST .../empresas/201/productos`
```json
{ "productos": ["PRD-778120"] }
```
**Respuesta 201**
```json
{ "vinculados": [ { "idProductoVinculado": 9001, "etiqueta": "Cuenta corriente ****4521" } ] }
```
**Errores:** `403` sin permiso · `404` el grupo o la empresa no existe, o la empresa no es de ese grupo · `409` el producto ya está vinculado · `422` el producto no es de esa empresa en COBIS, o su tipo no existe en `Tipo_producto` · `504` COBIS no respondió a tiempo.

### Criterios de aceptación
1. Solo se vinculan productos que COBIS devuelve para esa empresa. Al confirmar, **se vuelve a consultar COBIS**; no se confía en lo que manda la pantalla.
2. Se guarda solo el número enmascarado; el número completo nunca llega a la base.
3. Un producto no se vincula dos veces (409).
4. Si se eligen varios productos y uno falla, no se guarda ninguno.
5. Cada producto queda con `Estado_Vinculacion = ACTIVO`, su `Estado_core` y la fecha de sincronización.
6. Si COBIS no responde, devuelve 504 y no guarda nada.
7. Cada producto vinculado registra el evento `PRODUCTO_VINCULADO` (HU-13).

---

### Tareas aparte (cambios a tu DDL)
- **Agregar un UNIQUE en `(Core_module, Core_product_id)`** en `Producto_vinculacion`. Sin él, el criterio 3 depende solo del código.
- **`Producto_vinculacion` tiene `Moneda` y también `id_moneda`.** Deja solo `id_moneda`, o se pueden contradecir.
- **Agregar un CHECK en `Estado_Vinculacion`** con los estados de tu diapositiva 6: `PENDIENTE`, `ACTIVO`, `SUSPENDIDO`, `DESVINCULADO`.
- **Agregar `PRODUCTO_VINCULADO` a `TIPO_EVENTO`.**

### Dependencias
- **Equipo del core:** qué servicio de COBIS lista los productos de un cliente empresa y qué códigos usa para módulo, tipo de producto y moneda, para cargar la equivalencia en tus catálogos.
- **Una pregunta para ti:** `Core_cliente_id` en `Producto_vinculacion`, ¿es el mismo valor que `core_empresasoc_id` en `Empresas_asociadas`? Si es el mismo, sobra en `Producto_vinculacion`, porque ya se obtiene a través de la empresa.

**Fuera de alcance:** desvincular productos y la sincronización periódica de `Estado_core` con COBIS. Esa sincronización merece su propia historia.

### Pendientes con otros equipos
- **Equipo del core:** qué servicio de COBIS se usa para buscar empresas, por qué identificador (RUC o número de cliente) y cuál es su tiempo de respuesta, para fijar el timeout.
- **Líder técnico:** cómo se autentica el empleado del banco (igual que en la HU-02).

**Fuera de alcance:** desvincular una empresa o moverla de grupo, y vincular sus productos (eso es la HU-04).
